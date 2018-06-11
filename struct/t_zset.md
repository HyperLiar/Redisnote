# t_zset
## 1. zzlGetScore
```
double zzlGetScore(unsigned char *sptr)

ziplistGet(sptr, &vstr, &vlen, &vlong)
vstr: memcpy(bufm,vstr,vlen) score = strtod();
score = vlong;
```
## 2. ziplistGetObject
```
sds ziplistGetObject(unsigned char *sptr)

将一个ziplist元素返回为一个SDS string
ziplistGet, sdsnewlen或者sdsfromlonglong
```
## 3. zzlCompareElements
```
int zzlCompareElements(unsigned char *eptr,unsigned char *cstr, unsigned int clen)

memcmp,比较给定的ele和eptr的ele
ziplistGet, 比较两者最小长度. 
```
## 4. zzlLength
```
unsigned int zzlLength(unsigned char *zl)
return ziplistLen(zl)/2
```
## 5. zzlNext
```
void zzlNext(unsigned char *zl, unsigned char **eptr, unsigned char **sptr)

_eptr = ziplistNext(zl, *sptr)
_sptr = ziplistNext(zl, _eptr)
在eptr,sptr基础上,移动到下一个entry, 并赋值给*eptr, *sptr
```
## 6. zslPrev
```
void zzlPrev(unsigned char *zl, unsigned char **eptr, unsigned char **sptr)

在eptr,sptr基础上,移动到前一个entry
调用1-2次ziplistPrev
```
## 7. zzlIsInRange
```
int zzlIsInRange(unsigned char *zl, zrangespec *range)

返回zl是否有部分在range内

需要调用两次ziplistIndex,zzlGetScore, 判断首尾
```
## 8. zzlFirstInRange
```
unsigned char *zzlFirstInRange(unsigned char *zl, zrangespec *range)

找到特定范围内的第一个元素
while, ziplistNExt, zzlGetScore, 比较score是否在range内
```
## 9. zzlLastInRange
```
unsigned char *zzlLastInRange(unsigned char *zl, zrangespec *range)
```
## 10. zzlGetScore
```
double zzlGetScore(unsigned char *sptr)

使用ziplistGet, 获取value
字符串的场合 memcpy,strtod,否则直接赋值
返回score
```
## 11. ziplistGetObject
```
sds ziplistGetObject(unsigned char *sptr)

返回一个ziplist内的元素
ziplistGet(sptr, &vstr, &vlen, &vlong)
vstr不空， sdsnewlen((char*)vstr, vlen)
否则 sdsfromlonglong(vlong)
```
## 12. zzlCompareElements
```
int zzlCompareElements(unsigned char *eptr, unsigned char *cstr, unsigned int clen)

比较eptr cstr位置的元素
memcmp, 结果为0时返回vlen - clen
```
## 13. zzlNext
```
void zzlNext(unsigned char *zl, unsigned char **eptr, unsigned char **sptr)

ziplistNext，下一个值为空时设置为NULL
eptr, sptr保存ele value
```
## 14. zzlPrev
```
void zzlPrev(unsigned char *zl, unsigned char **eptr, unsigned char **sptr)

_sptr = ziplistPrev(zl,*eptr)
_eptr = ziplistPrev(zl,_sptr)
然后*eptr = _eptr *sptr = _sptr
```
## 15. zzlIsInRange
```
int zzlIsInRange(unsigned char *zl,zrangespec *range)

返回是否有range内的zset部分 只能用在zzlFirstInRange zzlLatInRange

p = ziplistIndex(zl, -1) 检查最后一个元素和最小值
p = ziplistIndex(zl, 1) 检查第一个元素和最大值
都为真返回1 否则0
```
## 16. zzlFirstInRange
```
unsigned char *zzlFirstInRange(unsigned char *zl, zrangespec *range)

找到特定range内的第一个元素
zzlIsInRange(zl, range)
eptr = ziplistIndex(zl, 0)
while循环
sptr = ziplistNext(zl, eptr)
判断score = zzlGetScore(sptr) 是否在范围内
eptr = ziplistNext(zl, sptr)
```
## 17. zzlLastInRange
```
unsigned char *zzlLastInRange(unsigned char *zl, zrangespec *range)

类似于zzlFristInRange
```
## 18. zzlLexValueGteMin
```
int zzlLexValueGteMin(unsigned char *p, zlexrangespec *spec)

sds value = ziplistGetObject(p)
int res = zslLexValueGteMin(value, spec)
sdsfree(value)
return res
```
## 19. zzlLexValueLteMax
```
int zzlLexValueLteMax(unsigned char *p, zlexrangespec *spec)

同zzlLexValueLteMax
```
## 20. zzlIsInLexRange
```
int zzlIsInLexRange(unsigned char *zl, zlexrangespec *range)

返回是否有部分zset在字典序内
```
## 21. zzlFirstInLexRange
```
unsigned char *zzlFirstInLexRange(unsigned char *zl, zlenrangespec *range)

找到字典范围内第一个元素，while zzlLextGteMin zzlLexValueLteMax判断
ziplistNext ziplistNext
```
## 22. zzlLastInLexRange
```
unsigned char *zzlLastInLexRange(unsigned char *zl, zlexrangespec *range)

字典范围内最后一个元素
```
## 23. zzlFind
```
unsigned char *zzlFind(unsigned char *zl, sds ele, double *score)

查找匹配ele的key，返回其地址, 将score保存在*score内
``` 
## 24. zzlDelete
```
unsigned char *zzlDelete (unsigned char *zl, unsigned char *eptr)

调用两次ziplistDelete(zl, &p)
```
## 25. zzlInsertAt
```
unsigned char *zzlInsertAt(unsigned char *zl, unsigned char *eptr, sds ele, double score)

eptr为空，两次ziplistPush
否则 ziplistInsert(zl, eptr, (unsigned char*)ele, sdslen(ele));
在ele后插入score
```
## 26. zzlInsert
```
unsigned char *zzlInsert(unsigned char *zl, sds ele, double score)

遍历zl, 如果score比给定的大，插入
否则score相等时，比较ele eptr的ele较大时，插入给定的元素
否则 ziplistNext(zl, sptr)

如果还是没有被插入, 将它push到ziplist的末尾
```
## 27. zzlDeleteRangeByScore
```
unsigned char *zzlDeleteRangeByScore(unsigned char *zl, zrangespec *range, unsigned long *deleted)

删除score值范围内的所有元素 deleted保存删除的元素个数
注意也是while zzlGetScore ziplistDelete两次
```
## 28. zzlDeleteRangeByLex
```
unsigned char *zzlDeleteRangeByLex(unsigned char *zl, zlexrangespec *range, unsigned long *deleted)

zslValueLteMax(score, range) zzlGetScore ziplistDelete两次
```
## 29. zzlDeleteRangeByRank
```
unsigned char *zzlDeleteRangeByRank(unsigned char *zl, unsigned int start, unsigned int end, unsigned long *deleted)

删除排序在strat-end间的所有元素 两者包含在内，注意1-base
ziplistDeleteRange
```
## 30. zsetLength
```
unsigned int zsetLength(const robj *zobj)

ziplist, 使用zzlLength(zobj->ptr)
skiplist ((const zset*)zobj->ptr)->zsl->length
```
## 31. zsetConvet
```
void zsetConvert(robj *zobj, int encoding)

转换zset的编码
ziplist 新建字典，skiplist while循环zslInsert, dictAdd
skiplist 新建ziplist while循环 zzlInsertAt
```
## 32. zsetConvertToZiplistIfNeeded
```
void zsetConvertToZiplistIfNeeded(robj *zobj, size_t maxelelen)

zsl->length <= server.zset_max_ziplist_entries 且
maxelelen <= server.zset_max_ziplist_value
调用zsetConvert
```
## 33. zsetScore
```
int zsetScore(robj *zobj, sds member, double *score)

返回member位置的元素, score存在*score中
存在返回C_OK，不存在，zobj/member为空，返回C_ERR

ziplist: zzlfind
dictEntry de = dictFind(zobj->ptr->dict) score = dictGetVal(de)
```
## 34. zsetAdd
```
int zsetAdd(robj *zobj, double score, sds ele, int *flags, double *newscore)

添加元素，或更新存在的元素的score
flags: ZADD_INCR 增加当前score对应元素的score 元素不存在时，假设之前的score为0
ZADD_NX 元素不存在时执行命令
ZADD_XX 元素已存在时执行命令

使用ZADD_INCR时，新的score被存储在*newscore

返回的flags分别是:
ZADD_NAN 结果score不是一个数字
ZADD_ADDED 元素被增加(与调用前不同)
ZADD_UPDATED 元素被更新
ZADD_NOP 由于NX/XX 没有执行

返回值:
成功返回1，设置flags为ADDED/UPDATED
失败返回0，NAN

新增元素可能会引起zset从ziplist变成skiplist

通过 |= 的方式，改变flags
```
## 35. zsetDel
```
int zsetDel(robj *zobj, sds ele)

从zset中删除ele元素
ziplist: zzlDelete
skiplist: dictUnlink(zs->dict, ele)
不为空，score = dictGetVal(de)
注意要先从hash table删除再从skiplist删除，
因为从skiplist删除事实上释放了element对应的sds
这是被hash/skiplist共享的

dictFreeUnlinkedEntry(zs->dict, de)
zslDelete
->htNeedsResize dictResize(zs->dict)
```
## 36. zsetRank
```
long zsetRank(robj *zobj, sds ele, int reverse)

返回0-based rank或者-1(不存在)
reverse是false的场合 返回score由低到高
否则reverse是非0值，score从高到低

ziplist时，需要while zzlNext
skiplist 使用dictFind, dictGetVal
```
## 37. zaddGenericCommand
```
void zaddGenericCommand(client *c, int flags)
供ZADD和ZINCRBY使用的通用方法

查找key 如果不存在有序集合则新建 XX时什么也不做
server允许创建ziplista时优先创建ziplist

for循环 zsetAdd添加, 根据retflags计数
```
## 38. zaddCommand
```
void zaddCommand(client *c)

zaddGenericCommand(c, ZADD_NONE)
```
## 39. zincrbyCommand
```
void zincrbyCommand(client *c)

zaddGenericCommand(c, ZADD_INCR)
```
## 40. zremCommand
```
void zremCommand(client *c)

循环 zsetDel
```
## 41. zremrangeGenericCommand
```
void zremrangeGenericCommand(client *c, int rangetype)

#define ZRANGE_RANK 0
#define ZRANGE_SCORE 1
#define ZRANGE_LEX 2

1. 解析range参数
2. 检查可用性
3. 根据zobj->encoding和rangetype调用相应的delete方法
```
## 42. zramrangebyrankCommand
## 43. zremrangebyscoreCommand
## 44. zremrangebylexCommand

