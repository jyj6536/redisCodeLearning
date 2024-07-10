# AOF

Redis的AOF（Append Only File）是一种持久化数据的机制，它通过将所有写入操作追加到一个文件中来记录数据库的操作。AOF文件以文本形式记录Redis服务器接收到的写命令，这些写命令以Redis协议格式保存在AOF文件中。

AOF文件有三种不同的写入策略：appendfsync always、appendfsync everysec和appendfsync no。分别对应着每次写入都进行fsync操作、每秒执行一次fsync操作以及让操作系统决定何时执行fsync操作。这些策略各有优缺点，可以根据需求选择适合的策略。

使用AOF持久化机制的好处是可以确保数据的完整性和一致性，即使服务器意外崩溃，也能够通过AOF文件来恢复数据。另外，AOF文件还可以用于在主从复制中传输数据。

Redis中，与AOF文件相关的配置项包括：

1. appendonly：用于启用或禁用AOF持久化机制。可以设置为yes（启用）或者no（禁用）。
2. appendfilename：指定AOF文件的名称，默认为appendonly.aof。
3. appendfsync：指定AOF文件写入磁盘的策略，可以设置为always、everysec或no，分别对应每次写入都进行fsync操作、每秒执行一次fsync操作以及让操作系统决定何时执行fsync操作。
4. appendonlyno_fsync_on_rewrite：当开启AOF重写时，是否关闭fsync操作。可以设置为yes或者no。
5. auto-aof-rewrite-percentage：触发AOF重写的条件，当AOF文件大小超过上次重写时大小的百分之多少时触发重写。
6. auto-aof-rewrite-min-size：触发AOF重写的条件，当AOF文件大小超过指定大小时才会触发AOF重写。

## 定时逻辑

AOF重写和刷新的逻辑同样在serverCron中触发

```C
//src/server.c
   /* Start a scheduled AOF rewrite if this was requested by the user while
     * a BGSAVE was in progress. *///aof_rewrite_scheduled不为0代表存在AOF重写操作，如果当前不存在子进程并且未达到AOF重写限制则执行重写操作
    if (!hasActiveChildProcess() &&
        server.aof_rewrite_scheduled &&
        !aofRewriteLimited())//aofRewriteLimited：AOF重写限制检测，避免AOF重写持续失败导致的大量小文件
    {
        rewriteAppendOnlyFileBackground();
    }

    /* Check if a background saving or AOF rewrite in progress terminated. */
    if (hasActiveChildProcess() || ldbPendingChildren())
    {
        run_with_period(1000) receiveChildInfo();
        checkChildrenDone();
    } else {
       /*...*/

        /* Trigger an AOF rewrite if needed. */
        if (server.aof_state == AOF_ON &&//在开启了AOF重写功能并且满足以下条件则重写：没有子进程，当其那AOF文件相比于上次的AOF文件增加的大小以及当前文件大小超过相关配置，通过重写限制检测
            !hasActiveChildProcess() &&
            server.aof_rewrite_perc &&
            server.aof_current_size > server.aof_rewrite_min_size)
        {
            long long base = server.aof_rewrite_base_size ?
                server.aof_rewrite_base_size : 1;
            long long growth = (server.aof_current_size*100/base) - 100;
            if (growth >= server.aof_rewrite_perc && !aofRewriteLimited()) {
                serverLog(LL_NOTICE,"Starting automatic rewriting of AOF on %lld%% growth",growth);
                rewriteAppendOnlyFileBackground();
            }
        }
    }
```

AOF重写以及刷新的逻辑都在rewriteAppendOnlyFileBackground中实现

