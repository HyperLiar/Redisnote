# object
## 1. createObject
```
robj *createObject(int type, void *ptr)

o->type = type;
o->encoding = OBJ_ENCODING_RAW;
o->ptr = ptr;
o->refcount = 1;
LRU设置为当前的lruclock 或者选择性设为LFU counter
o->lru = LRUK_CLOCK();
o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
```
## 2. makeObjectShared
```
robj *makeObjectShared(robj *o)

o->refcount = OBJ_SHARE_REFCOUNT;
为object设置一个特别的refcount

robj *myobject = makeObjectShared(createObject(...));
```
## 3. createRawString
```
robj *createRawStringObject(const char *ptr, size_t len)

创建一个OBJ_ENCODING_RAW encoding的string object
return createObject(OBJ_STRING, sdsnewlen(ptr, len))
```
## 4. createEmbeddedStringObject
```
robj *createEmbeddedStringObject(const char *ptr, size_t len)

创建一个OBJ_ENCODING_EMBSTR encoding的string object
会初始化object->buf为0
```
## 5. createStringObject
```
robj *createStringObject(const char *ptr, size_t len)

#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
根据len的长度使用3 或者 4
```
## 6. createStringObjectFromLongLong
```
robj *createStringObjectFromLongLong(long long value)

根据value值的大小使用不同的初始化方式
```
## 7. createStringObjectFromLongDouble
```
robj *createStringObjectFromLongDouble(long double value, int humanfriendly)

humanfriendly非0则不使用指数格式,去除尾部的0,会导致精度丢失
```
## 8. dupStringObject
```
robj *dupStringObject(const robj *o)

复制一个object,保证返回的object encoding与原始相同
refcount会设置为1
```
## 9. createQuicklistObject
```
robj *createOuicklistObject(void)

新建quicklist对应的object
```
## 10. createZiplistObject
```
robj *createZiplistObject(void)

新建ziplist对应的object
```
## 20. createSetObject
```
robj *createSetObject(void)

新建dict对应的object
encoding为OBJ_ENCODING_HT
```
## 21. createIntsetObject
```
robj *createIntsetObject(void)

新建intset对应的object
```
## 22. createHashObject
```
robj *createHashObject(void)

和createZiplistObject相同,但type为OBJ_HASH
encoding为OBJ_ENCODING_ZIPLIST
```
## 23. createZsetObject
```
robj *createZsetObject(void)

encoding为OBJ_ENCODING_SKIPLIST
```
## 24. createZsetZiplistObject
```
robj *createZsetZiplistObject(void)

type为OBJ_ZSET, encoding为OBJ_ENCODING_ZIPLIST
```
## 25. createStreamObject
```
robj *createStreamObject(void)

type: OBJ_STREAM, encoding: OBJ_ENCODING_STREAM
```
## 26. createModuleObject
```
robj *createModuleObject(moduleType *mt, void *value)

type: OBJ_MODULE
```
## 27. freeStringObject
```
void freeStringObject(robj *o)
encoding为OBJ_ENCODING_RAW时, sdsfree(o->ptr)
```
## 28. freeListObject
```
void freeListObject(robj *o)

quicklistRelease
```
## 29. freeSetObject
```
void freeSetObject(robj *o)

OBJ_ENCODING_HT: dictRelease((dict*) o->ptr)
OBJ_ENCODING_INTSET: zfree(o->ptr)
```
## 30. freeZsetObject
```
void freeZsetObject(robj *o)

OBJ_ENCODING_SKIPLIST: dictReleast(zs->dict) zslFree(zs->zsl) zfree(zs)
OBJ_ENCODING_ZIPLIST: zfree(o->ptr)
```
## 31. freeHashObject
```
void freeHashObject(robj *o)

OBJ_ENCODING_HT: dictRelease((dict *) o->ptr)
OBJ_ENCODING_ZIPLIST: zfree(o->ptr)
```
## 32. freeModuleObject
```
void freeModuleObject(robj *o)

mv = o->ptr
mv->type->free(mv->value)
zfree(mv)
```
## 33. freeStreamObject
```
void freeStreamObject(robj *o)

freeStream(o->ptr)
```
## 34. incrRefCount
```
void incrRefCount(robj *o)

o->refcount不是共享时,自增
```
## 35. decrRefCount
```
void decrRefCount(robj *o)

如果refcount == 1,执行对应type的free方法,最终zfree(o)
否则大于0且不是share值时,自减
```
## 36. decrRefCountVoid
```
void decrRefCountVoid(void *o)

decrRefCount(o) 区别在于参数为void *
```
## 37. resetRefCount
```
robj *resetRefCount(robj *robj)

设置refcount为0,方便供自增refcount的函数调用
不用新建object
```
## 38. checkType
```
int checkType(client *c, robj *o, int type)

检查o->type是否和type一致
```
## 39. isSdsRepresentableAsLongLong
```
int isSdsRepresentableAsLongLong(sds s, long long *llval)

return string2ll(s, sdslen(s), llval) ? C_OK : C_ERR;
```
## 40. isObjectRepresenttableAsLongLong
```
int isObjectRepresentableAsLongLong(robj *o, long long *llval)

如果encoding为OBJ_ENCODING_INT
返回C_OK

否则返回isSdsRepresentableAsLongLong(o->ptr, llval)
```
## 41. tryObjectEncoding
```
robj *tryObjectEncoding(robj *o)

规矩挺多的
```
## 42. getDecodedObject
```
robj *getDecodedObject(robj *o)

如果sdsEncodedObject(o), incrRefCount(o)
```
## 43. compareStringObjectsWithFlags
```
int compareStringObjectWithFlags(robj *a, robj *b, int flags)

#define REDIS_COMPARE_BINARY (1<<0)
#define REDIS_COMPARE_COLL (1<<1)
比较两个string object,按flag判断使用strcmp或strcoll
必须被integer-encoded
```
## 44. compareStringObjects
```
int compareStringObjects(robj *a, robj *b)

return compareStringObjectsWithFlags(a, b, REDIS_COMPARE_BINARY)
```
## 45. collateStringObjects
```
int collateStringObjects(robj *a, robj *b)

return compareStringObjectsWithFlasg(a, b, REDIS_COMPARE_POLL)
```
## 46. equalStringObjects
```
int equalStringObjects(robj *a, robj *b)

encoding都是int时,返回a->ptr == b->ptr
否则返回compareStringObjects(a, b)
```
## 47. stringObjectLen
```
size_t stringObjectLen(robj *o)

if(sdsEncodedObject(o)) return sdslen(o->ptr)
else return sdigits10((long)o->ptr)
```
## 48. getDoubleFromObject
```
int getDoubleFromObject(const robj *o, double *target)

if(sdsEncodedObject(o)) value = strtod(o->ptr, &eptr)
elseif (o->encoding == OBJ_ENCODING_INT) value = (long)o->ptr
*target = value
```
## 49. getDoubleFromObjectOrReply
```
int getDoubleFromObjectOrReply(client *c, robj *o, double *target, const char *msg)

getDoubleFromObject失败时,msg不为空则addReplyError(c, (char*)msg)
为空addReplyError(c, "value is not a valid float")
```
## 50. getLongDoubleFromObject
```
int getLongDoubleFromObject(robj *o, long double *target)

类似于getLongDoubleFromObject
value = strtold(o->ptr, &eptr)
value = (long)o->ptr
```
## 51. getLongDoubleFromObjectOrReply
```
int getLongDoubleFromObjectOrReply(client *c, robj *o, long double *target, const char *msg)

类似于getDoubleFromObjectOrReply
```
## 52. getLongLongFromObject
```
int getLongLongFromobject(robj *o, long long *target)

string2ll(o->ptr, sdslen(o->ptr), &value)
(long)o->ptr
```
## 53. getLongLongFromObjectOrReply
```
int getLongLongFromObjectOrReply(client *c, robj *o, long long *target, const char *msg)
```
## 54. getLongFromObjectOrReply
```
int getLongFromObjectOrReply(client *c, robj *o, long *target, const char *msg)
```
## 55. strEncoding
```
char *strEncoding(int encoding)

按encoding返回相应的字符串
```
## 56. streamRadixTreeMemoryUsage
```
size_t streamRadixTreeMemoryUsage(rax *rax)

估算用来存储stream ids的radix tree的内存大小
会加上一个overhead
```
## 57. objectComputeSize
```
#define OBJ_COMPUTE_SIZE_DEF_SAMPLES 5
size_t objectComputeSize(robj *o, size_t sample_size)
返回估计的size in bytes.
根据type不同, size的计算方式有所不同
```
## 58. freeMemoryOverheadData
```
void freeMemoryOverheadData(struct redisMemOverhead *mh)

释放getMemoryOverheadData()得到的数据
zfree(mh->db)
zfree(mh)
```
## 59. getMemoryOverheadData
```
struct redisMemOverhead *getMemoryOverheadData(void)

返回redisMemOverhead结构,保存MEMORY OVERHEAD和INFO命令需要的
memory overhead信息
```
## 60. inputCatSds
```
void inputCatSds(void *result, const char *str)

sds *info = (sds *)result *info = sdscat(*info, str)
```
## 61. getMemoryDocterReport
```
sds getMemoryDoctorRepost(void)

返回人类可读的内存信息
```
## 62. objectCommandLookup
```
robj *objectCommandLookup(client *c, robj *key)

de = dictFind(c->db->dict, key->ptr)
return dictGetVal(de)
```
## 63. objectCommandLookupOrReply
```
robj *objectCommandLookupOrReply(client *c, robj *key, robj *reply)

调用objectCommandLookUp, 为false addReply(c, reply)
```
## 64. objectCommand
```
void objectCommand(client *c)

匹配使用strcasecmp(c->argv[1]->ptr, "")
OBJECT <refcount|encoding|idletime|freq> <key>
如果只有两个参数,且第二个为help返回帮助信息
refcount时o = objectCommandLookupOrReply(c, c->argv[2], shared.nullbulk)

```
