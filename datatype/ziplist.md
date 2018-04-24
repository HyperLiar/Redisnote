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
