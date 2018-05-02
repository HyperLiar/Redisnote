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
我们需要在8 bit types内设置6 bit的counter.
使用宏来保障运行速度。
```
## 8. HLL_DENSE_GET_REGISTER
```
#define HLL_DENSE_GET_REGISTER(target, p, regnum) do { \
    uint8_t *_p = (uint8_t*) p; \
    unsigned long _byte = regnum*HLL_BITS/8; \
    unsigned long _fb = regnum*HLL_BITS&7; \
    unsigned long _fb8 = 8 - _fb; \
    unsigned long b0 = _p[_byte]; \
    unsigned long b1 = _p[_byte+1]; \
    target = ((b0 >> _fb) | (b1 << _fb8)) & HLL_REGISTER_MAX; \
} while(0)

将register regnum位置的值存储到变量target中
p是unsigned byte的数组
```
## 9. HLL_DENSE_SET_REGISTER
```
#define HLL_DENSE_SET_REGISTER(p, regnum, val) do { \
    uint8_t *_p = (uint8_t*) p; \
    unsigned long _byte = regnum*HLL_BITS/8; \
    unsigned long _fb = regnum*HLL_BITS&7; \
    unsigned long _fb8 = 8 - _fb; \
    unsigned long _v = val;
    _p[_byte] &= ~(HLL_REGISTER_MAX << _fb); \
    _p[_byte] |= _v << _fb; \
    _p[_byte+1] &= ~(HLL_REGISTER_MAX >> _fb8); \
    _p[_byte+1] |= _v >> _fb8; \
} while (0) 
设置register在regnum位置的值为val
```
## 10. macros to access the sparse representation
```
#define HLL_SPARSE_XZERO_BIT 0x40 /* 01xxxxxx */
#define HLL_SPARSE_VAL_BIT 0x80 /* 1vvvvvxx */
#define HLL_SPARSE_IS_ZERO(p) (((*(p)) & 0xc0 ) == 0) /* 00xxxxxx */
#define HLL_SPARSE_IS_XZERO(p) (((*p)) & 0xc0) == HLL_SPARSE_XZERO_BIT)
#define HLL_SPARSE_IS_VAL(p) ((*(p)) & HLL_SPARSE_VAL_BIT)
#define HLL_SPARSE_ZERO_LEN(p) (((*(p) & 0x3f)+1)
#define HLL_SPARSE_XZERO_LEN(p) (((((*(p)) & 0x3f) << 8) | (*((p)+1)))+1)
#define HLL_SPARSE_VAL_VALUE(p) ((((*(p)) >> 2) & 0x1f)+1)
#define HLL_SPARSE_VAL_LEN(p) (((*(p)) & 0x3)+1)
#define HLL_SPARSE_VAL_MAX_VALUE 32
#define HLL_SPARSE_VAL_MAX_LEN 4
#define HLL_SPARSE_ZERO_MAX_LEN 64
#define HLL_SPARSE_XZERO_MAX_LEN 16384
#define HLL_SPARSE_VAL_SET(p, val, len) do { \
    *(p) = (((val)-1)<<2|((len)-1)|HLL_SPARSE_VAL_BIT; \
} while(0)
#define HLL_SPARSE_ZERO_SET(p,len) do { \
    *(p) = (len)-1; \
} while(0)
#define HLL_SPARSE_XZERO_SET(p, len) do { \
    int _l = (len)-1; \
    *(p) = (_l>>8) | HLL_SPARSE_XZERO_BIT; \
    *((p)+1) = (_l&0xff); \
} while(0)
#define HLL_ALPHA_INF 0.721347520444481703680
/* 0.5 / ln2 */
```
## 11. MurmurHash64A
```
uint64_t MurmurHash64A (const void *key, int len, unsigned int seed)
hash function is murmurhash2, 64 bit version
做了一些修改，保证big little endian archs会得到相同的结果
```
## 12. hllPatLen
```
int hllPatLen(unsigned char *ele, size_t elesize, long *regp)

给出要添加到hyperloglog的string，返回000...1这个模式在ele的hash
中的长度,reg被设置为这个元素出来register中的index
index = hash & HLL_P_MASK; 存储,
hash >> = HLL_P; 移除index部分
hash |= ((uint64_t)1<<HLL_Q) -> 保证count <= Q-1

