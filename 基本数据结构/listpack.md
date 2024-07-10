# listpack

listpack 是对压缩列表结构的优化，主要解决了压缩列表级联更新的问题。

## 存储结构

```
<lpbytes><lpnumbers><entry>...<entry><end>
```

+ lpbytes：listpack 占用的总字节数量
+ lpnumbers：listpack 中的元素数量
+ entry：listpack 节点，格式为 `<encode><val><backlen>`
  + encode：节点编码格式
  + val：节点存储的数据
  + backlen：记录当前节点的长度，用于反向遍历。backlen 最长5个字节，每个字节的 MSB 表示 backlen 是否结束，0表示结束，1表示尚未结束

encode 的含义如下

| code 内容 |  类型  |                            含义                             |
| :-------: | :----: | :---------------------------------------------------------: |
| 0xxxxxxx  |  整数  |                           [0-127]                           |
| 10xxxxxx  | 字符串 |                     长度小于64的字符串                      |
| 110xxxxx  |  整数  | [-4096,4095]，13比特的整数，后5比特以及下一个字节为数据内容 |
| 1110xxxx  | 字符串 |   长度小于4096的字符串，后4比特以及下一个字节为字符串长度   |
| 11110000  | 字符串 |      长度大于等于4096的字符串，后四个字节为字符串长度       |
| 11110001  |  整数  |                          16位整数                           |
| 11110010  |  整数  |                          24位整数                           |
| 11110011  |  整数  |                          32位整数                           |
| 11110100  |  整数  |                          64位整数                           |

## 初始化

listpack 的初始化比较简单

```C
//src/listpack.c
unsigned char *lpNew(size_t capacity) {
    unsigned char *lp = lp_malloc(capacity > LP_HDR_SIZE+1 ? capacity : LP_HDR_SIZE+1);
    if (lp == NULL) return NULL;
    lpSetTotalBytes(lp,LP_HDR_SIZE+1);//共7字节
    lpSetNumElements(lp,0);
    lp[LP_HDR_SIZE] = LP_EOF;//设置结束位
    return lp;
}
```

## 插入元素



