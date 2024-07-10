# RDB

redis 是内存数据库，重启数据丢失。因此，redis 提供了持久化的功能——RDB和AOF。

RDB：在不同的时点，将数据库快照以二进制的形式保存到磁盘文件中。

```
save 900 100 #每900秒内至少有100个键被修改则生成一个新的RDB文件，可以配置多个
dbfilename dump.rdb #配置rdb文件的名字
```

## 定时逻辑

redis 在 `serverCron` 中执行了定时生成RDB文件的逻辑

```C
//
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    /* Check if a background saving or AOF rewrite in progress terminated. */
    if (hasActiveChildProcess() || ldbPendingChildren())//检查当前是否存在子进程
    {
        run_with_period(1000) receiveChildInfo();//获取子进程信息
        checkChildrenDone();//调用waitpid获取子进程的结束信息
    } else {
        /* If there is not a background saving/rewrite in progress check if
         * we have to save/rewrite now. *///不存在子进程则检查是否需要现在进行写RDB文件的操作
        for (j = 0; j < server.saveparamslen; j++) {//saveparams保存rdb文件的保存条件
            struct saveparam *sp = server.saveparams+j;

            /* Save if we reached the given amount of changes,
             * the given amount of seconds, and if the latest bgsave was
             * successful or if, in case of an error, at least
             * CONFIG_BGSAVE_RETRY_DELAY seconds already elapsed. */
            if (server.dirty >= sp->changes &&//server.dirty保存了上一次生成RDB文件后，redis变更了多少键
                server.unixtime-server.lastsave > sp->seconds &&//server.unixtime-server.lastsave距离上次生成RDB文件经过的时间
                (server.unixtime-server.lastbgsave_try >//上次保存出错距离现在至少经过了 CONFIG_BGSAVE_RETRY_DELAY 秒
                 CONFIG_BGSAVE_RETRY_DELAY ||
                 server.lastbgsave_status == C_OK))//或者上次是成功的
            {
                serverLog(LL_NOTICE,"%d changes in %d seconds. Saving...",
                    sp->changes, (int)sp->seconds);
                rdbSaveInfo rsi, *rsiptr;
                rsiptr = rdbPopulateSaveInfo(&rsi);
                rdbSaveBackground(SLAVE_REQ_NONE,server.rdb_filename,rsiptr,RDBFLAGS_NONE);//满足上述条件生成RDB文件
                break;
            }
        }

        /* Trigger an AOF rewrite if needed. */
    }
}
```

## RDB持久化

### fork 子进程

rdbSaveBackground 中进行了创建子进程并保存文件的工作

```C
//src/rdb.c
int rdbSaveBackground(int req, char *filename, rdbSaveInfo *rsi, int rdbflags) {
    pid_t childpid;

    if (hasActiveChildProcess()) return C_ERR;//检查是否已经存在子进程，存在则退出
    server.stat_rdb_saves++;

    server.dirty_before_bgsave = server.dirty;
    server.lastbgsave_try = time(NULL);

    if ((childpid = redisFork(CHILD_TYPE_RDB)) == 0) {//调用 redisFork 创建子进程
        int retval;//这里执行子进程逻辑

        /* Child */
        redisSetProcTitle("redis-rdb-bgsave");//设置进程标题
        redisSetCpuAffinity(server.bgsave_cpulist);//设置cpu亲和性
        retval = rdbSave(req, filename,rsi,rdbflags);//写入RDB文件
        if (retval == C_OK) {
            sendChildCowInfo(CHILD_INFO_TYPE_RDB_COW_SIZE, "RDB");//向主进程发送子进程的统计信息
        }
        exitFromChild((retval == C_OK) ? 0 : 1);//推出子进程
    } else {
        /* Parent *///这里执行父进程逻辑，更新与RDB相关的运行时数据
        if (childpid == -1) {
            server.lastbgsave_status = C_ERR;
            serverLog(LL_WARNING,"Can't save in background: fork: %s",
                strerror(errno));
            return C_ERR;
        }
        serverLog(LL_NOTICE,"Background saving started by pid %ld",(long) childpid);
        server.rdb_save_time_start = time(NULL);
        server.rdb_child_type = RDB_CHILD_TYPE_DISK;
        return C_OK;
    }
    return C_OK; /* unreached */
}
```

