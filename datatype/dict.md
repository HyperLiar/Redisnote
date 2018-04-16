# dict(哈希表，字典)
## 1. dictEntry
```
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;

字典节点，k,v 本质是链表
```
## 2. dictType
```
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType

字典的具体操作
```
## 3. dictht
```
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

字典数据结构
```
## 4. dict
```
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx;
    unsigned long iterators;
} dict;

真正的字典结构
```
## 5. dictIter
```
typedef struct dictIterator {
    dict *d;
    long index;
    int table, safe;
    dictEntry *entry, *nextEntry;
    long long fingerprint;
} dictIterator;

字典的迭代器
```
## 6. dictFreeVal
```
#define dictFreeVal(d, entry) \
    if ((d)->type->valDestructor) \
        (d)->type->valDestructor((d)->privdata, (entry)->v.val)

定义了valDestructor时，释放entry节点的value
```
## 7. dictSetVal
```
#define dictSetVal(d, entry. _val_) do { \
    if ((d)->type->valDup) \
        (entry)->v.val = (d)->type->valDup((d)->privdata, _val_); \
    else \
        (entry)->v.val = (_val_); \
} while(0)

设置entry的value.
```
## 8. dictSetSignedIntergerVal
```
#define dictSetSignedIntergerVal(entry, _val_) \
    do { (entry)->v.s64 = _val_; } while(0)

设置entry的有符号整型值
```
## 9. dictSetUnsignedIntegerVal
```
#define dictSetUnsignedIntegerVal(entry, _val_) \
    do { (entry)->v.u64 = _val_; } while(0)

设置entry的无符号整型值
```
## 10. dictSetDoubleVal
```
#define dictSetDoubleVal(entry, _val_) \
    do { (entry)->v.d = _val_; } while(0)

设置entry的double值
```
## 11. dictFreeKey
```
#define dictFreeKey(d, entry) \
    if ((d)->type->keyDestructor) \
        (d)->type->keyDestructor((d)->prvidata, (entry)->key)

定义了keyDestructor时， 释放节点的key
```
## 12. dictCompareKeys
```
#define dictCompareKeys(d, key1, key2) \
    (((d)->type->keyCompare) ? \
        (d)->type->keyCompare((d)->privata, key1, key2) : \
        (key1) == (key2))

比较两个key是否相等，相等true 不相等false.
```
## 13. dictHashKey
```
#define dictHashKey(d, key) (d)->type->hashFunction(key)

对key进行hash计算
```
## 14. dictGetKey
```
#define dictGetKey(he) ((he)->key)

获取key
```
## 15. dictGetVal
```
#define dictGetVal(he) ((he)->v.val)

获取value
```
## 16. dictGetSignedIntegerVal
```
#define dictGetSignedIntegerVal(he) ((he)->v.s64)

获取he的有符号整型值
```
## 17. dictGetUnsignedIntegerVal
```
#define dictGetUnsignedIntegerVal(he) ((he)->v.u64)

获取he的无符号整型值
```
## 18. dictGetDoubleVal
```
#define dictGetDoubleVal(he) ((he)->v.d)

获取he的double值
```
## 19. dictSlots
```
#define dictSlots(d) ((d)->ht[0].size+(d)->ht[1].size)

获取字典的size总大小
```
## 20. dictSize
```
#define dictSize((d)->ht[0].used+(d)->ht[1].used)

获取字典已使用的大小
```
## 21. dictIsRehashing
```
#define dictIsRehashing(d) ((d)->rehashidx != -1)

判断字典是否在rehash
```
/* -------------  hash functions ----------- */
## 22. dict_hash_function_seed
```
static uint8_t dict_hash_function_seed[16];

哈希种子
```
## 23. dickSetHashFunctionSeed
```
void dictSetHashFunctionSeed(uint_8 *seed)

将seed memcpy拷贝到dict_hash_function_seed中
```
## 24. dictGetHashFunctionSeed
```
uint8_t *dictGetHashFunctionSeed(void)

返回dict_hash_function_seed
```
## 25. siphash
```
uint64_t siphash(const uint8_t *in, const size_t inlen, const uint8_t *k)

SipHash implementation in siphash.c
```
## 26. siphash_nocase
```
uint64_t siphash_nocase(const uint8_t *in, const size_t inlen, const uint8_t *k)

同25
```
## 27. dictGenHashFunction
```
uint64_t dictGenHashFunction(const void *key, int len)

use siphash(key, len, dict_hash_function_seed))
```
## 28. dictGenCaseHashFunction
```
uint64_t dictGenCaseHashFunction(const unsigned char *buf, int len)

use siphash_nocase(buf, len, dict_hash_function_seed)
```
/* ------------ API implementation --------------*/
## 29. _dictReset
```
static void _dictReset(dictht *ht)

将已初始化的ht置空
```
## 30. dictCreate
```
dict *dictCreate(dictType *type, void *priDataPtr)

按给定的类型初始化dict,使用_dictInit
```
## 31. _dictInit
```
int _dictInit(dict *d, dictType *type, void *privDataPtr)

使用_dictReset设置&d->ht[0], &d->ht[1],
设置其他字段，返回DICK_OK.
```
## 32. dictResize
```
int dictResize(dict *d)

dict无法resize或正在rehash，返回DICT_ERR
否则使用d->ht[0]的used字段与DICT_HT_INITIAL_SIZE比较，
used较小则使用DICT_HT_INITIAL_SIZE
调用dickExpand(d, minimal)
```
## 33. dictExpand
```
int dictExpand(dict *d, unsigned long size)

dict正在rehash或size小于d->ht[0].used，返回DICT_ERR
    --> 在dictResize时已经做过判断，是否多余？
与d->ht[0].size相等，返回DICT_ERR。
创建dictht n, 为n分配空间,如果是第一次初始化
d->ht[0] = n,返回OK
否则，d->ht[1] = n,d->rehashidx = 0，返回OK.
```
## 34. dictRehash
```
int dictRehash(dict *d, int n)

由于可能会访问到很多空的索引，避免因此阻塞过长时间，
增加最大访问次数限制。

定义empty_visits，最大的空桶访问数
d未处于rehash，返回0
第一层循环，n-- && d->ht[0].used != 0，-->对字典的循环
第二层循环，d->ht[0].table[d->rehashidx] == NULL
满足条件,rehashidx自增，若--empty_visits变为0则return 1.

使de = d->ht[0].table[d->rehashidx]
第二层循环， de为真-->对字典特定索引的链表循环
试nextde = de->next, 将de->next指向d->ht[1]的对应位置,
变更ht[0],ht[1]的used字段值，将nextde赋值到de的位置。
结束，将de位置置空，rehashidx自增。

循环结束，判断ht[0].used，全部迁移完毕则释放ht[0]空间，
将ht[1]赋给ht[0]，rehashidx = -1, _dictReset(&d->ht[1]) return 0;
未全部迁移完毕，return 1.
```
## 35. timeInMilliseconds
```
long long timeInMilliseconds(void)

获取当前时间的时间戳毫秒值
```
## 36. dictRehashMilliseconds
```
int dictRehashMilliseconds(dict *d, int ms)

按毫秒级rehash，首先取start = timeInMilliseconds()
每次dictRehash(d, 100), 当当前毫秒时间戳 - start  > ms 时，
停止rehash, 返回rehash的key总数。
```
## 37. _dictRehashStep
```
static void _dictRehashStep(dict *d)

dictRehash(d, 1)，在d->iterators == 0时。
这是因为，如果设置了d->iterator, 可能会导致一些元素的缺失和复制。
```
## 38. dictAdd
```
int dictAdd(dict *d, void *key, void *val)

向dict *d添加一个dictEntry元素
新建entry = dictAddRaw(d, key, NULL)
dictSetVal(d, entry, val)
```
## 39. dictAddRaw
```
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)

用来向dict中增加新的entry,
只是返回了对应key的dictEntry结构，这样用户可以自行设置value.
若d正在rehash,调用一次_dictRehashStep(d)
令index为key在d中的索引，已存在为-1, 此时return NULL
同时会把查找到的存在的dictEntry赋值给existing.
若d正在rehash，把entry加入d->ht[1],否则d->ht[0]
dictSetKey(d, entry, key)设置entry所属的key.
return entry.
```
## 40. dictReplace
```
int dictReplace(dict *d, void *key, void *val)

添加一个新元素，如果key已经存在，舍弃旧的value
新增，则return 1
替换，return 0
先dictSetVal(d, existing, val)，再dictFreeVal(d, &auxentry)
这是因为value可能与旧有的值是相同的。
```
## 41. dictAddOrFind
```
dictEntry *dictAddOrFind(dict *d, void *key)

是dictAddRaw的简化版，总是返回dictEntry，无论key是否已经存在
key无法添加(返回已存在的key)
```
## 42. dictGenericDelete
```
static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree)

搜索，移除一个元素
d->ht[0],d->ht[1]都为空时返回NULL
处于rehash时，调用_dictRehashStep(d)
对ht[0],ht[1]都做如下操作:
h = dictHashKey(d, key)
idx = h & d->ht[table].sizemask
取he = d->ht[table].table[idx]，
当其为真时，如果key与he->key相等，
判断he是否有前置节点，有需要调整链表
否则直接设置d->ht[table].table[idx]为he->next
nofree为假时，dickFreeKey(d, he) dictFreeVal(d, he)

之后减小used, 返回he
不相等时，令he = he->next

ht[0]循环结束，如果dict没有处在rehash状态，跳出循环
```
## 43. dictDelete
```
int dictDelete(dict *ht, const void *key)

用于查找并删除一个元素
调用dictGenericDelete(ht, key, 0),
根据其是否为空，返回DICK_ERR或DICK_OK
```
## 44. dictUnlink
```
dickEntry *dictUnlink(dick *ht, const void *key)

用于删除一个元素，但并不会真正释放key,value和dictEntry
如果查找到了元素，会返回它。
调用dictGenericDelete(ht, key, 1)

这个函数是为了真正移除一个元素时，不需要进行多次的查找：

entry = dictFind(...);
dictDelete(dictionary, entry);--> the second lookup

entry = dictUnlink(dictionary, entry);
dictFreeUnlinkedEntry(entry);--> does not need to lookup again
```
## 45. dictFreeUnlinkedEntry
```
void dictFreeUnlinkedEntry(dict *d, dictEntry *he)

释放一个已经被unlink的dictEntry:
dictFreekey(d, he);
dictFreeVal(d, he);
zfree(he);
```
## 46. _dictClear
```
int _dictClear(dict *d, dicthe *ht, void(callback)(void *))

清空并释放完整的字典结构
循环条件i < ht->size & ht->used > 0
在i & 65535 == 0时，调用callback(d->privdata)
he = ht->table[i] == 0时继续循环
否则将he从链表中移除并释放空间，he = nextHe
最终释放ht->table, _dictReset(ht)--> 重新初始化table
返回DICT_OK
```
## 47. dictRelease
```
void dictRelease(dict *d)

清空并释放整个哈希表
_dictClear(d, &d->ht[0], NULL);
_dictClear(d, &d->ht[1], NULL);
zfree(d)
```
## 48. dictFind
```
dictEntry *dictFind(dict *d, const void *key)

在dict结构中，找到给定的key对应的元素。
d->ht[0], d->ht[1]都会进行查找
```
## 49. dictFetchValue
```
void *dictFetchValue(dict *d, const void *key)

返回dict中，对应key对应的value
调用dictFind, 获取dictEntry后调用dictGetVal(he)
```
## 50. dictFingerprint
```
long long dictFingerprint(dict *d)

fingerprint是一个64bit的数字，代表dict在某个时间的状态
只是几个属性的异或值
初始化一个不安全的迭代器时生成fingerprint, 释放时校验。
用fingerprint保证不会有不安全的迭代器对dict进行迭代。
```
## 51. dictGetIterator
```
dictIterator *dictGetIterator(dict *d)

返回一个在d上的空iterator
```
## 52. dictGetSafeIterator
```
dictIterator *dictGetSafeIterator(dict *d)

调用dictGetIterator，使iterator->safe = 1
```
## 53. dictNext
```
dictEntry *dictNext(dictIterator *iter)

返回iter的nextEntry，注意在找到非空的entry时，要在
迭代器中保存这个entry的next，因为可能当前的entry会被释放掉

iter->entry为空
第一次iter开始时
如果iter是安全的，那么每次增长iter->d->iterators的值，
否则存储iter->fingerprint

每次增加iter->index的值，当index>ht->size时
如果正在hash，且iter ht[0]，则设置为iter ht[1]
否则break跳出循环
置iter->entrt = ht->table[iter->index]

否则，iter->entry = iter->nextEntry

如果iter->entry不为空，返回。
```
## 54. dictReleaseIterator
```
void dictReleaseIterator(dictIterator *iter)

如果iter->index 不为-1，或iter->table != 0
安全时，减小iter->d->iterators
不安全断言fingerprint
释放iter
```
## 55. dictGetRandomKey
```
dictEntry *dictGetRandomKey(dict *d)

返回一个随机的entry
处于rehash中，只从ht[0]中选取
否则在ht[0], ht[1]中随机选择一个桶，
之后循环拿到桶中链表的数量，随机选择。
```
## 56. dictGetSomeKeys
```
unsigned int dictGetSomeKeys(dict *d, dictEntry **des, unsigned int count)

返回一组entry，保存在dsc中
```
## 57. rev
```
statis unsigned long rev(unsigned long v)

reverse bits.
```
## 58. dictScan
```
unsigned long dictScan(dict *d, unsigned long v,
                       dictScanFunction *fn,
                       dictScanBucketFUnction *bucketfn,
                       void *privdata)

用于扫描整个dict，每完成一次迭代，返回一个cursor，
下次调用需要使用cursor,返回0时表示迭代结束

保证会循环到所有的元素，但是有些元素可能被多次返回
每个返回的元素,调用fn(privdata, de)

从较高的bit位开始迭代，因为ht可能在迭代期间被resize.

如果hash的size变大了，
如果变小，
而rehash时有两个表，故每次都是从较小的table开始迭代，
之后测试在更大的表中是否有当前的cursor的任何拓展

优点在于: iterator是无状态的，且不需要任何额外的内存
缺点在于：可能多次返回同一个元素
          如果要返回多个元素，可能每次都需要访问桶中
相连的所有元素和所有的expansions，这是为了保证rehashing时不会miss keys.


如果没处于rehash，直接v & mask, 取对应的de，调用fn.
--> 迭代完de桶内的所有元素
否则的话，先迭代较小的ht，
同样 v & mask0, 取de调用fn

对v, mask1做相同操作，

increment bits not covered by the smaller mask.
v = (((v | m0) + 1) & ~m0) | ( v & m0);
continue while bits covered by mask diff is non-zero
while ( v & (mo ^ m1))

Set unmasked bits so incrementing the reversed cursor
operates on the masked bits of the smaller table
v |= ~m0;

v = rev(v); v++; v = rev(v);

返回v.
(TODO:暂时没看懂)
```
## 59. _dictExpandIfNeeded
```
static int _dictExpandIfNeeded(dict *d)

已经处于rehash,返回DICT_OK
如果ht[0]为空，返回dictExpand(d, DICT_HT_INITIAL_SIZE)
如果ht[0].used多于size，且 (全局配置可以resize或
used/size比例大于dict_force_resize_ratio)
返回dictExpand(d, d->ht[0].used*2)
返回DICT_OK
```
## 60. _dictNextPower
```
static unsigned long _dictNextPower(unsigned long size)

计算hash的下一个容量(总是2的幂)
如果size >= LONG_MAX，返回LONG_MAX + 1LU
i = DICT_HT_INITIAL_SIZE，每次循环乘2
当i > size时，返回i
```
## 61. _dictKeyIndex
```
static long _dictKeyIndex(dict *d, const void *key, uint64_t hash, dictEntry **existring)

返回空闲位置是否被一个给定key代表的hash entry填充
_dictExpandIfNeeded(d) == DICT_ERR，返回-1
对tab循环，找到hash对应的idx, 返回对应位置的he
如果对应he->key和给定的key相等，返回-1，一直循环到he->next为空
如果处在rehash状态，会继续循环d->ht[1]
也就是返回idx = hash & d->ht[1].sizemask
抛弃ht[0]的sizemask
```
## 62. dictEmpty
```
void dictEmpty(dict *d, void(callback)(void*))

执行_dictClear清空ht[0]和ht[1]
置rehashidx = -1
iterators = 0
```
## 63. dictEnableResize
```
void dictEnableResize(void)

设置dict_can_resize = 1
```
## 64. dictDisableResize
void dictDisableResize(void)
```
void dictDisableResize(void)

设置dict_can_resize = 0
```
## 65. dictGetHash
```
uint64_t dictGetHash(dict *d, const void *key)

return dictHashKey)d, key)
```
## 66. dictFindEntryRefByPtrAndHash
```
dictEntry **dictFindEntryRefByPtrAndHash(dict *d, const void *oldptr, uint64_t hash)

靠指针和预计算的hash值找到dictEntry的引用
不执行任何key/string comparision
如果对应的引用是空返回NULL

dict为空，返回NULL
对ht[0] ht[1]，heref为对应位置的引用，
如果oldptr == he->key，返回heref

否则heref = &he->next，继续比较到NULL为止
如果未处在rehash状态，不会在ht[1]查找
```
/* --------------- debugging ----------------*/
## 67. _dictGetStatsHt
```
size_t _dictGetStatsHt(char *buf, size_t bufsize, dictht *ht, int tableid)
```
## 68. dictGetStats
```
void dictGetStats(char *buf, size_t bufsize, dict *d_
```
/* --------------- benchmark ---------------- */
## 69. hashCallback
```
uint64_t hashCallback(const void *key)
```
## 70. compareCallback
```
int compareCallback(void *privdata, const void *key1, const void *key2)
```
## 71. freeCallback
```
void freeCallback(void *privdata, void *val)
```
## 72. BenchmarkDictType
```
dictType BenchmarkDictType = {
    hashCallback,
    NULL,
    compareCallback,
    freeCallback,
    NULL
```
## 73. start_benchmark
```
#define start_benchmark() start = timeInMilliseconds()
```
## 74. end_benchmark
```
#define end_benchmark(msg) do { \
    elapsed = timeInMilliseconds()-start; \
    printf(msg ": %ld items in %lld ms\n", count, elapsed); \
} while(0)
```