```C
//src/aof.c
/*
	首先刷新AOF缓冲区的内容到磁盘，然后创建子进程进行AOF重写
	redis7中引入了mutil part AOF：子进程重写产生BASE AOF文件；AOF重写时产生INCR AOF文件，可能存在多个（在重写失败的情况下）；每次AOF重写完成后，将本次重写之前对应的BASE和INCR文件变为VHISTORY文件由redis自动删除。redis引入了 manifest 文件记录这些文件的信息。这些文件被放入单独的目录中，由配置项 appenddirname 决定
*/
int rewriteAppendOnlyFileBackground(void) {
    pid_t childpid;

    if (hasActiveChildProcess()) return C_ERR;//判断是否存在子进程，存在则退出

    if (dirCreateIfMissing(server.aof_dirname) == -1) {//创建AOF文件目录
        serverLog(LL_WARNING, "Can't open or create append-only dir %s: %s",
            server.aof_dirname, strerror(errno));
        server.aof_lastbgrewrite_status = C_ERR;
        return C_ERR;
    }

    /* We set aof_selected_db to -1 in order to force the next call to the
     * feedAppendOnlyFile() to issue a SELECT command. */
    server.aof_selected_db = -1;
    flushAppendOnlyFile(1);//刷新AOF缓冲区的内容到文件中
    if (openNewIncrAofForAppend() != C_OK) {//这里，在开始AOF重写之前，创建新的INCR文件用于保存重写期间的增量命令，更新清单文件中的信息
        server.aof_lastbgrewrite_status = C_ERR;
        return C_ERR;
    }

    if (server.aof_state == AOF_WAIT_REWRITE) {
        /* Wait for all bio jobs related to AOF to drain. This prevents a race
         * between updates to `fsynced_reploff_pending` of the worker thread, belonging
         * to the previous AOF, and the new one. This concern is specific for a full
         * sync scenario where we don't wanna risk the ACKed replication offset
         * jumping backwards or forward when switching to a different master. */
        bioDrainWorker(BIO_AOF_FSYNC);

        /* Set the initial repl_offset, which will be applied to fsynced_reploff
         * when AOFRW finishes (after possibly being updated by a bio thread) */
        atomicSet(server.fsynced_reploff_pending, server.master_repl_offset);
        server.fsynced_reploff = 0;
    }

    server.stat_aof_rewrites++;

    if ((childpid = redisFork(CHILD_TYPE_AOF)) == 0) {//类似于RDB文件写入，这里创建AOF重写的子进程
        char tmpfile[256];

        /* Child */
        redisSetProcTitle("redis-aof-rewrite");
        redisSetCpuAffinity(server.aof_rewrite_cpulist);
        snprintf(tmpfile,256,"temp-rewriteaof-bg-%d.aof", (int) getpid());
        if (rewriteAppendOnlyFile(tmpfile) == C_OK) {//子进程逻辑：在临时文件中进行AOF写入
            serverLog(LL_NOTICE,
                "Successfully created the temporary AOF base file %s", tmpfile);
            sendChildCowInfo(CHILD_INFO_TYPE_AOF_COW_SIZE, "AOF rewrite");
            exitFromChild(0);
        } else {
            exitFromChild(1);
        }
    } else {
        /* Parent */
        if (childpid == -1) {
            server.aof_lastbgrewrite_status = C_ERR;
            serverLog(LL_WARNING,
                "Can't rewrite append only file in background: fork: %s",
                strerror(errno));
            return C_ERR;
        }
        serverLog(LL_NOTICE,
            "Background append only file rewriting started by pid %ld",(long) childpid);
        server.aof_rewrite_scheduled = 0;
        server.aof_rewrite_time_start = time(NULL);
        return C_OK;
    }
    return C_OK; /* unreached */
}
```

## AOF刷新

flushAppendOnlyFile 将AOF缓冲区的内容写入到磁盘文件