redis 通过 redisFork 创建子进程

```C
//src/server.c
int redisFork(int purpose) {
    if (isMutuallyExclusiveChildType(purpose)) {
        if (hasActiveChildProcess()) {
            errno = EEXIST;
            return -1;
        }

        openChildInfoPipe();//子进程与父进程之间通过管道通信，这里对管道进行初始化，管道fd存放在 server.child_info_pipe 中
    }

    int childpid;
    long long start = ustime();
    if ((childpid = fork()) == 0) {
        /* Child.
         *
         * The order of setting things up follows some reasoning:
         * Setup signal handlers first because a signal could fire at any time.
         * Adjust OOM score before everything else to assist the OOM killer if
         * memory resources are low.
         */
        server.in_fork_child = purpose;//记录子的进程的作用
        setupChildSignalHandlers();//为子进程的SIGUSR1安装信号处理程序
        setOOMScoreAdj(CONFIG_OOM_BGCHILD);//调整 /proc/self/oom_score_adj，这是Linux内核提供的一个接口，用于调整当前进程的OOM（Out Of Memory）分数。OOM分数用于确定在系统内存不足时哪个进程会被内核选中作为OOM Killer的牺牲品，即被强制终止以释放内存。每个进程都有一个与之对应的OOM分数，默认为0。增加oom_score_adj值会降低进程的OOM分数，使其成为更不容易被选择为OOM Killer的目标。相反，减少oom_score_adj值会增加进程的OOM分数，使其更容易成为OOM Killer的目标。
        updateDictResizePolicy();//在子进程存在的情况下哈希表的调整会导致大量的CoW（写时复制），临时关闭哈希表的调整功能（更新dict_can_resize变量为DICT_RESIZE_FORBID）
        dismissMemoryInChild();//Jemalloc是一个开源的内存分配器，被广泛用于提高大型应用程序的内存管理性能。在使用Jemalloc的情况下，这里最终调用 madvise 将子进程不再需要的内存通知内核以避免CoW
        closeChildUnusedResourceAfterFork();//释放子进程不再需要的资源
        /* Close the reading part, so that if the parent crashes, the child will
         * get a write error and exit. */
        if (server.child_info_pipe[0] != -1)//关闭管道的一端
            close(server.child_info_pipe[0]);
    } else {
        /* Parent */
        if (childpid == -1) {
            int fork_errno = errno;
            if (isMutuallyExclusiveChildType(purpose)) closeChildInfoPipe();
            errno = fork_errno;
            return -1;
        }

        server.stat_total_forks++;
        server.stat_fork_time = ustime()-start;
        server.stat_fork_rate = (double) zmalloc_used_memory() * 1000000 / server.stat_fork_time / (1024*1024*1024); /* GB per second. */
        latencyAddSampleIfNeeded("fork",server.stat_fork_time/1000);

        /* The child_pid and child_type are only for mutually exclusive children.
         * other child types should handle and store their pid's in dedicated variables.
         *
         * Today, we allows CHILD_TYPE_LDB to run in parallel with the other fork types:
         * - it isn't used for production, so it will not make the server be less efficient
         * - used for debugging, and we don't want to block it from running while other
         *   forks are running (like RDB and AOF) */
        if (isMutuallyExclusiveChildType(purpose)) {
            server.child_pid = childpid;
            server.child_type = purpose;
            server.stat_current_cow_peak = 0;
            server.stat_current_cow_bytes = 0;
            server.stat_current_cow_updated = 0;
            server.stat_current_save_keys_processed = 0;
            server.stat_module_progress = 0;
            server.stat_current_save_keys_total = dbTotalServerKeyCount();
        }

        updateDictResizePolicy();
        moduleFireServerEvent(REDISMODULE_EVENT_FORK_CHILD,
                              REDISMODULE_SUBEVENT_FORK_CHILD_BORN,
                              NULL);
    }
    return childpid;
}
```

### 生成 RDB 文件

