# ziplist
## 1. ZIPLIST OVERALL LAYOUT
```
<zlbytes> <zltail> <zllen> <entry> <entry> ... <zlend>

<uint32_t zlbytes> 保存跳跃表的总byte数，包括它本身的4byte
<uint32_t zltail> 是到表最后一个entry的offset
<uint16_t zllen> 是entry的数量,超过2^16-2时，值为2^16-1，需要去遍历ziplist
确认item的总数
<uint8_t zlend> 是一个特殊的entry，代表ziplist的结尾，
encoded as a signle byte equal to 255,其他的entry不会以255开始
```
## 2. ZIPLIST ENTRIES
```
<prevlen> <encoding> <entry-data>

每个entry会有一个前缀，具有两部分数据
1: 前一个entry的长度，这是为了能够从后向前遍历
2: entry encoding，代表entry的数据类型，有效的长度

有时encoding会代表entry数据,例如小整数
<prevlen> <encoding>

<prevlen>:
如果长度小于254bytes, 使用unsigned 8 bit int的一个byte存储
如果长度大于等于254, 会消费5bytes。
第一个byte被设置为254表示后面跟随着更大的数
后面的4byte表示前一个entry的长度

<prevlen 0-253> <encoding> <entry>

0xFE <4 bytes unsigned little endian prevleb> <encoding> <entry>

<encoding>:
当entry为string时，第一个byte的前两个bit会用来encoding的长度
当entry为integer时, 前两个bit被设置为1，接着的两个bit用来标志
header后存储的数字类型，根据第一个byte总是能判断出数字的类型

|00pppppp| - 1 byte
    String, 长度小于等于63bytes (6 bits)
    "pppppp" 代表无符号的6 bit length.
|01pppppp | qqqqqqqq| - 2 byte
    String 长度小于等于 16383 bytes (14 bits)
    14 bit 的数字被存储在big-endian(高位编址)
|10000000|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt| - 5 bytes
    String, 长度大于等于 16383 bytes
    2-5个bytes代表长度最大为2^32 - 1，第一个byte的低6位没有使用
    被置为0
    32 bit的数字存储在big-endian (高位编址)
|11000000| - 3 bytes
    Integer encoded as int16_t (2 bytes)
|11010000| - 5 bytes
    Integer encoded as int32_t (4 bytes)
|11100000| - 9 bytes
    Integer encoded as int64_t (8 bytes)
|11110000| - 4 bytes
    Integer encoded as 24 bit signed (3 bytes)
|11111110| - 2 bytes
    Integer encoded as 8 bit signed (1 byte)
|1111xxxx| - (0000 - 1101) 表示 0 - 12 的 4bit integer.
    实际上值为1-13，因为0000和1111不会被使用
|11111111| - end of ziplist special entry

所有数字按little endian存储，即使是在高位编址的系统中。
```
## 3. EXAMPLES OF ACTUAL ZIPLISTS
```
15 bytes 的一个ziplist:
[0f 00 00 00] [0c 00 00 00] [02 00] [00 f3] [02 f6] [ff]
      |             |          |       |       |     |
   zlbytes       zltail      entries  "2"     "5"    end
最开始的4 bytes代表数字15，为整个ziplist组成的byte数量，
第二组4 bytes为最后一组entry的起始位置12
接下来的16 bit integer代表的是ziplist的元素个数2
[00 f3]，00表示其上一个entry的长度，F3为[1111xxxx]格式，
表示4 bit integer, 将F移除，3 - 1 = 2.
同理解释[02 f6]

向这个条约表添加"Hello world"来了解跳跃表如何编码小的string
假设存储在"5"之后

[02][0b][48 65 6c 6c 6f 20 57 6f 72 6c 64]
02表示前一个entry的长度，0b表示string为11bytes
后续的值代表ASCII码
```
## 4. 常量
```
ZIP_END 255
    ziplist 的结尾entry
ZIP_BIG_PREVLEN 254
    前一个entry的最大byte数
    当长度大于254时，使用FF AA BB CC DD，后四个byte去代表长度
ZIP_STR_MASK 0xc0 [1100 0000]
ZIP_INT_MASK 0x30 [0011 0000]
ZIP_STR_06B ( 0 << 6 ) [0000 0000]
ZIP_STR_14B ( 1 << 6 ) [0100 0000]
ZIP_STR_32B ( 2 << 6 ) [1000 0000]
ZIP_INT_16B ( 0xc0 | 0<<4) [1100 0000]
ZIP_INT_32B ( 0xc0 | 1<<4) [1101 0000]
ZIP_INT_64B ( 0xc0 | 2<<4) [1110 0000]
ZIP_INT_24B ( 0xc0 | 3<<4) [1111 0000]
ZIP_INT_8B  0xfe [11111110]

ZIP_INT_IMM_MASK 0x0f [00001111] mask to extrace 4 bit value
    需要 +1
ZIP_INT_IMM_MIN  0xf1 [11110001]
ZIP_INT_IMM_MAX  0xfd [11111101]

INT24_MAX 0x7fffff
INT24_MIN (-INT24_MAX - 1)
```
## 5. 宏
```
ZIP_IS_STR(enc) (((enc) & ZIP_STR_MASK) < ZIP_STR_MASK)
判断是否为string
因为String不会以11开头，所以&运算后不会比ZIP_STR_MASK大

ZIPLIST_BYTES(zl) (*((uint32_t*)(zl)))
return total bytes a ziplist is composed of

ZIPLIST_TAIL_OFFSET(z1) (*((uint32_t*)((zl+sizeof(uint32_t))))
return the offset of the last item inside the ziplist

ZIPLIST_LENGTH(zl) (*((uint16_t*)((zl)+sizeof(uint32_t)*2)))
return the length of a ziplist, or UINT16_MAX if the length 
cannot be determined without scanning the whole ziplist
返回ziplist的长度

ZIPLIST_HEADER_SIZE (sizeof(uint32_t)*2+sizeof(uint16_t))
size of a ziplist header: 2 32 bit integers for tge total
bytes count and last item offset, one 16 bit integer for the
number of items field
就是zlbyte zltail 和entries三个部分的总长度

ZIPLIST_END_SIZE （sizeof(uint8_t))
ziplist结尾entry的size 只有1 byte

ZIPLIST_ENTRY_HEAD(zl) ((zl)+ZIPLIST_HEADER_SIZE)
return the pointer to the first entry of a ziplist
返回ziplist第一个entry的指针

ZIPLIST_ENTRY_TAIL(zl) ((zl)+intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl)))
返回最后一个entry的指针

ZIPLIST_ENTRY_END(zl) ((zl)+intrev32ifbe(ZIPLIST_BYTES(zl)-1)
返回ziplist结尾FF entry的指针

ZIPLIST_INCR_LENGTH(zl, incr) { \
    if (ZIPLIST_LENGTH(zl) < UINT16_MAX) \
        ZIPLIST_LENGTH(zl) = intrev16ifbe(intrev16ifbe(ZIPLIST_LENGTH(zl)+incr); \
}
增加header中的元素数量，注意这个宏不会溢出无符号16位，当这个宏到
UINT16_MAX时，不再进行count，发出一个信号表示需要进行完整的扫描获取ziplist内的元素个数。
```
## 6. zipIntSize
```
unsigned int zipIntSize(unsigned char encoding)

返回对应encoding值的存储byte
```
## 7. zipStoreEntryEncoding
```
unsigned int zipStoreEntryEncoding(unsigned char *p, unsigned char encoding, unsigned int rawlen)

encoding 表示entry的编码方式
rawlen 仅为ZIP_STR_*的编码使用，表示这个entry内string的长度
返回存储在p中的 已使用的byte数量

使用了一次memcpy
```
## 8. ZIP_DECODE_LENGTH
```
#define ZIP_DECODE_LENGTH(ptr, encoding, lensize, len) do { \
     ZIP_ENTRY_ENCODING((ptr), (encoding); \
     if ((encoding) < ZIP_STR_MASK) {      \
         if ((encoding) == ZIP_STR_06B) {  \
             (lensize) = 1;                \
             (len) = (ptr)[0] & 0x3f;      \
         } else if ((encoding) == ZIP_STR_14B) { \
             (lensize) = 2;                \
             (len) = (((ptr[0] & 0x3f) << 8) | (ptr)[1]; \
         } else if ((encoding) == ZIP_STR_32B) {   \
             (lensize) = 5;                \
             (len) = ((ptr)[1] << 24) |   \
                     ((ptr)[2] << 16) |   \
                     ((ptr)[3] <<  8) |   \
                     ((ptr)[4]);          \
         } else {                         \
             panic("Invalid string encoding 0x%02X", (encoding)); \
         }                                 \
    } else {                               \
        (lensize) = 1;                     \
        (len) = zipIntSize(encoding);      \
    }                                      \
} while(0);                                \

decode the entry encoding type and data length encoded in ptr.
'encoding' hold the entry encoding.
'lensize' hold the number of bytes required to encode the entry length.
'len' hold the entry length.
```
## 9. zipStorePrevEntryLengthLarge
```
int zipStorePrevEntryLengthLarge(unsigned char *p, unsigned int len)

encode the length of the previous entry and write it to "p"
only uses the larger encoding
把p[0]设为ZIP_BIG_PREVLEN，然后做memcpy(p+1, &len, sizeof(len))
返回1 + sizeof(len)
```
## 10. zipStorePrevEntryLength
```
unsigned int zipStorePrevEntryLength(unsigned char *p, unsigned int len)

return the number of bytes needed to encode this length.

p为空时 len < ZIP_BIG_PREVLEN,返回1
否则返回sizeof(len) + 1

p不为空 len < ZIP_BIG_PREVLEN,置p[0] = len，返回1
否则 返回zipStorePrevEntryLengthLarge(p, len)
```
## 11. ZIP_DECODE_PREVLENSIZE
```
#define ZIP_DECODE_PREVLENSIZE(ptr, prevlensize) do {    \
    if ((ptr)[0] < ZIP_BIG_PREVLEN) {  \
        (prevlensize) = 1;             \
    } else {   \
        (prevlensize) = 5;             \
    }
} while(0);  \

返回用来encode前一个entry的byte长度，取决于(ptr[0])和ZIP_BIG_PREVLEN的比较
```
## 12. ZIP_DECODE_PREVLEN
```
#define ZIP_DECODE_PREVLEN(ptr, prevlensize, prevlen) do { \
    ZIP_DECODE_PREVLENSIZE(ptr, prevlensize);  \
    if ((prevlensize) == 1) {  \
        (prevlen) = (ptr)[0];  \
    } else if ((prevlensize) == 5) { \
        assert(sizeof((prevlen)) == 4); \
        memcpy(&(prelen),((char*)(ptr)) + 1, 4); \
        memrev32ifbe(&prevlen);     \
    } \
} while(0); \

ptr 是指向前一个entry的prevlen prefix的指针
prevlen保存前一个entry的长度
prevlensize存储encode前一个entry长度所需的bytes
```
## 13. zipPrevLenByteDiff
```
int zipPrevlenByteDiff(unsigned char *p, unsigned int len)

判断当前一个entry改变size时，所需encode previous entry的byte数

为新的空间与旧的空间的差值，需要扩大为负，缩小为正，不变为0
```
## 14. zipRawEntryLength
```
unsigned int zipRawEntryLength(unsigned char *p)

返回p指向的entry要使用的总byte数
ZIP_DECODE_PREVLENSIZE，获取prevlensize
ZIP_DECODE_LENGTH(p + prevlensize, encoding, lensize, len)
返回prevlensize + lensize + len
```
## 15. zipTryEncoding
```
int zipTryEncoding(unsigned char *entry, unsigned int entrylen, long long *v,
    unsigned char *encoding)

检查entry内的string是否能encode成一个integer
v保存int的value，encoding保存在encoding中，如果entrylen >= 32 或 == 0
两者都空
```
## 16. zipSaveInteger
```
void zipSaveInteger(unsigned char *p, int64_t value, unsigned char encoding)

按encoding将value存储在p中。
注意当encoding 在ZIP_INT_IMM_MIN和ZIP_INT_IMM_MAX间时
无操作，因为value存储在encoding内
```
## 17. zipLoadInteger
```
int64_t zipLoadInteger(unsigned char *p, unsigned char encoding)

从p中读取按encoding encode的integer
```
## 18. zipEntry
```
void zipEntry(unsigned char *p, zlentry *e)

在e中保存entry的全部信息
```
## 19. ziplistNew
```
unsigned char *ziplistNew(void)

创建空的ziplist
```
## 20. ziplistResize
```
unsigned char *ziplistResize(unsigned char *zl, unsigned int len)

resize ziplist
按len重新分配空间，重设zl的bytes数，将结尾设为ZIP_END
```
## 21. __ziplistCascadeUpdate
```
unsigned char *__ziplistCascadeUpdate(unsigned char *zl, unsigned char *p)

将p指向的entry存储在cur，p+cur.headersize+cur.len指向的entry存储在next
比较next.prevrawlen和rawlen = cur.headersize + cur.len

相等跳出循环

next.prevrawlen较小，说明prevlen需要更多的bytes，使用ziplistResize
接着更新tail offset如果下一个元素不是尾部元素
从p+rawlen+next.prevrawlensize 后移到p+rawlen+rawlensize
即将原节点下一节点到尾部的全部数据向后偏移出rawlensize个字节
将rawlen存储在p+rawlen中
p += rawlen; curlen += extra;
这是在移动循环的游标，只有这种需要拓展空间的情况会继续循环。

cur.headersize + cur.len较小,
比较next.prevrawlensize和rawlensize(zipStorePrevEntryLength的长度 1或5)
next.prevrawlensize较大
zipStorePreEntryLengthLarge(p+rawlen, rawlen))
在p+rawlen中记录rawlen的长度
否则 zipStorePrevEntryLength(p+rawlen, rawlen))
也是在p+rawlen中记录rawlen的长度，不同的是小于ZIP_BIG_PREVLEN时不需要memcpy

不进行压缩，防止扩容需要的循环调整。

在rawlen较小时，不需要调整next的rawlength，故跳出循环。

返回zl指针。
```
## 22. __ziplistDelete
```
unsigned char *__ziplistDelete(unsigned char *zl, unsigned char *p, unsigned int num)

删除p开始的num个entry
p += zipRawEntryLengh(p);通过这个方法不停的找到p的后一个entry的指针
直到num或p[0]为ZIP_END, 注意这个过程执行了一次循环
p指向待删除节点后第一个不被删除的节点

如果p[0] != ZIP_END，此时没有到表的结尾，需要改变p的PrevEntryLength
nextdiff  = zipPrevLenByteDiff(p, first.prevrawlen)
这是之前存储entry而现在被删除的空间
调整zl的tail_offset --> 好像多余了

判断p后面还有没有其他entry,有的情况下需要在尾节点偏移量加上nextdiff
--> 也是调整zl的tail_offset
进行memmove

否则整个tail被删除，不需要移动内存，调整tail_offset即可

接着需要resize zl，使用ZIPLIST_INCR_LENGTH调整zl的长度.
如果nextdiff不为0，p的长度改变了，则需要调用__ziplistCascadeUpdate，调整ziplist
```
## 23. __ziplistInsert
```
unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen)

如果s可以被编码为integer,则reqlen = zipIntSize(encoding)
否则reqlen = slen

reqlen需要加上前一entry的长度和encoding存储所需的长度

计算nextdiff->因为下一节点的rawlen需要保证能存储新增节点的长度
保存offset = p - zl,因为realloc可能改变zl的地址
之后p = zl + offset保证位置正确

如果p[0]不是尾节点
memmove，将p节点后的内容后移
更新tail_offset
当p节点的下一个节点不是尾节点，偏移量需要加上nextdiff

p[0]是尾节点，重设tail_offset即可

如果nextdiff不为0，执行__ziplistCascadeUpdate(zl, p+reqlen)

ziplist扩容完毕，将entry数据写入即可。
返回zl
```
## 24. ziplistMerge
```
unsigned char *zpilistMerge(unsigned char **first, unsigned char **second)
合并两个ziplist,append 'second' to 'first'

会realloc较大的ziplist加入另一个ziplist
realloc, append到后面执行memcpy,否则需要先memmove到后面,memcpy另一个ziplist
更新header的metadata，调用__ziplistCascadeUpdate重构appended ziplist
最后释放被append的ziplist。
```
## 25. ziplistPush
```
unsigned char *ziplistPush(unsigned char *zl, unsigned char *s, unsigned int slen, int where)
where 是ZIPLIST_HEAD时，在头节点插入s，否则在尾节点插入
return __ziplistInsert(zl,p,s,slen)
```
## 26. ziplistIndex
```
unsigned char *ziplistIndex(unsigned char *zl, int index)

返回index对应位置entry的指针
不包含给定index对应的entry的场合返回NULL

index < 0 表示从后往前迭代，使用ZIP_DECODE_PREVLEN(p, prevlensize, prevlen)
获取prevlen，p-=prevlen得到前一个元素的指针

index > 0 p += zipRawEntryLength(p); 不停拿到下一个元素的指针

这个方法内部需要循环。
```
## 27. ziplistNext
```
unsigned char *ziplistNext(unsigned char *zl, unsigned char *p)

拿到ziplist的下一个entry
注意会判断p[0]和ZIP_END是否相等，后移前后相等都会返回NULL
否则返回p.
```
## 28. ziplistPrev
```
unsigned char *ziplistPrev(unsigned char *zl, unsigned char *p))

获取p的前一个entry的指针
p[0] == ZIP_END, 则p = ZIPLIST_ENTRY_TAIL(zl)
若p[0] == ZIP_END 返回NULL,否则返回p

否则若p是Head,也返回NULL

否则通过ZIP_DECODE_PREVLEN(p, prevlensize,prevlen)，判断prevlen是否大于0
返回p - prevlen
```
## 29. ziplistGet
```
unsigned int ziplistGet(unsigned char *p, unsigned char **sstr, unsigned int *slen, long long *sval)

找到p对应的entry,按encode类型存储在*sstr或者*sval

调用zipEntry,之后ZIP_IS_STR(entry.encoding)决定如何读取entry中的值
```
## 30. ziplistInsert
```
unsigned char *ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen)

调用__ziplistInsert(zl, p, s, slen)
```
## 31. ziplistDelete
```
unsigned char *ziplistDelete(unsigned char *zl, unsigned char **p)

从ziplist删除一个entry
offset = *p - zl;
注意删除后，*p = zl + offset
这是因为__ziplistDelete会realloc，地址会发生变化。
```
## 32. ziplistDeleteRange
```
unsigned char *ziplistDeleteRange(unsigned char *zl, int index, unsigned int num)
p = ziplistIndex(zl, index)
p为空直接返回zl, 否则返回__ziplistDelete(zl,p,num)
```
## 33. ziplistCompare
```
unsigned int ziplistCompare(unsigned char *p, unsigned char *sstr, unsigned int slen))

相等返回1，否则返回0
比较p中的entry和*sstr, slen
如果ZIP_IS_STR(entry.encoding)，
entry.len == slen ，返回memcmp(p+entry.headersize, sstr, slen) == 0
否则返回0

如果是integer, 尝试将*sstr encode为int，比较zval = zipLoadInteger(p+entry.headersize, entry.coding)
和sval return zval == sval;
无法encode为int,返回0
```
## 34. ziplistFind
```
unsigned char *ziplistFind(unsigned char *p, unsigned char *vstr, unsigned int vlen, unsigned int skip)

找到对应entry的pointer，未发现返回空
```
## 35. ziplistLen
```
unsigned int ziplistLen(unsigned char *zl)

返回ziplist的长度
intrev16ifbe(ZIPLIST_LENGTH(zl)) < UINT16_MAX时
直接返回intrev16ifbe(ZIPLIST_LENGTH(zl))

否则需要循环 p += zipRawEntryLength(p)
len < UINT16_MAX时， ZIPLIST_LENGTH(zl) = intrev16ifbe(len)

返回len
```
## 36. ziplistBlobLen
```
size_t ziplistBlobLen(unsigned char *zl)

return intrev32ifbe(ZIPLIST_BYTES(zl))
```