```C
//src/aof.c
void flushAppendOnlyFile(int force) {
    ssize_t nwritten;
    int sync_in_progress = 0;
    mstime_t latency;

    if (sdslen(server.aof_buf) == 0) {//AOF缓冲区为空，但是依然可能存在需要同步的情况，以下进行判断是否需要进行同步
        /* Check if we need to do fsync even the aof buffer is empty,
         * because previously in AOF_FSYNC_EVERYSEC mode, fsync is
         * called only when aof buffer is not empty, so if users
         * stop write commands before fsync called in one second,
         * the data in page cache cannot be flushed in time. */
        if (server.aof_fsync == AOF_FSYNC_EVERYSEC &&
            server.aof_last_incr_fsync_offset != server.aof_last_incr_size &&
            server.unixtime > server.aof_last_fsync &&
            !(sync_in_progress = aofFsyncInProgress())) {
            goto try_fsync;

        /* Check if we need to do fsync even the aof buffer is empty,
         * the reason is described in the previous AOF_FSYNC_EVERYSEC block,
         * and AOF_FSYNC_ALWAYS is also checked here to handle a case where
         * aof_fsync is changed from everysec to always. */
        } else if (server.aof_fsync == AOF_FSYNC_ALWAYS &&
                   server.aof_last_incr_fsync_offset != server.aof_last_incr_size)
        {
            goto try_fsync;
        } else {
            return;
        }
    }

    if (server.aof_fsync == AOF_FSYNC_EVERYSEC)//同步策略为每秒同步
        sync_in_progress = aofFsyncInProgress();//检查是否有后台线程在进行同步操作

    if (server.aof_fsync == AOF_FSYNC_EVERYSEC && !force) {//同步策略为每秒同步，并且强制刷新（force）为0的情况下，如果没有进行中的同步或者距离上次被延迟的同步的时间少于2秒，则延迟本次AOF刷新，否则继续向下执行
        /* With this append fsync policy we do background fsyncing.
         * If the fsync is still in progress we can try to delay
         * the write for a couple of seconds. */
        if (sync_in_progress) {
            if (server.aof_flush_postponed_start == 0) {
                /* No previous write postponing, remember that we are
                 * postponing the flush and return. */
                server.aof_flush_postponed_start = server.unixtime;
                return;
            } else if (server.unixtime - server.aof_flush_postponed_start < 2) {
                /* We were already waiting for fsync to finish, but for less
                 * than two seconds this is still ok. Postpone again. */
                return;
            }
            /* Otherwise fall through, and go write since we can't wait
             * over two seconds. */
            server.aof_delayed_fsync++;
            serverLog(LL_NOTICE,"Asynchronous AOF fsync is taking too long (disk is busy?). Writing the AOF buffer without waiting for fsync to complete, this may slow down Redis.");
        }
    }
    /* We want to perform a single write. This should be guaranteed atomic
     * at least if the filesystem we are writing is a real physical one.
     * While this will save us against the server being killed I don't think
     * there is much to do about the whole server stopping for power problems
     * or alike */

    if (server.aof_flush_sleep && sdslen(server.aof_buf)) {
        usleep(server.aof_flush_sleep);
    }

    latencyStartMonitor(latency);//记录本次AOF文件写入的耗时
    nwritten = aofWrite(server.aof_fd,server.aof_buf,sdslen(server.aof_buf));//将AOF文件写入磁盘
    latencyEndMonitor(latency);
    /* We want to capture different events for delayed writes:
     * when the delay happens with a pending fsync, or with a saving child
     * active, and when the above two conditions are missing.
     * We also use an additional event name to save all samples which is
     * useful for graphing / monitoring purposes. */
    if (sync_in_progress) {//记录的导致写入延迟的原因用于分析/监控
        latencyAddSampleIfNeeded("aof-write-pending-fsync",latency);
    } else if (hasActiveChildProcess()) {
        latencyAddSampleIfNeeded("aof-write-active-child",latency);
    } else {
        latencyAddSampleIfNeeded("aof-write-alone",latency);
    }
    latencyAddSampleIfNeeded("aof-write",latency);

    /* We performed the write so reset the postponed flush sentinel to zero. */
    server.aof_flush_postponed_start = 0;

    if (nwritten != (ssize_t)sdslen(server.aof_buf)) {//错误处理
        /*记录错误信息*/
        }

        /* Handle the AOF write error. */
        if (server.aof_fsync == AOF_FSYNC_ALWAYS) {//在同步策略是always的情况下，写入失败会导致redis退出
            /* We can't recover when the fsync policy is ALWAYS since the reply
             * for the client is already in the output buffers (both writes and
             * reads), and the changes to the db can't be rolled back. Since we
             * have a contract with the user that on acknowledged or observed
             * writes are is synced on disk, we must exit. */
            serverLog(LL_WARNING,"Can't recover from AOF write error when the AOF fsync policy is 'always'. Exiting...");
            exit(1);
        } else {
            /* Recover from failed write leaving data into the buffer. However
             * set an error to stop accepting writes as long as the error
             * condition is not cleared. */
            server.aof_last_write_status = C_ERR;

            /* Trim the sds buffer if there was a partial write, and there
             * was no way to undo it with ftruncate(2). */
            if (nwritten > 0) {
                server.aof_current_size += nwritten;
                server.aof_last_incr_size += nwritten;
                sdsrange(server.aof_buf,nwritten,-1);
            }
            return; /* We'll try again on the next call... */
        }
    } else {
        /* Successful write(2). If AOF was in error state, restore the
         * OK state and log the event. */
        if (server.aof_last_write_status == C_ERR) {//本次写入成功，如果之前有错误信息则清除
            serverLog(LL_NOTICE,
                "AOF write error looks solved, Redis can write again.");
            server.aof_last_write_status = C_OK;
        }
    }
    server.aof_current_size += nwritten;
    server.aof_last_incr_size += nwritten;

    /* Re-use AOF buffer when it is small enough. The maximum comes from the
     * arena size of 4k minus some overhead (but is otherwise arbitrary). */
    if ((sdslen(server.aof_buf)+sdsavail(server.aof_buf)) < 4000) {//如果AOF缓冲区总空间小于4000B则重用，否则创建新的缓冲区
        sdsclear(server.aof_buf);
    } else {
        sdsfree(server.aof_buf);
        server.aof_buf = sdsempty();
    }

try_fsync://以下开始进行同步操作
    /* Don't fsync if no-appendfsync-on-rewrite is set to yes and there are
     * children doing I/O in the background. */
    if (server.aof_no_fsync_on_rewrite && hasActiveChildProcess())//如果开启了aof_no_fsync_on_rewrite并且存在子进程则不同步
        return;

    /* Perform the fsync if needed. */
    if (server.aof_fsync == AOF_FSYNC_ALWAYS) {//如果同步策略是always，则立即同步
        /* redis_fsync is defined as fdatasync() for Linux in order to avoid
         * flushing metadata. */
        latencyStartMonitor(latency);
        /* Let's try to get this data on the disk. To guarantee data safe when
         * the AOF fsync policy is 'always', we should exit if failed to fsync
         * AOF (see comment next to the exit(1) after write error above). */
        if (redis_fsync(server.aof_fd) == -1) {//同步失败会导致redis退出
            serverLog(LL_WARNING,"Can't persist AOF for fsync error when the "
              "AOF fsync policy is 'always': %s. Exiting...", strerror(errno));
            exit(1);
        }
        latencyEndMonitor(latency);
        latencyAddSampleIfNeeded("aof-fsync-always",latency);
        server.aof_last_incr_fsync_offset = server.aof_last_incr_size;//更新已同步记录aof_last_incr_fsync_offset 
        server.aof_last_fsync = server.unixtime;
        atomicSet(server.fsynced_reploff_pending, server.master_repl_offset);
    } else if (server.aof_fsync == AOF_FSYNC_EVERYSEC &&
               server.unixtime > server.aof_last_fsync) {//如果策略是每秒同步且距离上次同步超过1s（unixtime单位为秒）则通过后台线程进行同步
        if (!sync_in_progress) {//后台线程是否还有同步中的任务，有的话则不进行同步
            aof_background_fsync(server.aof_fd);
            server.aof_last_incr_fsync_offset = server.aof_last_incr_size;
        }
        server.aof_last_fsync = server.unixtime;
    }
}
```

