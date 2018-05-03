# t_string redis中的字符串
## 1. checkStringLength
```
static int checkStringLength(client *c, long long size)

size不能大于521*1024*1024
```
## 2. setGenericCommand
```
#define OBJ_SET_NO_FLAGS 0
#define OBJ_SET_NX (1<<0) set if key not exists
#define OBJ_SET_XX (1<<1) set if key exists
#define OBJ_SET_EX (1<<2) set if tim ein seconds is given
#define OBJ_SET_PX (1<<3) set if time in ms is given
void setGenericCommand(client *c, int flags, robj *key, robj *val, robj *expire
    int unit, robj *ok_reply, robj *abort_reply)

根据不同的选项进行调用
flags 改变command的行为
expire表示设置在redis传给用户的object的超时时间
'ok_reply' 'abort_reply' 表示函数给客户端的返回
ok_reply: NULL时使用"+OK"
abort_reply: NULL时使用"$-1"

会执行setkey操作,然后检查是否有expire,
有会执行setExpire(c,c->db,key,mstime()+millseconds)

最终addReply
```
## 3. setCommand
```
void setCommand(client *c)

根据c->argc->ptr的格式, flags做|=运算
对c->argv[2]做tryObjectEncoding(c->argv[2])
expire为新定义robj,unit为UNIT_SECONDS或UNIT_MILLISECONDS
setGenericCommand(c, flags, c->argv[1], c->argv[2], expire, unit, NULL, NULL)
```
## 4. setnxCommand
```
void setnxCommand(client *c)

flag:OBJ_SET_NX, expire: NULL, unit: 0, ok:shared,cone, abort: shared.czero
```
## 5. setexCommand
```
void setexCommand(client *c)

flag:OBJ_SET_NO_FLAGS, expire:c->argv[2] unit:UNIT_SECONDS, NULL, NULL
```
## 6. pserexCommand
```
void psetexCommand(client *c)

flag:OBJ_SET_NO_FLAGS, expire:c->argv[2] unit:UNIT_MILLISECONDS, NULL, NULL
```
## 7. getGenericCommand
```
int getGenericCommand(client *c)

o = lookupKeyReadOrReplay(c, c->argv[1], shared.nullnulk)
为空时返回C_OK
如果 o->type != OBJ_STRING, C_ERR
否则C_OK
```
## 8. getCommand
```
void getCommand(client *c)

调用getGenericCommand(c)
```
## 9. getsetCommand
```
void getsetCommand(client *c)

getGenericCommand(c)为C_ERR 则return
tryObjectEncoding(c->argv[2])
setKey(c->db, c->argv[1], c->argv[2])
notifyKeyspaceEvent(NOTIFY_STRING, "set", c->argv[1], c->db->id)
server.dirty++

基本就是把代码抄了一遍。。
```
## 10. setrangeCommand
```
void setrangeCommand(client *c)

getLongFromObjectOrReplay(c, c->argv[2], &offset, NULL)
offset < 0 会报超限错误

o = lookupKeyWrite(c->db, c->argv[1])
o为空, 如果sdslen(value) = 0 返回shared.czeroe
如果offset+sdslen(value)不通过checkStringLength,直接return
通过检查，o = createObject(OBJ_STRING, sdsnewlen(NULL, offset+sdslen(value)))
dbAdd(c->db, c->argv[1], o)

o不为空,说明key存在
checkType(c, o, OBJ_STRING)

当value为空时返回stringObjectLen(o)
offset+sdslen(value)不通过checkStringLength, return
o = dbUnshareStringValue(c->db, c->argv[1], o)
当object shared or encoded，创建一个copy

如果sdslen(value) > 0
o->ptr = sdsgrowzero(o->ptr, offset+sdslen(value)) 0填充sds
```
## 11. getrangeCommand
```
void getrangeCommand(client *c)

argv[2]为start, argv[3]为end，首先保证getLongLongFromObjectOrReply
为C_Ok
判断o = lookupKeyReadOrReply(c, c->argv[1], shared.emptybulk)
且o->type == OBJ_STRING

区分o->encoding是INT, str = llbuf, 存储ll2string(o->ptr)
否则str = o->ptr, strlen = sdslen(str)

接着对start, end做合法范围处理

addReplyBulkCBuffer(c, (char*)str+start, end-start+1)
```
## 12. mgetCommand
```
void mgetCommand(client *c)

addReplyMultiBulkLen(c, c->argc-1)
for(j=1; j < c->argc; j++)循环,
o = lookupKeyRead(c->db, c->argv[j])
o为空或type不对,addReply(c, shared.nullbulk);
否则addReplyBulk(c, o)
```
## 13. msetGenericCommand
```
void msetGenericCommand(client *c, int nx)

如果是nx场合,只要有一个key存在就不会执行key设置
lookupKeyWrite(c->db, c->argv[j])

for循环, tryObjectEncoding(value), setkey(c->db, c->argv[j], c->argv[j+1])
server.dirty += (c->argc-1)/2
```
## 14. msetCommand
```
void msetCommand(client *c)

msetGenericCommand(c, 0)
```
## 15. msetnxCommand
```
void msetnxCommand(c, 1)
```
## 16. incrDecrCommand
```
void incrDecrCommand(client *c, long long incr)

o = lookupKeyWrite(c->db, c->argv[1])
o不等于空且o->type不是OBJ_STRING会返回
getLongLongFromObjectOrReply(c, o, &value, NULL)
判断incr的值与value计算后是否会overflow,
value += incr;

检查o->refcount, o->encoding和value是否在long范围内
符合条件,robj *new = o, o->ptr = (void*)((long)value)
--> 即value是否可表示为long

否则 new = createStringObjectFromLongLong(value)
o为空(key不存在),dbAdd(c->db, c->argv[1], new)
o不为空, dbOverwrite(c->db, c->argv[1], new)

server.dirty++
addReply(c, shared.colon) addReply(c,new) addReply(c, shared.crlf)
```
## 17. incrCommand
```
void incrCommand(client *c)

incrDecrCommand(c, 1)
```
## 18. decrCommand
```
void decrCommand(client *c)

incrDecrCommand(c, -1)
```
## 19. incrbyCommand
```
void incrbyCommand(client *c)

getLongLongFromObjectOrReply(c, c->argv[2], &incr, NULL)
获取参数中的incr

incrDecrCommand(c, incr)
```
## 20. decrbyCommand
```
void decrbyCommand(client *c)

同incrbyCommand
```
## 21. incrbyfloatCommand
```
void incrbyfloatCommand(client *c)

需要!isnan(value) && !isinf(value)
与incrDecrCommand基本相同,
new = createStringObjectFromLongDouble(value, 1)

最后会replicate INCRBYFLOAT as SET,这是为了保证float的精度和格式
在复制中不会出现差异。

aux = createStringObject("SET",3)
rewriteClientCommandArgument(c,0,aux)
decrRefCount(aux)
rewriteClientCommandArgument(c,2,new)
```
## 22. appendCommand
```
void appendCommand(client *c)

o = lookupKeyWrite(c->db, c->argv[1])为空,
新建key
dbAdd(c->db, c->argv[1], c->argv[2])
incrRefCount(c->argv[2])

o不为空, 检查type, append=c->argv[2],检查stringObjectLen(o)+sdslen(append->ptr)
长度，即append后的长度

dbUnshareStringValue(c->db, c->argv[1], o)
o->ptr = sdscatlen(o->ptr, append->ptr, sdslen(append->ptr)
增加了o->ptr的长度

server.dirty++
addReplyLongLong(c, totlen)
```
## 23. strlenCommand
```
void strlenCommand(client *c)

o=lookupKeyReadOrReply(c, c->argv[1], shared.czero)
checkType(c,o,OBJ_STRING)

addReplyLongLong(c, stringObjectLen(o))
```