redis会在RDB文件的每一部分之前添加一个类型字节以标识其类型

```C
//src/rdb.h
/* Special RDB opcodes (saved/loaded with rdbSaveType/rdbLoadType). */
#define RDB_OPCODE_FUNCTION2  245   /* function library data */
#define RDB_OPCODE_FUNCTION_PRE_GA   246   /* old function library data for 7.0 rc1 and rc2 */
#define RDB_OPCODE_MODULE_AUX 247   /* Module auxiliary data. */
#define RDB_OPCODE_IDLE       248   /* LRU idle time. */
#define RDB_OPCODE_FREQ       249   /* LFU frequency. */
#define RDB_OPCODE_AUX        250   /* RDB aux field. */
#define RDB_OPCODE_RESIZEDB   251   /* Hash table resize hint. */
#define RDB_OPCODE_EXPIRETIME_MS 252    /* Expire time in milliseconds. */
#define RDB_OPCODE_EXPIRETIME 253       /* Old expire time in seconds. */
#define RDB_OPCODE_SELECTDB   254   /* DB number of the following keys. */
#define RDB_OPCODE_EOF        255   /* End of the RDB file. */
```

redis 通过 rdbSave 生成 RDB 文件

```C
//src/rdb.c
int rdbSave(int req, char *filename, rdbSaveInfo *rsi, int rdbflags) {
    char tmpfile[256];
    char cwd[MAXPATHLEN]; /* Current working dir path for error messages. */

    startSaving(RDBFLAGS_NONE);
    snprintf(tmpfile,256,"temp-%d.rdb", (int) getpid());//tmpfile 临时文件

    if (rdbSaveInternal(req,tmpfile,rsi,rdbflags) != C_OK) {//进一步封装
        stopSaving(0);
        return C_ERR;
    }
    
    /* Use RENAME to make sure the DB file is changed atomically only
     * if the generate DB file is ok. */
    if (rename(tmpfile,filename) == -1) {
        /*...*/
    }
    if (fsyncFileDir(filename) != 0) {//在linux下，针对filename所在的目录调用fdatasync确保filename的元数据被成功写入磁盘
        serverLog(LL_WARNING,
            "Failed to fsync directory while saving DB: %s", strerror(errno));
        stopSaving(0);
        return C_ERR;
    }

    serverLog(LL_NOTICE,"DB saved on disk");
    server.dirty = 0;
    server.lastsave = time(NULL);
    server.lastbgsave_status = C_OK;
    stopSaving(1);
    return C_OK;
}

//src/rdb.c
static int rdbSaveInternal(int req, const char *filename, rdbSaveInfo *rsi, int rdbflags) {
    char cwd[MAXPATHLEN]; /* Current working dir path for error messages. */
    rio rdb;
    int error = 0;
    int saved_errno;
    char *err_op;    /* For a detailed log */

    FILE *fp = fopen(filename,"w");//打开临时文件
    if (!fp) {
        /*...*/
    }

    rioInitWithFile(&rdb,fp);//rio是redis对读写层的抽象，将不同存储介质的 I/O 操作统一封装到rio中

    if (server.rdb_save_incremental_fsync) {//如果设置了 rdb_save_incremental_fsync，那么每写入REDIS_AUTOSYNC_BYTES字节的数据自动执行一次fsync
        rioSetAutoSync(&rdb,REDIS_AUTOSYNC_BYTES);
        if (!(rdbflags & RDBFLAGS_KEEP_CACHE)) rioSetReclaimCache(&rdb,1);//打开缓存回收标记。在每次自动fsync后，通知内核已刷新的内容不再需要以便内核将对应的内存回收，例子：posix_fadvise(fd, offset, len, POSIX_FADV_DONTNEED)
    }

    if (rdbSaveRio(req,&rdb,&error,rdbflags,rsi) == C_ERR) {//将redis数据库的内容写入到临时文件
        errno = error;
        err_op = "rdbSaveRio";
        goto werr;
    }

    /* Make sure data will not remain on the OS's output buffers */
    if (fflush(fp)) { err_op = "fflush"; goto werr; }//强制将输出缓冲区中的数据写入到文件中
    if (fsync(fileno(fp))) { err_op = "fsync"; goto werr; }//一个系统调用函数，用于强制将文件的数据和元数据写入到磁盘中。它能够确保文件的所有修改操作都已经同步地存储到磁盘上
    if (!(rdbflags & RDBFLAGS_KEEP_CACHE) && reclaimFilePageCache(fileno(fp), 0, 0) == -1) {//释放fp对应的内存页
        serverLog(LL_NOTICE,"Unable to reclaim cache after saving RDB: %s", strerror(errno));
    }
    if (fclose(fp)) { fp = NULL; err_op = "fclose"; goto werr; }

    return C_OK;

werr:
    saved_errno = errno;
    serverLog(LL_WARNING,"Write error while saving DB to the disk(%s): %s", err_op, strerror(errno));
    if (fp) fclose(fp);
    unlink(filename);
    errno = saved_errno;
    return C_ERR;
}

```

