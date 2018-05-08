# t_set
## 1. setTypeAdd
```
int setTypeAdd(robj *subject, sds value)

向set中添加特定value
如果已经在set内, return 0
否则添加, return 1

encoding为OBJ_ENCODING_HT, 
dictEntry *de = dictAddRaw(subject->ptr, value, NULL)
dictSetKey(ht, de, sdsdup(value))
dictSetVal(ht, de, NULL)

OBJ_ENCODING_INISET
如果value可以被表示为sds
subject->ptr = intsetAdd(subject->ptr, llval, &success)
否则
setTypeConvert(subject, OBJ_ENCODING_HT)
dictAdd(subject->ptr, sdsdup(value), NULL)
```
## 2. setTypeRemove
```
int setTypeRemove(robj *setobj, sds value)

查找value是否在set内
如果是HT, dictFind
否则value可被表示为longlong时
intsetFind()

其他情况返回0
```
## 3. setTypeInitIterator
```
setTypeIterator *setTypeInitIterator(robj *subject)

生成set的Iterator, HT设置si->di为dictGetIterator
否则设置si->ii = 0
```
## 4. setTypeReleaseIterator
```
void setTypeReleaseIterator(setTypeIterator *si)

是HT的场合, dictReleaseIterator(si->di)

zfree(si)
```
## 5. setTypeNext
```
int setTypeNext(setTypeIterator *si, sds *sdsele, int64_t *llele)

移动到下一个entry, 返回si->encoding 或-1(next为空)
HT: *sdsele = dictGetKey(de),llele没有用到
INTSET: intsetGet(si->subject->ptr, si->ii++, llele)
sdsele 设置为NULL
```
## 6. setTypeNextObject
```
sds setTypeNextObject(setTypeIterator *si)

sdsTypeNext后，按encoding创建sds.
```
## 7. setTypeRandomElement
```
int setTypeRandomElement(robj *setobj, sds *sdsele, int64_t *llele)

从非空的set中返回随机元素
dictGetRandomKey, intsetRandom
return setobj->encoding
```
## 8. setTypeSize
```
unsigned long setTypeSize(const robj *subject)

返回set的大小
dictSize, intsetLen
```
## 9. setTypeConvert
```
void setTypeConvert(robj *setobj, int enc)

需要预先分配dict空间 dictExpand(d, intsetlen(setobj->ptr))
通过setTypeIninIterator, 遍历set, dictAdd
zfree(setobj->ptr); setobj->ptr = d;
```
## 10. saddCommand
```
void saddCommand(client *c)

批量向set中增加元素, 循环
set为空,setTypeCreate, dbAdd(c->db, c-argv[1], set)

对j = 2; j < c->argc,循环setTypeAdd(set, c->argv[j]->ptr)
```
## 11. sremCommand
```
void sremCommand(client *c)

批量删除,
setTypeRemove(set, c->argv[j]->ptr)
清空的场合dbDelete
```
## 12. smoveCommand
```
void smoveCommand(client *c)

先从src中删除，再在dst中添加
setTypeRemove(srcset, ele->ptr)
setTypesize(srcset) == 0, dbDelete
setTypeAdd(dstset, ele->ptr)
```
## 13. sismemberCommand
```
void sismemberCommand(client *c)

setTypeIsMember(set, c->argv[2]->ptr)
```
## 14. scardCommand
```
void scardCommand(client *c)

addReplyLongLong(c, setTypeSize(o))
```
## 15. spopWithCountCommand
```
#define SPOP_MOVE_STRATEGY_MUL 5
void spopWithCountCommand(client *c)

count: 要pop的元素个数 size: set内的总数 remaining: size - count
1. count >= size, 返回整个set
2 3. 分解SPOP为多个SREM
2. remaining * SPOP_MOVE_STRATEGY_MUL > count, 随机的remove元素,作为SREM操作处理
3. 其他情况, 请求个数非常大接近于set本身的大小, 创建新的set，
保存不会return的值
遍历remaining 随机取出元素保存在newset,并setTypeRemove(set, sdsele)
dbOverwrite(c->db, c->argv[1], newset)

遍历旧set, 将旧set的内容返回给客户端,释放旧set
```
## 16. spopCommand
```
void spopCommand(client *c)

当参数个数 == 3, 调用spopWithCountCommand
否则随机选取一个元素, Remove
```
## 17. srandmemberWithCountCommand
```
#define SRANDMEMBER_SUB_STRATEZAGY_MUL 3
void srandmemberWithCountCommand(client *c)

1.count<0, 循环count返回随机元素,仅取不rem
2.count >= size, 返回整个set
sunionDiffGenericCommand(c,c->argv+1,1,NULL,SET_OP_UNION)
3.count*SRANDMEMBER_SUB_STRATEGY_MUL > size
创建一个dict, 遍历set将其添加到dict中,然后将dict删除
随机元素直到size == count
4.循环count次 dictAdd(d, objele, NULL)

3 4需要返回dict给客户端.
```
## 18. srandmemberCommand
```
void srandmemberCommand(client *c)

如果c->argv == 3, 即命令附带个数 srandmemberWithCountCommand(c)
否则使用setTypeRandomElement获取随机元素返回
```
## 19. qsortCompareSetsByCardinality
```
int qsortCompareSetsByCardinality(const void *s1, const void *s2)

判断s1和s2的size大小比，1大1， 2大-1，相等0
```
## 20. qsortCompareSetsByRevCardinality
```
int qsortCompareSetsByRevCardinality(const void *s1, const void *s2)

这个方法会把空set的size作为0来比较
```
## 21. sinterGenericCommand
```
void sinterGenericCommand(client *c, robj **setkeys, unsigned long setnum,
robj *dstkey)

求一批set的交集
首先通过qsortCompareSetsByRevCardinality对集合做快排,
循环全部集合, 如果最小集合中有一个元素在其他集合中不存在,则舍弃它
如果dstkey不为空,将交集存到dstkey
```
## 22. sinterCommand
```
void sinterCommand(client *c)

sinterGenericCommand(c,c->argv+1,c->argc-1,NULL)
```
## 23. sinterstorCommand
```
void sinterstorCommand(client *c)

sinterGenericCommand(c,c->argv+2,c->argc-2,c->argv[1])
```
## 24. sunionDiffGenericCommand
```
#define SET_OP_UNION 0
#define SET_OP_DIFF 1
#define SET_OP_INTER 2
void sunionDIffGenericCommand(client *c, robj **setkeys, int setnum,
robj *dstkey, int op)

SET_OP_DIFF
O(N*M) N是第一个set的元素size, M是set的总数
O(N) N是所有集合内的所有元素总数

第一种算法的常数时间更少,计算时/=2,
采用第一种算法时,使用qsort(CompareSetsByRevCardinality)对第二个以后
的set做快排

SET_OP_UNION时,单纯将所有元素加入临时set
SET_OP_DIFF 算法1, 循环第一个集合内的元素,如果它不存在
于其他所有集合内,将它加入结果集
算法2, 循环所有其他集合元素,执行setTypeRemove

如果dstset不为空,dbAdd
```
## 25. sunionCommand
```
void sunionCommand(client *c)

sunionDiffGenericCommand(c,c->argv+1,c->argv-1,NULL,SET_OP_UNION)
```
## 26. sunionstoreCommand
```
void suionstoreCommand(client *c)

sunionDiffgenericCommand(c,c->argv+2,c->argc-2,c->argv[1],SET_OP_UNION)
```
## 27. sdiffCommand
```
void sdiffCommand(client *c)

sunionDiffGenericCommand(c,c->argv+1,c->argc-1,NULL,SET_OP_DIFF)//
```
## 28. sdiffstoreCommand
```
void sdiffstoreCommand(client *c)

sunionDiffGenericCommand(c, c->argv+2, c->argv-2, c->argv[1], SET_OP_DIFF)
```
## 29. sscanCommand
```
void sscanCommand(client *c)

scanGenericCommand(c,set,cursor)
```