## AOF重写

AOF重写的逻辑在rewriteAppendOnlyFile中

```C
//src/aof.c
/*
	创建BASE AOF文件
*/
int rewriteAppendOnlyFile(char *filename) {
    rio aof;
    FILE *fp = NULL;
    char tmpfile[256];

    /* Note that we have to use a different temp name here compared to the
     * one used by rewriteAppendOnlyFileBackground() function. */
    snprintf(tmpfile,256,"temp-rewriteaof-%d.aof", (int) getpid());
    fp = fopen(tmpfile,"w");
    if (!fp) {
        serverLog(LL_WARNING, "Opening the temp file for AOF rewrite in rewriteAppendOnlyFile(): %s", strerror(errno));
        return C_ERR;
    }

    rioInitWithFile(&aof,fp);

    if (server.aof_rewrite_incremental_fsync) {//配置选项控制是否在AOF重写过程中进行增量地fsync操作
        rioSetAutoSync(&aof,REDIS_AUTOSYNC_BYTES);//每4MB执行一次数据落盘
        rioSetReclaimCache(&aof,1);
    }

    startSaving(RDBFLAGS_AOF_PREAMBLE);

    if (server.aof_use_rdb_preamble) {//是否开启混合持久化，默认开启。开启之后，redis会将数据以RDB格式保存到新文件中，再将重写缓冲区的命令以AOF格式写入文件；关闭的话redis将数据转化为写入命令写入文件
        int error;
        if (rdbSaveRio(SLAVE_REQ_NONE,&aof,&error,RDBFLAGS_AOF_PREAMBLE,NULL) == C_ERR) {//以RDB格式写入
            errno = error;
            goto werr;
        }
    } else {
        if (rewriteAppendOnlyFileRio(&aof) == C_ERR) goto werr;//针对不同对象以命令格式写入
    }

    /* Make sure data will not remain on the OS's output buffers */
    if (fflush(fp)) goto werr;
    if (fsync(fileno(fp))) goto werr;
    if (reclaimFilePageCache(fileno(fp), 0, 0) == -1) {
        /* A minor error. Just log to know what happens */
        serverLog(LL_NOTICE,"Unable to reclaim page cache: %s", strerror(errno));
    }
    if (fclose(fp)) { fp = NULL; goto werr; }
    fp = NULL;

    /* Use RENAME to make sure the DB file is changed atomically only
     * if the generate DB file is ok. */
    if (rename(tmpfile,filename) == -1) {
        serverLog(LL_WARNING,"Error moving temp append only file on the final destination: %s", strerror(errno));
        unlink(tmpfile);
        stopSaving(0);
        return C_ERR;
    }
    stopSaving(1);

    return C_OK;

werr:
    serverLog(LL_WARNING,"Write error writing append only file on disk: %s", strerror(errno));
    if (fp) fclose(fp);
    unlink(tmpfile);
    stopSaving(0);
    return C_ERR;
}
```