rdbSaveRio 对RDB文件的写入进行了进一步封装

```C
//src/rdb.c
int rdbSaveRio(int req, rio *rdb, int *error, int rdbflags, rdbSaveInfo *rsi) {
    char magic[10];
    uint64_t cksum;
    long key_counter = 0;
    int j;

    if (server.rdb_checksum)
        rdb->update_cksum = rioGenericUpdateChecksum;
    snprintf(magic,sizeof(magic),"REDIS%04d",RDB_VERSION);//RDB文件的标识
    if (rdbWriteRaw(rdb,magic,9) == -1) goto werr;
    if (rdbSaveInfoAuxFields(rdb,rdbflags,rsi) == -1) goto werr;//依次写入:版本号、32/64比特标识、创建时间、内存使用量，rsi不为null还会写入主从复制相关字段
    if (!(req & SLAVE_REQ_RDB_EXCLUDE_DATA) && rdbSaveModulesAux(rdb, REDISMODULE_AUX_BEFORE_RDB) == -1) goto werr;//触发模块自定义类型中的aux_save或者aux_save2函数，可以将数据库字典之外的数据保存到RDB文件中

    /* save functions */
    if (!(req & SLAVE_REQ_RDB_EXCLUDE_FUNCTIONS) && rdbSaveFunctions(rdb) == -1) goto werr;

    /* save all databases, skip this if we're in functions-only mode */
    if (!(req & SLAVE_REQ_RDB_EXCLUDE_DATA)) {
        for (j = 0; j < server.dbnum; j++) {//遍历所有数据库，写入数据
            if (rdbSaveDb(rdb, j, rdbflags, &key_counter) == -1) goto werr;//针对数据库j，遍历其所有键值对写入数据库
        }
    }

    if (!(req & SLAVE_REQ_RDB_EXCLUDE_DATA) && rdbSaveModulesAux(rdb, REDISMODULE_AUX_AFTER_RDB) == -1) goto werr;

    /* EOF opcode *///写入结束标识
    if (rdbSaveType(rdb,RDB_OPCODE_EOF) == -1) goto werr;

    /* CRC64 checksum. It will be zero if checksum computation is disabled, the
     * loading code skips the check in this case. */
    cksum = rdb->cksum;
    memrev64ifbe(&cksum);
    if (rioWrite(rdb,&cksum,8) == 0) goto werr;//写入CRC64校验和
    return C_OK;

werr:
    if (error) *error = errno;
    return C_ERR;
}

//src/rdb.c
ssize_t rdbSaveDb(rio *rdb, int dbid, int rdbflags, long *key_counter) {
    dictIterator *di;
    dictEntry *de;
    ssize_t written = 0;
    ssize_t res;
    static long long info_updated_time = 0;
    char *pname = (rdbflags & RDBFLAGS_AOF_PREAMBLE) ? "AOF rewrite" :  "RDB";

    redisDb *db = server.db + dbid;
    dict *d = db->dict;
    if (dictSize(d) == 0) return 0;
    di = dictGetSafeIterator(d);

    /* Write the SELECT DB opcode */
    if ((res = rdbSaveType(rdb,RDB_OPCODE_SELECTDB)) < 0) goto werr;//写入dbid
    written += res;
    if ((res = rdbSaveLen(rdb, dbid)) < 0) goto werr;
    written += res;

    /* Write the RESIZE DB opcode. */
    uint64_t db_size, expires_size;//将db->dict和db->expires的大小写入文件
    db_size = dictSize(db->dict);
    expires_size = dictSize(db->expires);
    if ((res = rdbSaveType(rdb,RDB_OPCODE_RESIZEDB)) < 0) goto werr;
    written += res;
    if ((res = rdbSaveLen(rdb,db_size)) < 0) goto werr;
    written += res;
    if ((res = rdbSaveLen(rdb,expires_size)) < 0) goto werr;
    written += res;

    /* Iterate this DB writing every entry */
    while((de = dictNext(di)) != NULL) {//遍历db写入文件
        sds keystr = dictGetKey(de);
        robj key, *o = dictGetVal(de);
        long long expire;
        size_t rdb_bytes_before_key = rdb->processed_bytes;

        initStaticStringObject(key,keystr);
        expire = getExpire(db,&key);
        if ((res = rdbSaveKeyValuePair(rdb, &key, o, expire, dbid)) < 0) goto werr;//将键值对的类型、超时时间、键、值写入文件
        written += res;

        /* In fork child process, we can try to release memory back to the
         * OS and possibly avoid or decrease COW. We give the dismiss
         * mechanism a hint about an estimated size of the object we stored. */
        size_t dump_size = rdb->processed_bytes - rdb_bytes_before_key;
        if (server.in_fork_child) dismissObject(o, dump_size);//通过MADV_DONTNEED快速向操作系统释放内存以避免子进程CoW

        /* Update child info every 1 second (approximately).
         * in order to avoid calling mstime() on each iteration, we will
         * check the diff every 1024 keys */
        if (((*key_counter)++ & 1023) == 0) {//每一秒向发送一次子进程信息
            long long now = mstime();
            if (now - info_updated_time >= 1000) {
                sendChildInfo(CHILD_INFO_TYPE_CURRENT_INFO, *key_counter, pname);
                info_updated_time = now;
            }
        }
    }

    dictReleaseIterator(di);
    return written;

werr:
    dictReleaseIterator(di);
    return -1;
}

//src/rdb.c
int rdbSaveKeyValuePair(rio *rdb, robj *key, robj *val, long long expiretime, int dbid) {
    int savelru = server.maxmemory_policy & MAXMEMORY_FLAG_LRU;//是否保存键的空闲时间
    int savelfu = server.maxmemory_policy & MAXMEMORY_FLAG_LFU;//是否保存LFU计数

    /* Save the expire time */
    if (expiretime != -1) {//如果设置了过期时间则写入
        if (rdbSaveType(rdb,RDB_OPCODE_EXPIRETIME_MS) == -1) return -1;
        if (rdbSaveMillisecondTime(rdb,expiretime) == -1) return -1;
    }

    /* Save the LRU info. */
    if (savelru) {
        uint64_t idletime = estimateObjectIdleTime(val);
        idletime /= 1000; /* Using seconds is enough and requires less space.*/
        if (rdbSaveType(rdb,RDB_OPCODE_IDLE) == -1) return -1;
        if (rdbSaveLen(rdb,idletime) == -1) return -1;
    }

    /* Save the LFU info. */
    if (savelfu) {
        uint8_t buf[1];
        buf[0] = LFUDecrAndReturn(val);
        /* We can encode this in exactly two bytes: the opcode and an 8
         * bit counter, since the frequency is logarithmic with a 0-255 range.
         * Note that we do not store the halving time because to reset it
         * a single time when loading does not affect the frequency much. */
        if (rdbSaveType(rdb,RDB_OPCODE_FREQ) == -1) return -1;
        if (rdbWriteRaw(rdb,buf,1) == -1) return -1;
    }

    /* Save type, key, value *///保存格式<值类型><键，值>
    if (rdbSaveObjectType(rdb,val) == -1) return -1;//根据值类型（redisObject::type）写入一个表示值类型的宏定义
    if (rdbSaveStringObject(rdb,key) == -1) return -1;//写入键
    if (rdbSaveObject(rdb,val,key,dbid) == -1) return -1;//写入值

    /* Delay return if required (for testing) */
    if (server.rdb_key_save_delay)
        debugDelay(server.rdb_key_save_delay);

    return 1;
}
```

