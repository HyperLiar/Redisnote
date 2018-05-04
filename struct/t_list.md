# t_list
## 1. listTypePush
```
void listTypePush(robj *subject, robj *value, int where)

将元素推到subject list的头或尾,以where做标志
不需要调用incrRefCount,方法内会处理
encoding必须是OBJ_ENCODING_QUICKLIST
where为LIST_HEAD时加到表头,其他情况为表尾
quicklistPush(subject->ptr, value->ptr, len, pos)
decrRefCount(value)
```
## 2. listPopSaver
```
void *listPopSaver(unsigned char *data, unsigned int sz)

createStringObject((char*)data, sz)
```
## 3. listTypePop
```
robj *listTypePop(robj *subject, int where)

quicklistPopCustom(subject->ptr, ql_where, (unsigned char **)&value,
NULL, &vlong, listPopSaver)

!value时 value = createStringObjectFromLongLong(vlong)

返回value
```
## 4. listTypeLength
```
unsigned long listTypeLength(const robj *subject)

quicklistCount(subject->ptr)
```
## 5. listTypeInitIterator
```
listTypeIterator *listTypeInitIterator(robj *subject, long index,
unsigned char direction)

在特定的index初始化一个iterator
listTypeIterator *li = zmalloc(sizeof(listTypeIterator));
li->iter = quicklistGetIteratorAtIdx(li->subject->ptr,
iter_dicrction, index)
```
## 6. listTypeReleaseIterator
```
void listTypeReleaseIterator(listTypeIterator *li)

清空iterator
zfree(li->iter)
zfree(li)
```
## 7. listTypeNext
```
int listTypeNext(listTypeIterator *li, listTypeEntry *entry)
调用quicklistNext(li->iter, &entry->entry)
current entry是entry的话返回1，否则返回0
```
## 8. listTypeGet
```
robj *listTypeGet(listTypeEntry *entry)

返回当前iterator位置的entry或者NULL
```
## 9. listTypeInsert
```
void listTypeInsert(listTypeEntry *entry, robj *value, int where)
根据where quicklistInsertAfter 或者 quicklistInsertBefore
decrRefCount(value)
```
## 10. listTypeEqual
```
int listTypeEqual(listTypeEntry *entry, robj *o)

quicklistCompare(entry->entry.zi, o->ptr, sdslen(o->ptr))
比较给出的object和当前位置的entry
```
## 11. listTypeDelete
```
void listTypeDelete(listTypeIterator *iter, listTypeEntry *entry)

删除iter位置的元素
```
## 12. listTypeConvert
```
void listTypeConvert(robj *subject, int enc)

create a quicklist from a single ziplist
subject->ptr = quicklistCreateFromZiplist(zlen, depth, subject->ptr)
```
## 13. pushGenericCommand
```
void pushGenericCommand(client *c, int where)

获取lobj = lookupKeyWrite(c->db, c->argv[1])
循环，lobj为空时，新建lobj，执行dbadd(c->db,c->argv[1],lobj)操作
对2到c->argc, listTypePush(lobj, c->argv[j], where)
```
## 14. lpushCommand
```
void lpushCommand(client *c)

pushGenericCommand(c, LIST_HEAD)
```
## 15. rpushCommand
```
void rpushCommand(client *c)

pushGenericCommand(c, LIST_TAIL)
```
## 16. pushxGenericCommand
```
void pushxGenericCommand(client *c, int where)

与pushGenericCommand的区别在于,这个方法的obj
为subject = lookupKeyWriteOrReply(c, c->argv[1],shared.czero)
如果key不存在，则什么也不会做
```
## 17. lpushxCommand
```
void lpushxCommand(client *c)

pushxGenericCOmmand(c, LIST_HEAD)
```
## 18. rpushxCommand
```
void rpushCommand(client *c)

pushxGenericCommand(c, LIST_TAIL)
```
## 19. linsertCommand
```
void linsertCommand(client *c)

subject = lookupKeyWriteOrReply(c, c->argv[1], shared.zero)
使用listIter,寻找c->argv[3],找到的话listTypeInsert argv[4]
无论插入与否都会返回listTypeLength(object)
```
## 20. llenCommand
```
void llenCommand(client *c)

o = lookupKeyReadOrReply(c, c->argv[1], shared.czero)
返回listTypeLength(o)
```
## 21. lindexCommand
```
void lindexCommand(client *c)

o = lookupKeyReadOrReply(c, c->argv[1], shared.nullnulk)
根据quicklistIndex(o->ptr, index, &enrry)获取index处的entry
entry.value存在时 value=createStringObject((char*)entry.value, entry.sz)
否则,value = createStringObjectFromLongLong(entry.longval)

addReplyBulk(c, value)
decrRefCount(value)
```
## 22. lsetCommand
```
void lsetCommand(client *c)

o = lookupKeyWriteOrReply(c, c->argv[1], shared.nokeyerr)
ql = o->ptr
replaced = quicklistReplaceAtIndex(ql,index,value->ptr,sdslen(value->ptr))

根据replaced返回。
```
## 23. popGenericCommand
```
void popGenericCommand(client *c, int where)

o = lookupKeyWriteOrReply(c,c->argv[1],shared.nullbulk)
robj *value = listTypePop(o, where)
当value不为空,如果listTypeLength(o)==0,dbDelete(c->db, c->argv[1])
```
## 24. lpopCommand
```
void lpopCommand(client *c)

popGenericCommand(c, LIST_HEAD)
```
## 25. rpopCommand
```
void rpopCommand(client *c)

popGenericCommand(c, LIST_TAIL)
```
## 26. lrangeCommand
```
void lrangeCommand(client *c)

o = lookupKeyReadOrReply(c, c->argv[1], shared.emptymultibulk)
根据o的长度和start, end获得最终使用的start,end
iter = listTypeInitIterator(o, start, LIST_TAIL)
按rangelen--去迭代list

注意到iter的获取也是循环得到的
```
## 27. ltrimCommand
```
void ltrimCommand(client *c)

o = lookupKeyWriteOrReply(c, c->argv[1], shared.ok)
需要o的长度和start,end获取实际的下标
注意start>end||start>=llen时会清空list
quicklistDelRange(o->ptr, 0, ltrim)
quicklistDelRange(o->ptr, -rtrim, rtrim)
只保留ltrim-rtrim区间内的列表元素
```
## 28. lremCommand
```
void lremCommand(client *c)

lookupKeyWriteOrReply,根据c->argv[2]判断移除在头部还是尾部
iter循环整个列表,只要值相等就listTypeDelete
加入移除元素判断，可能不会遍历整个list
会返回被删除的元素个数
```
## 29. rpoplpushHandlePush
```
void rpoplpushHandlePush(client *c, robj *dstkey, robj *dstobj, robj *value)

在rpop的同时lpush元素到另一列表,在消息队列应用中避免消息的丢失.
还可以用来遍历列表, 使用相同的key作为rpoplpush的两个参数.

首先，如果key不存在，新建一个list
createQuikclistObject()
quikclistSetOptions()
dbAdd(c->db, dstkey, dstobj)

执行listTypePush(dstobj, value, LIST_HEAD)
```
## 30. rpoplpushCommand
```
void rpoplpushCommand(client *c)

sobj = lookupKeyWriteOrReply(c, c->argv[1], shared.nullbulk)
dobj = lookupKeyWrite(c->db, c->argv[2])
存在dobj 且type为OBJ_LIST，return --> 没懂
incrRefCount(touchedkey) 保存touchedkey(c->argv[1]) 因为rpoplpushHandlePush
可能改变命令参数
rpoplpushHandlePush(c,c->argv[2],dobj, value)
decrRefCount(value)
如果sobj列表为空,则删除它
decrRefCount(touchedKey)
```
## 31. serveClientBlockedOnList
```
int serveClientBlockedOnList(client *receiver, robj *key, robj *dstkey, redisDb *db,
robj *value, int where)

服务特定的在某个db的某个key阻塞的client
1. 为client提供value元素
2. 如果dstkey非空(brpoplpush) 也push value到dst list(lpush方)
3. 传播作为结果的brpop,blpop和额外的lpush,如果这三者任意进入了AOF和replication channel

where: LIST_TAIL 或者 LIST_HEAD 指示value元素是从头还是从尾部弹出
可以serve的场合返回C_OK，否则返回C_ERR告知client pop操作无法执行
只会在dst key由于type不正确无法被push的场合

dst key为空的时候
定义argv[0] = shared.lpop 或 shared.rpop, argv[1] = key
propagate(server.l/rpopCommand, db->id, argv, 2, PROPAGATE_AOF|PROPAGARE_REPL)
addReplyMultiBulkLen(receiver,2)
addReplyBulk(receiver,key) addReplyBulk(receiver,value)

dst key不为空
如果dstobj存在且type是OBJ_LIST
执行BRPOPLPUSH, argv[0] = shared.rpop, argv[1] = key
propagate(server,rpopCommand,db->id,argv,2,...)
相似的操作，propagate后执行rpoplpushHandlePush(receiver, dstkey, dstobj, value)

argv[0] = shared.lpuhs, argv[1] = dstkey, argv[2] = value
propagate(server.lpushCommand,db->id,argv,3,...)

dstobj不满足条件返回C_ERR

其他情况均返回C_OK
```
## 32. blockingPopGenericCommand
```
void blockingPopGenericCommand(client *c, int where)

循环j = 1; j < c->argc - 1
o = lookupKeyWrite(c->db, c->argv[j])
需要o非空, value为listTypePop(o, where)

rewriteClientCommandVector(c, 2, shared.lpop/rpop, c->argv[j])
return
如果multi/exec模式且列表为空,只能按timeout处理,不会阻塞

所有列表为空或key不存在,必须要block-->timeout为参数

也就是说,给出的多个列表发现某一个有元素时就会弹出并return
```
## 33. blpopCommand
```
void blpopCommand(client *c)

blockingPopGenericCommand(c, LIST_HEAD)
```
## 34. brpopCommand
```
void brpopCommand(client *c)

blockingPopGenericCommand(c, LIST_TAIL)
```
## 35. brpoplpushCommand
```
void brpoplpushCommand(client *c)

timeout 为UNIT_SECONDS
key = lookupKeyWrite(c->db, c->argv[1])

key为空,若c->flags & CLIENT_MULTI
addReply(c, shared.nullbulk)
否则 进行阻塞

key存在,rpoplpushCommand(c)
```
