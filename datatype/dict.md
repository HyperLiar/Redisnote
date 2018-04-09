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