```C
//src/listpack.c
/*
	在 listpack lp 的 p 位置插入 字符串 elestr 或者 整数 eleint。where 的取值可以为 LP_BEFORE、LP_AFTER 或者 LP_REPLACE。如果 elestr 和 eleint 均为 NULL，则删除 p 所指向的元素。
*/
unsigned char *lpInsert(unsigned char *lp, unsigned char *elestr, unsigned char *eleint,
                        uint32_t size, unsigned char *p, int where, unsigned char **newp)
{
    unsigned char intenc[LP_MAX_INT_ENCODING_LEN];
    unsigned char backlen[LP_MAX_BACKLEN_SIZE];

    uint64_t enclen; /* The length of the encoded element. */
    int delete = (elestr == NULL && eleint == NULL);//elestr 和 eleint 均为 NULL 说明是删除

    /* when deletion, it is conceptually replacing the element with a
     * zero-length element. So whatever we get passed as 'where', set
     * it to LP_REPLACE. */
    if (delete) where = LP_REPLACE; //通过使用0长元素来代替实现删除

    /* If we need to insert after the current element, we just jump to the
     * next element (that could be the EOF one) and handle the case of
     * inserting before. So the function will actually deal with just two
     * cases: LP_BEFORE and LP_REPLACE. */
    if (where == LP_AFTER) {//将 LP_AFTER 转换为 LP_BEFORE
        p = lpSkip(p);
        where = LP_BEFORE;
        ASSERT_INTEGRITY(lp, p);
    }

    /* Store the offset of the element 'p', so that we can obtain its
     * address again after a reallocation. */
    unsigned long poff = p-lp;

    int enctype;
    if (elestr) {
        /* Calling lpEncodeGetType() results into the encoded version of the
        * element to be stored into 'intenc' in case it is representable as
        * an integer: in that case, the function returns LP_ENCODING_INT.
        * Otherwise if LP_ENCODING_STR is returned, we'll have to call
        * lpEncodeString() to actually write the encoded string on place later.
        *
        * Whatever the returned encoding is, 'enclen' is populated with the
        * length of the encoded element. */
        enctype = lpEncodeGetType(elestr,size,intenc,&enclen);//判断 elestr 能否以整数进行编码
        if (enctype == LP_ENCODING_INT) eleint = intenc;
    } else if (eleint) {
        enctype = LP_ENCODING_INT;
        enclen = size; /* 'size' is the length of the encoded integer element. */
    } else {
        enctype = -1;
        enclen = 0;
    }

    /* We need to also encode the backward-parsable length of the element
     * and append it to the end: this allows to traverse the listpack from
     * the end to the start. */
    unsigned long backlen_size = (!delete) ? lpEncodeBacklen(backlen,enclen) : 0;//编码 backlen
    uint64_t old_listpack_bytes = lpGetTotalBytes(lp);//计算 listpack 的总字节数
    uint32_t replaced_len  = 0;
    if (where == LP_REPLACE) {
        replaced_len = lpCurrentEncodedSizeUnsafe(p);
        replaced_len += lpEncodeBacklen(NULL,replaced_len);
        ASSERT_INTEGRITY_LEN(lp, p, replaced_len);
    }

    uint64_t new_listpack_bytes = old_listpack_bytes + enclen + backlen_size
                                  - replaced_len;//计算调整后的 listpack 的长度
    if (new_listpack_bytes > UINT32_MAX) return NULL;

    /* We now need to reallocate in order to make space or shrink the
     * allocation (in case 'when' value is LP_REPLACE and the new element is
     * smaller). However we do that before memmoving the memory to
     * make room for the new element if the final allocation will get
     * larger, or we do it after if the final allocation will get smaller. */

    unsigned char *dst = lp + poff; /* May be updated after reallocation. */

    /* Realloc before: we need more room. */
    if (new_listpack_bytes > old_listpack_bytes &&//如果新的 listpack 的长度大于原来的 listpack 的长度，则重新申请空间
        new_listpack_bytes > lp_malloc_size(lp)) {
        if ((lp = lp_realloc(lp,new_listpack_bytes)) == NULL) return NULL;
        dst = lp + poff;
    }

    /* Setup the listpack relocating the elements to make the exact room
     * we need to store the new one. */
    if (where == LP_BEFORE) { //移动元素，腾出空间
        memmove(dst+enclen+backlen_size,dst,old_listpack_bytes-poff);
    } else { /* LP_REPLACE. */
        memmove(dst+enclen+backlen_size,
                dst+replaced_len,
                old_listpack_bytes-poff-replaced_len);
    }

    /* Realloc after: we need to free space. */
    if (new_listpack_bytes < old_listpack_bytes) {//释放多余空间
        if ((lp = lp_realloc(lp,new_listpack_bytes)) == NULL) return NULL;
        dst = lp + poff;
    }

    /* Store the entry. */
    if (newp) {//对于插入和替换，newp 指向当前元素，否则指向下一个元素
        *newp = dst;
        /* In case of deletion, set 'newp' to NULL if the next element is
         * the EOF element. */
        if (delete && dst[0] == LP_EOF) *newp = NULL;
    }
    if (!delete) {//将元素拷贝到指定位置
        if (enctype == LP_ENCODING_INT) {
            memcpy(dst,eleint,enclen);
        } else if (elestr) {
            lpEncodeString(dst,elestr,size);
        } else {
            redis_unreachable();
        }
        dst += enclen;
        memcpy(dst,backlen,backlen_size);
        dst += backlen_size;
    }

    /* Update header. */
    if (where != LP_REPLACE || delete) {//更新元素数量
        uint32_t num_elements = lpGetNumElements(lp);
        if (num_elements != LP_HDR_NUMELE_UNKNOWN) {
            if (!delete)
                lpSetNumElements(lp,num_elements+1);
            else
                lpSetNumElements(lp,num_elements-1);
        }
    }
    lpSetTotalBytes(lp,new_listpack_bytes);

#if 0
    /* This code path is normally disabled: what it does is to force listpack
     * to return *always* a new pointer after performing some modification to
     * the listpack, even if the previous allocation was enough. This is useful
     * in order to spot bugs in code using listpacks: by doing so we can find
     * if the caller forgets to set the new pointer where the listpack reference
     * is stored, after an update. */
    unsigned char *oldlp = lp;
    lp = lp_malloc(new_listpack_bytes);
    memcpy(lp,oldlp,new_listpack_bytes);
    if (newp) {
        unsigned long offset = (*newp)-oldlp;
        *newp = lp + offset;
    }
    /* Make sure the old allocation contains garbage. */
    memset(oldlp,'A',new_listpack_bytes);
    lp_free(oldlp);
#endif

    return lp;
}
```