就是为了数出最后的1所在位置
```
## 13. hllDenseSet
```
int hllDenseSet(uint_8 *registers, long index, uint8_t count)
HLL_DENSE_GET_REGISTER得到oldcount
如果count > oldcount
HLL_DENSE_SET_REGISTER
```
## 14. hllDenseAdd
```
int hllDenseAdd(uint8_t *registers, unsigned char *ele, size_t elesize)
本质上没有做任何添加，如果有必要会增加元素所属的subset
内,max 0 pattern counter.
只是hllDenseSet的封装
```
## 15. hllDenseRegHisto
```
void hllDenseRegHisto(uint8_t *registers, int* reghisto)
计算dense representation的register histogram

没有使用循环,使用写死的16384(register)和6(bits)
```
## 16. hllSparseToDense
```
int hllSparseToDense(robj *o)

```
## 17. hllSparseSet
```
int hllSparseSet(robj *o, long index, uint8_t count)

```
## 18. hllSparseAdd
```
int hllSparseAdd(robj *o, unsigned char *ele, size_t elesize)
类似于hllDenseAdd,只是hllSparseSet的封装
```
## 19. hllSparseRegHisto
```
void hllSparseRegHisto(uint8_t *sparse, int sparselen, int* invalid, int *reghisto)

计算register histogram
```
## 20. hllRawRegHisto
```
void hllRawRegHisto(uint8_t *registers, int* reghisto)

为uint8_t的数据类型拓展register histogram计算
只用来为批量key的PFCOUNT做提速用

word为uint64_t的registers, 遍历word(0~HLL_REGISTERS/8)
如果*word = 0, reghisto[0]+=8
否则,bytes = (uint8_t*)word
reghisto[bytes[0]]++; 对bytes每一位对应的位置,reghisto增加
```
## 21. hllSigma
```
double hllSigma(double x)
参照"New cardinality estimation algorithms for HyperLogLog sketches"
```
## 22. hllTau
```
double hllTau(double x)
参照"New cardinality estimation algorithms for HyperLogLog sketches"
```
## 23. hllCount
```
uint64_t hllCount(struct hllhdr *hdr, int *invalid)

返回估算的基数值,hdr指向SDS的开始位置
根据hdr->encoding 调用hllDenseRegHisto,hllSparseRegHisto,hllRawReghisto
参照"New cardinality estimation algorithms for HyperLogLog sketches"
计算最终的值
```
## 24. hllAdd
```
int hllAdd(robj *o, unsigned char *ele, size_t elesize)

按照hllencoding 调用hllDenseAdd或者hlSparseAdd,否则返回-1
struct hllhdr *hdr = o->ptr;
```
## 25. hllMerge
```
int hllMerge(uint8_t *max, robj *hll)

max 指向带有array of uint8_t HLL_REGISTERS registers.
hll 计算MAX(registers[i],hll[i]

如果hll是sparse且不可用会返回C_ERR,否则总是成功

struct hllhdr *hdr = hll->prt;
hdr->encoding为HLL_DENSE的场合
循环 i < HLL_REGISTERS 执行HLL_DENSE_GET_REGISTER(val, hdr->registers, i)
val > max[i]则max[i] = val

其他情况,*p = hll->ptr, p += HLL_HDR_SIZE
*end = p + sdslen(hll->ptr)

在p<end的情况下, 判断HLL_SPARSE_IS_ZEOR(p) HLL_SPARSE_IS_XZERO(p)
根据结果不同,p++或p+=2, i记载HLL_SPARSE_ZERO_LEN(p)的和
都不是的情况,p++,i++,
i != HLL_REGISTERS,认为hll不可用，返回C_ERR
```
## 26. createHllObject
```
robj *createHllObject(void)

总是使用sparse encoding创建hll,如果有需要会升级为dense
```
## 27. isHllObjectOrReply
```
int isHllObjectOrReply(client *c, robj *o)

检查object是否是一个具有可用的hll representation的String
```
## 28. pfaddcommand
```
void pfaddCommand(client *c)

PFADD var ele ele ele ... ele => :0 or :1

循环对参数执行hlladd
```
## 29. pfcountCommand
```
void pfcountCommand(client *c)

PFCOUNT var -> approximated cardinality of set
```
## 30. pfmergeCommand
```
void pfmergeCommand(client *c)

PFMERGE dest src1 src2 src3 ... srcN => OK
```