### 父进程收尾

linux中，子进程结束后需要父进程进行资源回收工作。redis 通过checkChildrenDone实现

```C
//src/server.c
void checkChildrenDone(void) {
    int statloc = 0;
    pid_t pid;

    if ((pid = waitpid(-1, &statloc, WNOHANG)) != 0) {//以非阻塞的方式等待任意一个子进程退出
        int exitcode = WIFEXITED(statloc) ? WEXITSTATUS(statloc) : -1;//判断子进程是否正常退出并获取返回码
        int bysignal = 0;

        if (WIFSIGNALED(statloc)) bysignal = WTERMSIG(statloc);//判断子进程是否因为信号终止，是则获取导致子进程终止的信号

        /* sigKillChildHandler catches the signal and calls exit(), but we
         * must make sure not to flag lastbgsave_status, etc incorrectly.
         * We could directly terminate the child process via SIGUSR1
         * without handling it */
        if (exitcode == SERVER_CHILD_NOERROR_RETVAL) {//子进程收到SIGUSR1信号后，会返回SERVER_CHILD_NOERROR_RETVAL
            bysignal = SIGUSR1;
            exitcode = 1;
        }

        if (pid == -1) {
            serverLog(LL_WARNING,"waitpid() returned an error: %s. "
                "child_type: %s, child_pid = %d",
                strerror(errno),
                strChildType(server.child_type),
                (int) server.child_pid);
        } else if (pid == server.child_pid) {
            if (server.child_type == CHILD_TYPE_RDB) {
                backgroundSaveDoneHandler(exitcode, bysignal);//执行RDB文件生成之后的收尾工作
            } else if (server.child_type == CHILD_TYPE_AOF) {
                backgroundRewriteDoneHandler(exitcode, bysignal);
            } else if (server.child_type == CHILD_TYPE_MODULE) {
                ModuleForkDoneHandler(exitcode, bysignal);
            } else {
                serverPanic("Unknown child type %d for child pid %d", server.child_type, server.child_pid);
                exit(1);
            }
            if (!bysignal && exitcode == 0) receiveChildInfo();
            resetChildState();
        } else {
            if (!ldbRemoveChild(pid)) {
                serverLog(LL_WARNING,
                          "Warning, detected child with unmatched pid: %ld",
                          (long) pid);
            }
        }

        /* start any pending forks immediately. */
        replicationStartPendingFork();
    }
}
```

