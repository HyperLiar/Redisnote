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