## 命令追加到缓冲区

redis 通过feedAppendOnlyFile将命令写入到AOF缓冲区

````C
//src/aof.c
void feedAppendOnlyFile(int dictid, robj **argv, int argc) {
    sds buf = sdsempty();

    serverAssert(dictid == -1 || (dictid >= 0 && dictid < server.dbnum));

    /* Feed timestamp if needed */
    if (server.aof_timestamp_enabled) {//redis7.0配置：记录命令时间戳
        sds ts = genAofTimestampAnnotationIfNeeded(0);
        if (ts != NULL) {
            buf = sdscatsds(buf, ts);
            sdsfree(ts);
        }
    }

    /* The DB this command was targeting is not the same as the last command
     * we appended. To issue a SELECT command is needed. */
    if (dictid != -1 && dictid != server.aof_selected_db) {//必要时插入select命令切换数据库
        char seldb[64];

        snprintf(seldb,sizeof(seldb),"%d",dictid);
        buf = sdscatprintf(buf,"*2\r\n$6\r\nSELECT\r\n$%lu\r\n%s\r\n",
            (unsigned long)strlen(seldb),seldb);
        server.aof_selected_db = dictid;
    }

    /* All commands should be propagated the same way in AOF as in replication.
     * No need for AOF-specific translation. */
    buf = catAppendOnlyGenericCommand(buf,argc,argv);//将命令写入缓冲区

    /* Append to the AOF buffer. This will be flushed on disk just before
     * of re-entering the event loop, so before the client will get a
     * positive reply about the operation performed. */
    if (server.aof_state == AOF_ON ||
        (server.aof_state == AOF_WAIT_REWRITE && server.child_type == CHILD_TYPE_AOF))
    {
        server.aof_buf = sdscatlen(server.aof_buf, buf, sdslen(buf));//将命令追加到AOF缓冲区aof_buf
    }

    sdsfree(buf);
}
````

## 重写结束

与RDB文件类似，redis在checkChildrenDone中检查AOF进程是否结束，AOF进程结束后调用backgroundRewriteDoneHandler进行AOF收尾工作