### 加载RDB文件

loadDataFromDisk负责加载RDB文件（由main）函数触发，调用顺序为main->loadDataFromDisk->rdbLoad->rdbLoadRio

````C
//src/rdb.c
int rdbLoadRio(rio *rdb, int rdbflags, rdbSaveInfo *rsi) {
    functionsLibCtx* functions_lib_ctx = functionsLibCtxGetCurrent();
    rdbLoadingCtx loading_ctx = { .dbarray = server.db, .functions_lib_ctx = functions_lib_ctx };
    int retval = rdbLoadRioWithLoadingCtx(rdb,rdbflags,rsi,&loading_ctx);
    return retval;
}

//src/rdb.c
int rdbLoadRioWithLoadingCtx(rio *rdb, int rdbflags, rdbSaveInfo *rsi, rdbLoadingCtx *rdb_loading_ctx) {
    if (rioRead(rdb,buf,9) == 0) goto eoferr;//校验版本号
    buf[9] = '\0';
    if (memcmp(buf,"REDIS",5) != 0) {
        serverLog(LL_WARNING,"Wrong signature trying to load DB from file");
        return C_ERR;
    }
    rdbver = atoi(buf+5);
    if (rdbver < 1 || rdbver > RDB_VERSION) {
        serverLog(LL_WARNING,"Can't handle RDB format version %d",rdbver);
        return C_ERR;
    }
    while(1) {
        sds key;
        robj *val;

        /* Read type. */
        if ((type = rdbLoadType(rdb)) == -1) goto eoferr; //获取内容类型
        
        /* Handle special types. *///根据获取到的类型作具体处理
        if (type == RDB_OPCODE_EXPIRETIME) {
            /*...*/
            
        /* Read key *///执行到这里，说明读取的是键值对类型，分别读取键值
        if ((key = rdbGenericLoadStringObject(rdb,RDB_LOAD_SDS,NULL)) == NULL)
            goto eoferr;
        /* Read value */
        val = rdbLoadObject(type,rdb,key,db->id,&error);
            
        if (val == NULL) {
            /* Since we used to have bug that could lead to empty keys
             * (See #8453), we rather not fail when empty key is encountered
             * in an RDB file, instead we will silently discard it and
             * continue loading. */
            if (error == RDB_LOAD_ERR_EMPTY_KEY) {
                if(empty_keys_skipped++ < 10)
                    serverLog(LL_NOTICE, "rdbLoadObject skipping empty key: %s", key);
                sdsfree(key);
            } else {
                sdsfree(key);
                goto eoferr;
            }
        } else if (iAmMaster() && //如果当前是主节点且键已过期，则删除键
            !(rdbflags&RDBFLAGS_AOF_PREAMBLE) &&
            expiretime != -1 && expiretime < now)
        {
            if (rdbflags & RDBFLAGS_FEED_REPL) {
                /* Caller should have created replication backlog,
                 * and now this path only works when rebooting,
                 * so we don't have replicas yet. */
                serverAssert(server.repl_backlog != NULL && listLength(server.slaves) == 0);
                robj keyobj;
                initStaticStringObject(keyobj,key);
                robj *argv[2];
                argv[0] = server.lazyfree_lazy_expire ? shared.unlink : shared.del;
                argv[1] = &keyobj;
                replicationFeedSlaves(server.slaves,dbid,argv,2);
            }
            sdsfree(key);
            decrRefCount(val);
            server.rdb_last_load_keys_expired++;
        } else {
            robj keyobj;
            initStaticStringObject(keyobj,key);

            /* Add the new object in the hash table */
            int added = dbAddRDBLoad(db,key,val);
            server.rdb_last_load_keys_loaded++;
            if (!added) {
                if (rdbflags & RDBFLAGS_ALLOW_DUP) {
                    /* This flag is useful for DEBUG RELOAD special modes.
                     * When it's set we allow new keys to replace the current
                     * keys with the same name. */
                    dbSyncDelete(db,&keyobj);
                    dbAddRDBLoad(db,key,val);//调用dbAddRDBLoad将键值对添加到数据库中
                } else {
                    serverLog(LL_WARNING,
                        "RDB has duplicated key '%s' in DB %d",key,db->id);
                    serverPanic("Duplicated key found in RDB file");
                }
            }

            /* Set the expire time if needed */
            if (expiretime != -1) {
                setExpire(NULL,db,&keyobj,expiretime);
            }

            /* Set usage information (for eviction). */
            objectSetLRUOrLFU(val,lfu_freq,lru_idle,lru_clock,1000);

            /* call key space notification on key loaded for modules only */
            moduleNotifyKeyspaceEvent(NOTIFY_LOADED, "loaded", &keyobj, db->id);
        }
	}
    /* Verify the checksum if RDB version is >= 5 */
    if (rdbver >= 5) {
        uint64_t cksum, expected = rdb->cksum;

        if (rioRead(rdb,&cksum,8) == 0) goto eoferr;
        if (server.rdb_checksum && !server.skip_checksum_validation) {
            memrev64ifbe(&cksum);
            if (cksum == 0) {
                serverLog(LL_NOTICE,"RDB file was saved with checksum disabled: no check performed.");
            } else if (cksum != expected) {
                serverLog(LL_WARNING,"Wrong RDB checksum expected: (%llx) but "
                    "got (%llx). Aborting now.",
                        (unsigned long long)expected,
                        (unsigned long long)cksum);
                rdbReportCorruptRDB("RDB CRC error");
                return C_ERR;
            }
        }
    }
}
````