## 45. zsetopsrc
```
typedef struct {
    robj *subject;
    int type; // set/zset
    int encoding;
    double weight;

    union {
        // set iterators
        union _iterset {
            struct {
                intset *is;
                int ii;
            } is;
            struct {
                dict *dict;
                dictIterator *di;
                dictEntry *de;
            } ht;
        } set;

        // zset iterators
        union _iterzset {
            struct {
                unsigned char *zl;
                unsigned char *eptr, *sptr;
            } zl;
            struct {
                zset *zs;
                zskiplistNode *node;
            } sl;
        } zset;
    } iter;
} zsetopsrc;
```
## 46. zsetopval
```
#define OPVAL_DIRTY_SDS 1
#define OPVAL_DIRTY_LL 2
#define OPVAL_VALID_LL 4

typedef struct {
    int flags;
    unsigned char _buf[32];
    sds ele;
    unsigned char *estr;
    unsigned int elen;
    long long ell;
    double score;
} zsetopval;

flag在下次基于zsetopval的iteration执行前需要被清理
longlong value是个特例
```
## 47. zuiInitIterator
```
void zuiInitIterator(zsetopsrc *op)

初始化set的iterator
```
## 48. zuiClearIterator
```
void zuiClearIterator(zsetopsrc *op)

清空set的iterator
```
## 49. zuiLength
```
int zuiLength (zsetopsrc *op)

返回set的长度，根据op->type,op->encoding有所不同
```
## 50. zuiNext
```
int zuiNext(zsetopsrc *op, zsetopval *val)

检查当前的value是否合法，合法则保存在传来的值中，移动到下一个元素
否则认为移动到了结构体末尾
```
## 51. zuiLongLongFromValue
```
int zuiLongLongFromValue(zsetopval *val)

改变val->ele为longlong 
v->flags |= OPVAL_VALID_LL
返回val->flags & OPVAL_VALID_LL
```
## 52. zuiSdsFromValue
```
sds zuiSdsFromValue(zsetopval *val)

从value中创建sds, val->flags |= OPVAL_DIRTY_SDS
```
## 53. zuiNewSdsFeromValue
```
sds zuiNewSdsFromValue(zsetopval *val)

区别在于是新建一个sds，而不是把val->ele设为sds
如果val->ele已经存在，置空val->ele, 返回ele的值
```
## 54. zuiBufferFromValue
```
int zuiBufferFromValue(zsetopval *val)

当val->estr为空，设置val->elen, val->estr
val->ele不为空，则设置val->elen, val->estr基于val->ele
否则，依照val->_buf
固定return 1
```
## 55. zuiFind
```
int zuiFind(zsetopsrc *op, zsetopval *val, double *score)

在op结构中查找val->ele set找到后，*score = 1.0, return 1 否则return 0
zset zzlFind return 1(score在zzlFind内已设置)
否则 *score = *(double*)dictGetVal(de)
```
## 56. zuiCompareByCardinality
```
int zuiCompareByCardinality(const void *s1, const void *s2)

return zuiLength((zsetopsrc*)s1) - zuiLength((zsetopsrc*)s2);
```
## 57. zunionInterDictValue
```
#define REDIS_AGGR_SUM 1
#define REDIS_AGGR_MIN 2
#define REDIS_AGGR_MAX 3
#define zunionInterDictValue(_e) (dictGetVal(_e) == NULL ? 1.0 : *(double*)dictGetVal(_e))
``` 
## 58. zunionInterAggregate
```
inline static void zunionInterAggregate(double *target, double val, int aggregate)

根据aggregate的值，设置target为和/小/大
```
## 59. setAccumulatorDictType
```
dictType setAccumulatorDictType = {
    dictSdsHash,        hash function
    NULL,               key dup
    NULL,               val dup
    dictSdsKeyCompare,  key compare
    NULL,               key destructor
    NULL                val destructor
};
```
## 60. zunionInterGenericCommand
```
void zunionInterGenericCommand(client *c, robj *dstkey, int op)

zset union inter的共用方法
需要对所有集合先进行一次排序，
qsort(src, setnum, sizeof(zsetopsrc), zuiCompareByCardinality)
op == SET_OP_INTER时
当zuiLength(&src[0] > 0) 则后续的src也非空
while(zuiNext(&src[0], &zval))
for (j = 1; j< setnum; j++)
判断src[j].subject 与 src[0].subject
相等 设置value = zval.score * src[j].weight，与&score做比较
zuiFind(&src[j], &zval, &value)时
value *= src[j].weight 与&score做比较
否则，跳出循环
当j == setnum 即所有输入都包含 插score入zset

op == SET_OP_UNION
union至少和最大的集合一样大，dictExpand(accumulator, zuiLength(&src[setnum-1]))

首先，创建dict 遍历集合 elements->aggregated-scores
score = src[i].weight * zval.score
de = dictAddRaw(accumulator, zuiSdsFromValue(&zval), &existing)
如果不存在zval tmp = zuiNewSdsFromValue(&zval)
dictSetKey, dictSetDoubleVal

存在时，更新score

然后 将dict转换为最终的zset
dictGetkey, dictGetDoubleVal, zslInsert dictAdd
```
## 61. zunionstoreCommand
```
void zunionstoreCommand(client *c)
zunionInterGenericCommand(c, c->argv[1], SET_OP_UNION)
```
## 62. zinterstorCommand
```
void zinterstoreCommand(client *c)
zunionInterGenericCommand(c, c->argv[1], SET_OP_INTER);
```
## 63. zrangeGenericCommand
```
void zrangeGenericCommand(client *c, int reverse)

根据zobj->encoding和reverse，获取的方式不同 
```
## 64. zrangeCommand
```
void zrangeCommand(client *c)
zrangeGenericCommand(c, 0)
```
## 65. zrevrangeCommand
```
void zrevrangeCommand(client *c)
zrangeGenericCommand(c, 1)
```
## 66. genericZrangebyscoreCommand
```
void genericZrangebyscoreCommand(client *c, int reverse)

implements ZRANGEBYSCORE ZREVRANGEBYSCORE

遍历就行了，每次循环都需要检查，是否node仍在range内
```
## 67. zrangebyscoreCommand
```
void zrangebyscoreCommand(client *c)
genericZrangebyscoreCommand(c, 0)
```
## 68. zrevrangebyscoreCommand
```
void zrevrangebyscoreCommand(client *c)
genericZrangebyscoreCommand(c, 1)
```
## 69. zcountCommand
```
void zcountCommand(client *c)

统计在range范围内的ele数量 range为元素
ziplist时，需要zzlFirstInRange(zl, &range)
循环 zzlNext, zslValueLteMax终止

skiplist时，zn = zslFirstInRange(zsl, &range)
zslGetRank, zslLastInRange, zslGetRank，相减即可
```
## 70. zlexcountCommand
```
void zlexcountCommand(client *c)

与zcountCommand类似，只是范围采用相关的lex方法
```
## 71. genericZrangebylexCommand
```
void genericZrangebylexCommand(client *c, int reverse)

implements ZRANGEBYLEX ZREVRANGEBYLEX
```
## 72. zrangebylexCommand
```
void zrangebylexCommand(client *c)
```
## 73. zrevrangebylexCommand
```
void zrevrangebylexCommand(client *c)
```
## 74. zcardCommand
```
void zcardCommand(client *c)

zsetLength(zobj)
```
## 75. zscoreCommand
```
void zscoreCommand(client *c)

zsetScore(zobj, c->argv[2]->ptr, &score)
```
## 76. zrankGenericCommand
```
void zrankGenericCommand(client *c, int reverse)

zsetRank(zobj, ele->ptr, reverse)
```
## 77. zrankCommand
```
void zrankCommand(client *c)

zrankGenericCommand(c, 0)
```
## 78. zrevrankCommand
```
void zrevrankCommand(client *c)

zrankGenericCommand(c, 1)
```
## 79. zscanCommand
```
void zscanCommand(client *c)

scanGenericCommand(c, o, cursor)
```