```C
//src/aof.c
/*
	与redis6采用管道的方式相比，redis7采用mutil part AOF方式写入AOF文件，这里进行了收尾工作，包括：
	1.创建临时清单文件
	2.将子进程生成的临时base文件重名为new_base_filename
	3.如果状态为AOF_WAIT_REWRITE，则存在临时INCR文件，将其重命名为new_incr_filename
	4.将现存的INCR文件标记为HISTORY并放入HISTORY文件列表中
	5.将上述所有变更持久化到manifest文件中
	6.删除HISTORY文件
*/
void backgroundRewriteDoneHandler(int exitcode, int bysignal) {
    if (!bysignal && exitcode == 0) {
        /*...*/
        /* Dup a temporary aof_manifest for subsequent modifications. */
        temp_am = aofManifestDup(server.aof_manifest);

        /* Get a new BASE file name and mark the previous (if we have)
         * as the HISTORY type. */
        sds new_base_filename = getNewBaseFileNameAndMarkPreAsHistory(temp_am);
        serverAssert(new_base_filename != NULL);
        new_base_filepath = makePath(server.aof_dirname, new_base_filename);

        /* Rename the temporary aof file to 'new_base_filename'. */
        latencyStartMonitor(latency);
        if (rename(tmpfile, new_base_filepath) == -1) {
            /*...*/
        }
        latencyEndMonitor(latency);
        latencyAddSampleIfNeeded("aof-rename", latency);
        serverLog(LL_NOTICE,
            "Successfully renamed the temporary AOF base file %s into %s", tmpfile, new_base_filename);

        /* Rename the temporary incr aof file to 'new_incr_filename'. */
        if (server.aof_state == AOF_WAIT_REWRITE) {
            /*...*/
        }

        /* Change the AOF file type in 'incr_aof_list' from AOF_FILE_TYPE_INCR
         * to AOF_FILE_TYPE_HIST, and move them to the 'history_aof_list'. */
        markRewrittenIncrAofAsHistory(temp_am);

        /* Persist our modifications. */
        if (persistAofManifest(temp_am) == C_ERR) {
            bg_unlink(new_base_filepath);
            aofManifestFree(temp_am);
            sdsfree(new_base_filepath);
            if (new_incr_filepath) {
                bg_unlink(new_incr_filepath);
                sdsfree(new_incr_filepath);
            }
            server.aof_lastbgrewrite_status = C_ERR;
            server.stat_aofrw_consecutive_failures++;
            goto cleanup;
        }
        sdsfree(new_base_filepath);
        if (new_incr_filepath) sdsfree(new_incr_filepath);

        /* We can safely let `server.aof_manifest` point to 'temp_am' and free the previous one. */
        aofManifestFreeAndUpdate(temp_am);

        if (server.aof_state != AOF_OFF) {
            /* AOF enabled. */
            server.aof_current_size = getAppendOnlyFileSize(new_base_filename, NULL) + server.aof_last_incr_size;
            server.aof_rewrite_base_size = server.aof_current_size;
        }

        /* We don't care about the return value of `aofDelHistoryFiles`, because the history
         * deletion failure will not cause any problems. */
        aofDelHistoryFiles();

        server.aof_lastbgrewrite_status = C_OK;
        server.stat_aofrw_consecutive_failures = 0;

        serverLog(LL_NOTICE, "Background AOF rewrite finished successfully");
        /* Change state from WAIT_REWRITE to ON if needed */
        if (server.aof_state == AOF_WAIT_REWRITE) {
            server.aof_state = AOF_ON;

            /* Update the fsynced replication offset that just now become valid.
             * This could either be the one we took in startAppendOnly, or a
             * newer one set by the bio thread. */
            long long fsynced_reploff_pending;
            atomicGet(server.fsynced_reploff_pending, fsynced_reploff_pending);
            server.fsynced_reploff = fsynced_reploff_pending;
        }

        serverLog(LL_VERBOSE,
            "Background AOF rewrite signal handler took %lldus", ustime()-now);
    } else if (!bysignal && exitcode != 0) 
        /*...*/
}
```

## AOF文件加载

redis启动时如果启用了AOF在会在loadDataFromDisk中加载AOF文件，调用的函数为loadAppendOnlyFiles

