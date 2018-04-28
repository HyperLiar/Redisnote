# hyperloglog
## 1. 概述
```
为了突破64bit的哈希函数生成基数在10^9级别的限制，
使它消耗一个额外的bit per register.
16384个6-bit 高度精确的register每个key只使用总共12k

redis使用两种表述:
1. dense 密集
6-bit的integer代表每个entry
2. sparse 稀疏
步长压缩，许多register被设置为0
```
## 2. HLL header
```
无论哪种表述模式,都会有如下的16 bytes header
+------+---+-----+---------+
| HYLL | E | N/U | Cardin. |
+------+---+-----+---------+

HYLL占用4 bytes
E是1 bytes的encoding,设置为HLL_DENSE或者HLL_SPARSE
N/U是三个没有使用的字节

Cardin. 是一个用最新的基数格式化的64bit integer 按little endian存储
这是因为HLLADD操作很可能没有修改数据的接口和大概的基数
这样数据结构修改时，这个值可能可以复用.
```
## 3. Dense representation
```
每个桶内采用6bit大小存放
例如:

+--------+--------+--------+--------+--// //---+
|11000000|22221111|33333322|55444444    ...    |
+--------+--------+--------+--------+--// //---+

6 bits的计数器在其他内容从最低有效位到最低有效位编码结束后，
被编码为1个bit，根据需要使用下一个bytes
```
## 4. Sparse representation
```
使用一个run length encoding
由三个操作码组成，两个使用1 byte,另一个使用2 byte
分别称为ZERO XZERO VAL

ZERO: 00xxxxxx. 6-bit integer由6个bit xxxxxx表示,
这个操作码能表示1-64个连续的值为0的register

XZERO: 有两个byte 01xxxxxx yyyyyyyy. 这个14-bit的integer
中表示0-16384个连续set为0的值

VAL: 1vvvvvxx 包括5-bit integer代表register的值, 2-bit integer
代表连续被设置为vvvvv的值
为了获取值和run length, vvvvv和xx的增长必须是1, 这个操作码
能代表1-32的值和1-4的重复次数

Sparse表述无法代表一个大于32的值.
纯按位的表述,对空的HLL的表述：XZERO:16384

一个只有三个非0 register 在1000 1020 1021位置分别设置为
2 3 3的HLL, 它的opcodes为

XZERO: 1000 ( Registers 0-999 are set to 0)
VAL: 2,1 (1 register set to value 2, that is register 1000)
ZERO: 19 ( Registers 1001-1019 set to 0)
VAL: 3, 2 (2 register set to value 3, that is registers 1020, 1021)
XZERO: 15362 (Registers 1022-16383 set to 0)

在这个例子中它只使用了7 bytes表示HLL registers.
在low cardinality它的空间效率非常高,尽管会消费更多CPU时间去获取

dense representation 使用12288 bytes,在 2000-3000左右这是非常节约空间的
通过server.hll_sparse_max_bytes去界限确切的切换到dense表述的最大长度
```
## 5. hllhdr
```
struct hllhdr {
    char magic[4]; /* HYLL */
    uint8_t encoding; /* HLL_DENSE or HLL_SPARSE */
    uint8_t notused[3]; /* Reserved for future use, must be zero */
    uint8_t card[8]; /* Cached cardinality, little endian */
    uint8_t registers[]; /* Data types */
};
```
## 6. 宏
```
#define HLL_INVALIDATE_CACHE(hdr) (hdr)->card[7] |= (1<<7)
#define HLL_VALID_CACHE(hdr) (((hdr)->card[7] & (1<<7)) == 0)
#define HLL_P 14 
#define HLL_Q (64 - HLL_P) /* hash value的用于决定leading zero个数的bit数量 */
#define HLL_REGISTERS (1<<HLL_P) /* p = 14, 16384 registers */
#define HLL_P_MASK (HLL_REGISTERS-1) /* mask to index register */
#define HLL_BITS 6 /* 63 leading zeros */
#define HLL_REGISTER_MAX ((1<<HLL_BITS)-1)
#define HLL_HDR_SIZE sizeof(struct hllhdr)
#define HLL_DENSE_SIZE (HLL_HDR_SIZE+((HLL_REGISTERS*HLL_BITS+7)/8))
#define HLL_DENSE 0 
#define HLL_SPARSE 1
#define HLL_RAW 255
#define HLL_MAX_ENCODING 1
```
## 7. Low level bit macros 概述
```
```
