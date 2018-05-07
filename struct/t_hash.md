# t_hash
## 1. hashTypeTryConversion
```
void hashTypeTryConversion(robj *o, robj **argv, int start, int end)

encoding必须为OBJ_ENCODING_ZIPLIST
检查objects数量的长度，判断是否需要将其从ziplist转化为真正的hash
符合条件调用hashTypeConvert
```
## 2. hashTypeGetFromZiplist
```
int hashTypeGetFromZiplist(robj *o, sds field, unsigned char **vstr
unsigned int *vlen, long long *vll)

从ziplist中获取field标志的value
fptr = ziplistIndex(zl, ZIPLIST_HEAD) 头指针
fptr = ziplistFind(fptr, (unsigned char*)field, sdslen(field)) 找到目标指针 指向的是field的pointer
vptr = ziplistNext(zl, fptr) 这是value的pointer
ziplistGet(vptr, vstr, vlen, vll)
成功返回0，否则返回-1
```
## 3. hashTypeGetFromHashTable
```
sds hashTypeGetFromHashTable(robj *o, sds field)

从hash table encoded hash中按field获取value
未找到返回NULL, 否则返回sds value
de = dictFind(o->ptr, field)
return dictGetVal(de)
```
## 4. hashTypeGetValue
```
int hashTypeGetValue(robj *o, sds field, unsigned char **vstr,
unsigned int *vlen, long long *vll)

高级的hashTypeGet*(), 返回和特定的field关联的hash value
找到的时候返回C_OK 否则C_ERR
string时存储在vstr和vlen
long long存储在vll

根据encoding不同调用
hashTypeGetFromZiplist
hashTypeGetFromHashTable
```
## 5. hashTypeGetValueObject
```
robj *hashTypeGetValueObject(robj *o, sds field)

类似于hashTypeGetValue，返回一个robj

hashTypeGetValue,根据string或是number
createStringObject()
createStringObjectFromLongLong()
```
## 6. hashTypeGetValueLength
```
size_t hashTypeGetValueLength(robj *o, sds field)

高级的hashTypeGet*()返回和特定的field关联的object的长度
不存在返回0

hashTypeGetFrom*,之后调用sdslen
```
## 7. hashTypeExists
```
int hashTypeExists(robj *o, sds field)

观察特定的field是否在给定的hash中存在
也是调用hashTyoeGetFrom*
```
## 8. hashTypeSet
```
#define HASH_SET_TAKE_FIELD (1<<0)
#define HASH_SET_TAKE_VALUE (1<<1)
#define HASH_SET_COPY 0
int hashTypeSet(robj *o, sds field, sds value, int flags)

添加新的field, 如果已存在覆盖
insert返回0 update返回1
flags用来标志保留field或者value 

ziplist时，ziplistIndex
非空 ziplistFind, ziplistNext拿到value,
ziplistDelete, ziplistInsert插入新的值

空 ziplistPush ziplistPush field和value

检查是否需要转变为一个hash table

ht时 dictFind
非空, sdsfree(dictGetVal(de))
flags && HASH_SET_TAKE_VALUE, dictGetVal(de) = value value = NULL
否则dictGetVal(de) = sdsdup(value)

空 flags & HASH_SET_TAKE_FIELD f = field, field = NULL
否则 f = sdsdup(field)
flags & HASH_SET_TAKE_VALUE v = value, value = NULL
否则 v = sdsdup(value)

调用dictAdd(o->ptr, f, v)

根据flags & HASH_SET_TAKE_FIELD/VALUE && field/value
sdsfree(field/value)
返回update的值
```
## 9. hashTypeDelete
```
int hashTypeDelete(robj *o, sds field)

从hash中删除元素 删除返回1 not found返回0
ziplist
ziplistIdnex
zuolistFind
两次ziplistDelete

ht
dictDelete
删除成功, htNeedsResize->dictResize
```
## 10. hashTypeLength
```
unsigned long hashTypeLength(const robj *o)
返回hash中的元素个数
ziplist时, ziplistLen(o->ptr)/2
ht时, dictSize((const dict*)o->ptr)
```
## 11. hashTypeInitIterator
```
hashTypeIterator *hashTypeInitIterator(robj *subject)

初始 hash的iterator
ziplist时, 设置fptr, vptr为NULL
ht时, di = dictGetIterator(subject->ptr)
```
## 12. hashTypeReleaseIterator
```
void hashTypeRelaseIterator(hashTypeIterator *hi)

是ht时, dictReleaseIterator(hi->di)

zfree(hi)
```
## 13. hashTypeNext
```
int hashTypeNext(hashTypeIterator *hi)

移动到下一个entry, C_OK, C_ERR

ziplist
fptr = hi->fptr
vptr = hi->vptr
fptr 为空
fptr = ziplistindex(zl, 0)
否则
fptr = ziplistNext(zl, vptr)

vptr = ziplistNext(zl, fptr)
设置hi->fptr = fptr, hi->vptr = vptr

ht
hi->de = dictNext(hi->di)
```
## 14. hashTypeCurrentFromZiplist
```
void hashTypeCurrentFromZiplist（hashTypeIterator *hi, int what,
unsigned char **vstr, unsigned int *vlen, long long *vll)

获取目前iterator cursor的field或value, 按what判断
what & OBJ_HASH_KEY, ziplistGet(hi->fptr,vstr,vlen.,vll)
ziplistGet(hi->vptr,vstr,vlen,vll)
```
## 15. hashTypeCurrentFromHashTable
```
sds hashTypeCurrentFromHashTable(hashTypeIterator *hi, int what)

同15, 从ht中获取档期iterator cursor的field或value
```
## 16. hashTypeCurrentObject
```
void hashTypeCurrentObject(hashTypeIterator *hi, int what,
unsigned char **vstr, unsigned int *vlen, long long *vll)

将返回的引用存储来vstr,vlen或vll中
hashTypeCurrentFrom*
```
## 17. hashTypeCurrentObjectNewSds
```
sds hashTypeCurrentObjectNewSds(hashTypeIterator *hi, int what)

返回当前iterator位置的key/value作为sds string
hashTypeCurrentObject, sdsnewlen或sdsfromlonglong
```
## 18. hashTypeLookupWriteOrCreate
```
robj *hashTypeLookupWriteOrCreate(client *c, robj *key)

查找指定key对应的hash object是否存在,不存在则创建
```
## 19. hashTypeConvertZiplist
```
void hashTypeConvertZiplist(robj *o, int enc)

将ziplist变为dict结构
新建一个dict, while(hashTypeNext(hi)) 遍历ziplist
将值不断dictAdd到dict中, o->ptr = dict
```
## 20. hashTypeConvert
```
void hashTypeConvert(robj *o, int enc)

检查o->encoding, 为OBJ_ENCODING_ZIPLIST时
hashTypeConvertZiplist(o, enc)
```
## 21. hsetnxCommand
```
void hsetnxCommand(client *c)

o = hashTypeLookupWriteOrCreate(c, c->argv[1])
hashTypeTryConversion(o, c->argv, 2, 3)
存在hash table, 什么也不做
否则 hashTypeSet(o,c->argv[2]->ptr,c->argv[3]->ptr,HASH_SET_COPY)
```
## 22. hsetCommand
```
void hsetCommand(client *c)

hmset 需要c->argc为偶数
循环hashTypeSet
hmset和hset的返回值不同
```
## 23. hincrbyCommand
```
void hincrbyCommand(client *c)

增加hash table中某个key对应的值
会检查增加后是否溢出
hashTypeGetValue(o, c->argv[2]->ptr, &vstr, &vlen, &value)
hashTypeSet
```
## 24. hincrbyfloatCommand
```
void hincrbyfloatCommand(client *c)

保存的是字符串 不会进行溢出检查
最终结果按hset处理, 保证精度不会有差异
```
## 25. addHashFieldToReply
```
static void addHashFieldToReply(client *c, robj *o, sds field)

公用的get方法，返回field对应的value
将hash的field对应的value加入reply中
hashTypeGetFrom*
```
## 26. hgetCommand
```
void hgetCommand(client *c)

o = lookupKeyReadOrReply()
addHashFieldToReply
```
## 27. hmgetCommand
```
void hmgetCommand(client *c)

o = lookupKeyRead()
NULL的key 只要返回空就可以了
循环执行addHashFieldToReply
```
## 28. hdelCommand
```
void hdelCommand(client *c)

o = lookupKeyWriteOrReply()

循环 hashTypeDelete
hashTypeLength(o) == 0时，dbDelete
```
## 29. hlenCommand
```
void hlenCommand(client *c)

o = lookupKeyReadOrReply()
addReplyLongLong(c, hashTypeLength(o)
```
## 30. hstrlenCommand
```
void hstrlenCommand(client *c)

返回field关联的value的长度
o = lookupKeyReadOrReply()
hashTypeGetValueLength(o, c->argv[2]->ptr)
```
## 31. addHashIteratorCursorToReply
```
static void addHashIteratorCursorToReply(client *c, hashTypeIterator *hi, int what)

hashTypeCurrentFrom*, addReplyBulkCBuffer()/addReplyBulkLongLong()
```
## 32. genericHgetallCommand
```
void genericHgetallCommand(client *c, int flags)

o = lookupReadOrReply()
hi = hashTypeInitIterator(o)
while (hashTypeNext(hi) != C_ERR)
addHashIteratorCursorToReply(c, hi, OBJ_HASH_KEY/VALUE)
```
## 33. hkeysCommand
```
void hkeysCommand(client *c)

genericHgetallCommand(c, OBJ_HASH_KEY)
```
## 34. hvalsCommand
```
void hvalsCommand(client *c)

genericHgetallCommand(c, OBJ_HASH_VALUE)
```
## 35. hgetallCommand
```
void hgetallCommand(client *c)

genericHgetallCommand(c, OBJ_HASH_KEY|OBJ_HASH_VALUE)
```
## 36. hexistsCommand
```
void hexistsCommand(client *c)

查看给定field是否存在
```
## 37. hscanCommand
```
void hscanCommand(client *c)

o = lookupKeyReadOrReply(c,c->argv[1],shared.emptyscan)
scanGenericCommand(c, o, cursor)
```
