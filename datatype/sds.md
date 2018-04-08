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
```
## 28. sdscpy
```
sds sdscpy(sds s, const char *t)

类似于sdscpylen, t必须是null-termined（以'\0'结尾）的字符串，
否则无法用strlen获取其长度。
```
## 29. sdsll2str
```
int sdsll2str(char *s, long long value)

convert value to string, and store it in s.
's' must point to a string with room for at least SDS_LL_SIZE bytes.
```
## 30. sdsull2str
```
同sdsll2str.
```
## 31. sdsfromlonglong
```
sds sdsfromlonglong(long long value)

create an sds string from a long long value
much faster than sdscatprintf(sdsempty(), "%lld\n", value);

use sdsll2str, sdsnewlen
```
## 32. sds sdscatprintf
```
sds sdscatvprintf(sds s, const char *fmt, va_list ap)

use vsnprintf.
检查了fmt字符的长度，如果其二倍大于静态大小(1024)，则重新分配
格式长度为buflen, buflen-2 处设置为'\0'，进行vsnprintf.
若替换后，buflen-2 处不是'\0', 则buflen *= 2， 再次重新分配，
循环进行设置'\0',  vsnprintf, 检查的工作直至长度足够。
```
## 33. sdscatprintf
```
sds sdscatprintf(sds s, const char *fmt, ...)

调用后使用新指针替换:
s = sdsnew("Sum is : ");
s = sdscatprintf(s, "%d+%d = %d", a, b, a+b);

格式化的sds字符串:
s = sdscatprintf(sdsempty(), "... your format ...", args);

使用sdscatvprintf, 返回sdscatvprintf返回的指针.
```
## 34. sdscatfmt
```
比sdsprintf要快，不依赖于sprintf()家族函数。
只支持了printf-alike格式的不完整子集(imcompatible subset)

每个循环会依次执行sdsMakeRoomFor, memcpy, sdsinclen
是string或者sds格式时，需要循环sdsMakeRoomFor；
是long long, int64_t或者signed int时，需要sdsll2str(buf, num)
循环sdsMakeRoomFor(s, 1)；
是unsigned long long或者unsigned int时，需要sdsull2str(buf, num)
循环sdsMakeRoomFor(s, 1);
是%%时，只执行sdsinclen(s, 1)
最后在末尾添加'\0'(null-term)
```
## 35. sdstrim
```
sds sdstrim(sds s, const char *cset)

移除s两端在cset内的字符串.
两端依次检查字符是否在cset中，是则移动指针
最后memmove(s, sp, len)，sdssetlen。
```
## 36. sdsrange
```
void sdsrange(sds s, ssize_t start, ssize_t end)

获取start到end间的sds s.
使用memmove，移动指定的长度，sdssetlen设置新长度
```
## 37. sdstolower
```
void sdstolower(sds s)

对sds s的每个字符应用tolower
```
## 38. sdstoupper
```
void sdstoupper(sds s)

apply toupper() to every character of the sds string 's'
```
## 39. sdscmp
```
int sdscmp(const sds s1, const sds s2)

use memcmp(s1, s2, minlen)
当值为0时，比较两者长度；
否则返回memcmp结果。
```
## 40. sdssplitlen
```
sds *sdssplitlen(const char *s, ssize_t len, const char *sep, int seplen, int *count)

split s，按sep分割为*count数目的sds数组。
循环，使用memcmp进行比较。
```
## 41. sdsfreesplitres
```
void sdsfreesplitres(sds *tokens, int count)

free the result returned by sdssplitlen
循环执行sdsfree
```
## 42. sdscatrepr
```
sds sdscatrepr(sds s, const char *p, size_t len)

在sds s后，添加不可打印(isprint())字符的表达形式.
```
## 43. is_hex_digit
```
int is_hex_digit(char c)

检查是否十六进制字符
```
## 44. hex_digit_to_int
```
int hex_digit_to_int(char c)

将十六进制字符转换为十进制数字
```
## 45. sdssplitargs
```
sds *sdssplitargs(const char *line, int *argc)

Split a line into arguments, where every argument can be in the
following programming-language REPL-alike form:

foo bar "newline are supported\n" and "\xff\x00otherstuff"

*argc代表arguments的数量，返回是一个sds数组。
```
## 46. sdsmapchars
```
sds sdsmapchars(sds s, const char *from, const char *to, size_t setlen)

sdslen(s) * setlen复杂度，将s中在from字符串内的字符转换为to中对应的字符.
```
## 47. sdsjoin
```
sds sdsjoin(char **argv, int argc, char *seq)

使用seq连接argv的变量，返回一个sds
```
## 48. sdsjoinsds
```
sds sdsjoinsds(sds *argv, int argc, const char *sep, size_t seplen)

类似sdsjoin，连接的是sds而非c的字符串
```
## 49. sds_malloc
```
void *sds_malloc(size_t size)

调用s_malloc(size)
```
## 50. sds_realloc
```
void *sds_realloc(void *ptr, size_t size)

调用s_realloc(ptr, size)
```
## 51. sds_free
```
void sds_free(void *ptr)

调用s_free(ptr)
```
## 52. sdsTest
```
判断是否定义了SDS_TEST_MAIN宏，定义则加入main方法，进行测试
```
