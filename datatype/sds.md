# Sds
## 1. what is sds
```
typedef char *sds;
```
## 2. member in sds
```
uint32_t len            /* used length */
uint32_t alloc          /* excluding the header and null terminator */
unsigned char flags     /* 3 lsb of type, 5 unused bits */
char buf[]              /* string */
```
## 3. sdslen
```
static inline size_t sdslen(const sds s)

return len
```
## 4. sdsavail
```
static inline size_t sdsavail(const sds s)

return alloc - len
```
## 5. sdssetlen
```
static inline void sdssetlen(sds s, size_t newlen)

change len to newlen
```
## 6. sdsinclen
```
static inline void sdsinclen(sds s, size_t inc)

add inc to len
```
## 7. sdsalloc
```
static inline size_t sdsalloc(const sds s)

return alloc
```
## 8. sdssetalloc
```
static inline void sdssetalloc(sds s, size_t newlen)

change allow to newlen
```
## 9. sdsHdrSize
```
static inline int sdsHdrSize(char type)

return sizeof(sds of the type)
```
## 10. sdsReqType
```
static inline char sdsReqType(size_t string_size)

give proper sds type for string_size
```
## 11. sdsnewlen
```
sds sdsnewlen(const void *init, size_t initlen)

create a new sds with lengh initlen

type = sdsReqType(type)
hdrlen = sdsHdrSize(type)
set sds:
len: initlen
alloc: initen

```
## 12. sdsempty
```
sds sdsempty(void)

return an empty sds by sdsnewlen("", 0)
```
## 13. sdsnew
```
sds sdsnew(const char *init)

create a new sds string starting form a null terminated C string
```
## 14. sdsdup
```
sds sdsdup(const sds s)
duplicate an sds string
```
## 15. sdsfree
```
void sdsfree(sds s)

free an sds string
```
## 16. sdsupdatelen
```
void sdsupdatelen(sds s)

when '\0' is set to an sds, len remain the value, update to set a new len
```
## 17. sdsclear
```
void sdsclear(sds s)

set len to 0, and s[0] = '\0'
```
## 18. sdsMakeRoomFor
```
sds sdsMakeRoomFor(sds s, size_t addlen)

newlen = (len + addlen)
newlen < SDS_MAX_PREALLOC ? newlen *= 2 : newlen += SDS_MAX_PREALLOC
make new room, if sds type changes, s must be moved to a new space.
```
## 19. sdsRemoveFreeSpace
```
sds sdsRemoveFreeSpace(sds s)

remove alloc - len, if type changes, s must be moved to a new space.
```
## 20. sdsAllocSize
```
size_t sdsAllocSize(sds s)

return total size of the allocation of the specifed sds.
```
## 21. sdsAllocPtr
```
void *sdsAllocPtr(sds s)

return (void *) (s - sdsHdrSize(s[-1])
return the pointer of the actul SDS allocation.
```
## 22. sdsIncrLen
```
void sdsIncrLen(sds s, ssize_t incr)

fix sds length after sdsMakeRoomFor.
incr可正可负，只用于重设len，断言判断尾部空间是否足够
```
## 23. sdsgrowzero
```
sds sdsgrowzero(sds s, size_t len)

设定为固定的长度，新增长度都设为0
--> sdsMakeRoomz在sds类型不变的情况不set len，故此处重新设置了len字段
```
## 24. sdscatlen
```
sds sdscatlen(sds s, const void *t, size_t len)

在sds s后，保存字符串t.
```
## 25. sdscat
```
sds sdscat(sds s, const char *t)

调用sdscatlen
```
## 26. sdscatsds
```
sds sdscatsds(sds s, const sds t)

调用sdscatlen, 参数为sds
```
## 27. sdscpylen
```
sds sdscpylen(sds s, const char *t, size_t len)

将t复制到s所在的空间，长度也进行拷贝(可能会销毁s所在的内存)