```C
//src/aof.c
/*
	首先，加载清单文件到server.aof_manifest
*/
void aofLoadManifestFromDisk(void) {
    server.aof_manifest = aofManifestCreate();
    if (!dirExists(server.aof_dirname)) {
        serverLog(LL_DEBUG, "The AOF directory %s doesn't exist", server.aof_dirname);
        return;
    }

    sds am_name = getAofManifestFileName();//清单文件命名规则为 server.aof_filename + “.manifest”
    sds am_filepath = makePath(server.aof_dirname, am_name);//拼装出完整的清单文件路径
    if (!fileExist(am_filepath)) {
        serverLog(LL_DEBUG, "The AOF manifest file %s doesn't exist", am_name);
        sdsfree(am_name);
        sdsfree(am_filepath);
        return;
    }

    aofManifest *am = aofLoadManifestFromFile(am_filepath);//加载清单文件
    if (am) aofManifestFreeAndUpdate(am);//修改server.aof_manifest的指向为am
    sdsfree(am_name);
    sdsfree(am_filepath);
}

//src/aof.c
int loadAppendOnlyFiles(aofManifest *am) {
    serverAssert(am != NULL);
    int status, ret = AOF_OK;
    long long start;
    off_t total_size = 0, base_size = 0;
    sds aof_name;
    int total_num, aof_num = 0, last_file;

    /* If the 'server.aof_filename' file exists in dir, we may be starting
     * from an old redis version. We will use enter upgrade mode in three situations.
     *
     * 1. If the 'server.aof_dirname' directory not exist
     * 2. If the 'server.aof_dirname' directory exists but the manifest file is missing
     * 3. If the 'server.aof_dirname' directory exists and the manifest file it contains
     *    has only one base AOF record, and the file name of this base AOF is 'server.aof_filename',//注释有点问题，代码条件是不相等
     *    and the 'server.aof_filename' file not exist in 'server.aof_dirname' directory
     * */
    if (fileExist(server.aof_filename)) {
        if (!dirExists(server.aof_dirname) ||
            (am->base_aof_info == NULL && listLength(am->incr_aof_list) == 0) ||
            (am->base_aof_info != NULL && listLength(am->incr_aof_list) == 0 &&
             !strcmp(am->base_aof_info->file_name, server.aof_filename) && !aofFileExist(server.aof_filename)))
        {
            aofUpgradePrepare(am);//创建AOF目录、BASE以及INCR等相关文件
        }
    }

    if (am->base_aof_info == NULL && listLength(am->incr_aof_list) == 0) {
        return AOF_NOT_EXIST;
    }

    total_num = getBaseAndIncrAppendOnlyFilesNum(am);
    serverAssert(total_num > 0);

    /* Here we calculate the total size of all BASE and INCR files in
     * advance, it will be set to `server.loading_total_bytes`. */
    total_size = getBaseAndIncrAppendOnlyFilesSize(am, &status);
    if (status != AOF_OK) {//一个文件记载清单文件中存在而磁盘上不存在则启动失败
        /* If an AOF exists in the manifest but not on the disk, we consider this to be a fatal error. */
        if (status == AOF_NOT_EXIST) status = AOF_FAILED;

        return status;
    } else if (total_size == 0) {
        return AOF_EMPTY;
    }

    startLoading(total_size, RDBFLAGS_AOF_PREAMBLE, 0);

    /* Load BASE AOF if needed. *///加载BASE文件
    if (am->base_aof_info) {
        /*...*/
        ret = loadSingleAppendOnlyFile(aof_name);//加载AOF文件：该函数兼容旧版本的混合持久化的AOF文件；对于RDB格式的内容通过rdbLoadRio加载；对于AOF格式的内容通过创建AFO伪客户端执行命令的方式加载
        /*...*/
    }

    /* Load INCR AOFs if needed. *///遍历并加载INCR文件
    if (listLength(am->incr_aof_list)) {
        listNode *ln;
        listIter li;

        listRewind(am->incr_aof_list, &li);
        while ((ln = listNext(&li)) != NULL) {
			/*...*/
            ret = loadSingleAppendOnlyFile(aof_name);
            /*...*/
        }
    }

    server.aof_current_size = total_size;
    /* Ideally, the aof_rewrite_base_size variable should hold the size of the
     * AOF when the last rewrite ended, this should include the size of the
     * incremental file that was created during the rewrite since otherwise we
     * risk the next automatic rewrite to happen too soon (or immediately if
     * auto-aof-rewrite-percentage is low). However, since we do not persist
     * aof_rewrite_base_size information anywhere, we initialize it on restart
     * to the size of BASE AOF file. This might cause the first AOFRW to be
     * executed early, but that shouldn't be a problem since everything will be
     * fine after the first AOFRW. */
    server.aof_rewrite_base_size = base_size;

cleanup:
    stopLoading(ret == AOF_OK || ret == AOF_TRUNCATED);
    return ret;
}
```

