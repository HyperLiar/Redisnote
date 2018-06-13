# redisdb.md
## 1. redisDb
```
typedef struct redisDb {
    dict *dict;             keyspace for DB
    dict *expires;          Timeout of keys with a timeout set
    dict *blocking_keys;    Keys with clients waiting for data (BLPOP)
    dict *ready_keys;       Blocked keys that received a PUSH
    dict *watched_keys;     WATCHED keys for MULTI/EXEC CAS
    int id;                 Database ID
    long long avg_ttl;      Average TTL, just for stats
} redisDb

multiple databases identified by integers from 0 up to the max configured database
```
## 2. updateLFU
```
void updateLFU(robj *val)

updateLFU when an object is accessed
到时时减少计数，然后逻辑性增加计数，更新access time
```
## 3. lookupKey
```
robj *lookupKey(redisDb *db, robj *key, int flags)

低级的lookup key API 不会被命令直接调用
lookupKeyRead/lookupKeyWrite/lookupKeyReadWithFlags

de = dictFind(db->dict, key->ptr)
robj *val = dictGetVal(de)
更新val->lru, 保存子集时不做处理(因为会引发写入混乱)
```
## 4. lookupKeyReadWithFlags
```
robj *lookupKeyReadWithFlags(redisDb *db, robj *key, int flags)

读操作查找key, 在特定的db没有找到时返回空
side effect: 
1. 一个key到达ttl时会过期
2. 更新key的last access time
3. 全局keys的his/misses stats 会被更新

flags: LOOKUP_NONE/0 no special flags
       LOOKUP_NOTOUCH: 不改变last access time of the key

注意: 这个方法在key逻辑上已经过期实际上还存在时总会返会NULL，
即使key是在主库删除操作延迟时，从库的读操作也能正确返回

expireIfNeeded(db, key)
在lookupkey前，判断key是否过期，如果过期且在主库，直接返回NULL
如果是从库，且设置了cmd->flags CMD_READONLY 也返回空
因为在从库，expireIfNeeded不会真正的使一个key过期，只会返回key的逻辑状态

lookupKey,透传flag值
根据val是否找到增加misses hits，返回val
```
## 5. lookupKeyRead
```
robj *lookupKeyRead(redisDb *db, robj *key)

return lookupKeyReadWithFlags(db, key, LOOKUP_NONE)
```
## 6. lookupKeyWrite
```
robj *lookupKeyWrite(redisDb *db, robj *key)

如果有需要，expire the key if its TTL is reached
和read相比，只是先执行了expireIfNeeded(db, key)
```
## 7. lookupKeyReadOrReply
```
robj *lookupKeyReadOrReply(client *c, robj *key, robj *reply)

当lookupKeyRead返回空时， addReply(c, reply) 返回o
```
## 8. lookupKeyWriteOrReply
```
robj *lookupKeyWriteOrReply(client *c, robj *key, robj *reply)

类似lookupKeyReadOrReply
```
## 9. dbAdd
```
void dbAdd(redisDb *db, robj *key, robj *val)

增加key到db, 是否增加value的counter取决于调用者
key已经存在时会中断程序
retval = dictAdd(db->dict, copy, val)
```
## 10. dbOverwrite
```
void dbOverwrite(redisDb *db, robj *key, robj *val)

复写已存在的key
dictEntry *de = dictFind(db->dict, key->ptr)
dictReplace(db->dict, key->ptr, val)

设置LFU时，复写的情况下, 需要复制之前key val->lfu, 复制后还需要
updateLFU(val)
```
## 11. setKey
```
void setKey(redisDb *db, robj *key, robj *val)

高级set方法，使用为set a key to a new object.
1. ref count of the value object is incremented
2. clients watching for the destination key notified
3. the expire time of the key is reset 都会被设置为持久化
所有新的key都需要通过这个接口新建

lookupKeyWrite(db, key) == NULL时 dbAdd 否则 dbOverWrite
incrRefCount(val)
removeExpire(db, key)
signalModifiedKey(db, key)
```
## 12. dbExists
```
int dbExists(redisDb *db, robj *key)

return dictFind(db->dict, key->ptr) != NULL
```
## 13. dbRandomKey
```
robj *dbRandomKey(redisDb *db)

robj类型返回一个随机的key，没有key时返回NULL，只会返回还没有过期的key
while(1)循环，使用dictGetRandomKey(db->dict)获取de
key = dictGetKey(de)
dictFind(db->expires,key) 如果key有过期时间，需要检查
过期的场合需要continue继续循环

return keyObj keyObj = createStringObject(key, sdslen(key))
```
## 14. dbSyncDelete
```
int dbSyncDelete(redisDb *db, robj *key)

删除key value和相关的过期时间entry

如果db->expires size> 0 执行dictDelete(db->expires, key->ptr)
dictDeletre(db->dict, key->ptr) == DICT_OK 返回1
否则返回0
```
## 15. dbDelete
```
int dbDelete(redisDb *db, robj *key)

server.lazyfree_lazy_server_del ?
dbAsyncDelete(db, key)
: dbSyncDelete(db, key)
```
## 16. dbUnshareStringValue
```
robj *dbUnshareStringValue(redisDb *db, robj *key, robj *o)

当o->refcount > 1 或 o->encoding != OBJ_ENCODING_RAW时，
即 o 处在shared状态，新建一个unshared/not-encoded copy，
decrRefCount(decoded) dbOverwrite(db, key, o)

最终都会return o
```
## 17. emptyDb
```
long long emptyDb(int dbnum, int flags, void(callback) (void*))

清空一个redis-server中的所有db中的所有key
dbnum: -1 全部 特定数字
flags: EMPTYDB_NO_FLAGS 无
EMPTYDB_ASYNC 如果我们希望在不同线程释放内存

成功时 返回从db中删除的key的数量
否则返回-1（db number 超限) 设置errno 为EINVAL
-> dbnum < -1 || dbnum >= server.dbnum

for循环
if (async) emptyDbAsync(&server.db[j])
else dictEmpty(server.db[j].dict, callback)
dictEmpty(server.db[j].expires, callback)

removed += dictSize(server.db[j].dict)
```
## 18. selectDb
```
int selectDb(client *c, int id)

id < 0 || id >= server.dbnum return C_ERR
c->db = &server.db[id]; return C_OK
```
## 19. signalModifiedKey
```
void signalModifiedKey(redisDb *db, robj *key)

每次db中key被修改都会调用这个方法
touchWatchedKey(db, key)
```
## 20. signalFlushedDb
```
void signalFlushedDb(int dbid)
每次db flushed 调用这个方法
touchWatchedKeysOnFlush(dbid)
```
## 21. getFlushCommandFlags
```
int getFlushCommandFlags(client *c, int *flags)

返回flags的集合供FLUSHALL/FLUSHDB命令使用

c->argv > 1时，如果c->argc > 2或者 c->argv[1]->ptr不是async 返回C_ERR
设置*flags = EMPTYDB_ASYNC
否则设置*flags = EMPTYDB_NO_FLAGS

返回C_OK
```
## 22. flushdbCommand
```
void flushdbCommand(client *c)

FLUSHDB [ASYNC]
flushes the currently SELECTED redis DB

需要getFlushCommandFlags(c, &flags) 返回C_OK
signalFlushedDb(c->db->id)
server.dirty += emptyDb()
```
## 23. flushallCommand
```
void flushallCommand(client *c)

FLUSHALL [ASYNC]
flush the whole server data set

signalFlushedDb(-1)
server.dirty += emptyDb(-1, flags, NULL) 来清空db

rdb设置的场合 需要调用rdbRemoveTempFile
TODO: 了解rdb

server.dirty++
```
## 24. delGenericCommand
```
void delGenericCommand(client *c, int lazy)

implements DEL and LAZYDEL
循环c->argc 对每个argv expireIfNeeded
lazy的场合，使用dbAsyncDelete 否则dbSyncDelete
```
## 25. delCommand
```
void delCommand(client *c)
delGenericCommand(c, 0)
```
## 26. unlinkCommand
```
void unlinkCommand(client *c)
delGenericCommand(c, 1)
```
## 27. existsCommand
```
void existsCommand(client *c)

EXISTS key1 key2 ... key N
返回存在key的个数

for循环 依次expireIfNeeded(c->db, c->argv[j])
dbExists(c->db, c->argv[j]) count++
```
## 28. selectCommand
```
void selectCommand(client *c)

selectDb(c, id) c->db设置为相应id的db
```
## 29. randomkeyCommand
```
void randomkeyCommand(client *c)

key = dbRandomKey(c->db)
```
## 30. keysCommand
```
void keysCommand(client *c)

dictNext循环，dictGetKey, createStringObject创建keyobj
增加numkeys
这个命令会遍历当前的c->db->dict
```
## 31. scanCallback
```
void scanCallback(void *privdata, const dictEntry *de)

将dictionary iterator返回的元素收集到一个list中
void **pd = (void**) privdata
list *keys = pd[0]
robj *o = pd[1]
按o->type区分key的获取方式
listAddNodeTail(keys, key)
```
## 32. parseScanCursorOrReply
```
int parseScanCursorOrReply(client *c, robj *o, unsigned long *cursor)

尝试解析存储在o中的scan cursor
如果结果合法 则将其存储为cursor中的无符号整数 返回C_OK
否则 返回C_ERR 想客户端发送错误

*cursol = strtoul(o->ptr, &eptr, 10)
```
## 33. scanGenericCommand
```
void scanGenericCommand(client *c, robj *o, unsigned long cursor)

传o参数场合，必须是Hash/Set的object 否则o为空时 命令会在db关联的字典上操作
o非空时 方法假设client参数集的第一个参数是一个key, 故在解析前会被跳过

Hash object的场合，方法会返回Hash中每个元素的field and value

第一步，解析options
第二步，遍历元素，注意是有限制的
第三步，过滤元素 需要匹配给定的pattern,不能过期
decrRefCount(kobj) listDelNode(keys, node)
注意如果是zset/hash， 需要删除key对应的value
第四步，返回给客户端 遍历list addReplyBulk(c, kobj)
```
## 34. scanCommand
```
void scanCommand(client *c)

scan命令完全依赖于scanGenericCommand
parseScanCursorOrReply(c, c->argv[1], &cursor) 需要返回C_OK
scanGenericCommand(c, NULL, cursor)
```
## 35. dbsizeCommand
```
void dbsizeCommand(client *c)

addReplyLongLong(c, dictSize(c->db->dict))
```
## 36. lastsaveCommand
```
void lastsaveCommand(client *c)

addReplyLongLong(c, server.lastsave)
```
## 37. typeCommand
```
void typeCommand(client *c)

o = lookupKeyReadWithFlags(c->db, c->argv[1], LOOKUP_NOTOUCH)
addReplyStatus(c, type)
```
## 38. shutdownCommand
```
void shutdownCommand(client *c)

需要argc == 2, argv[1]有nosave时(不区分大小写) flags |= SHUTDOWN_NOSAVE
否则有save时 flags |= SHUTDOWN_SAVE
否则返回 shared.syyntaxerr

如果调用时服务器正在内存中加载一个数据集 需要保证shutdown过程中不去保存数据集
server.loading || server.sentinel_mode
flags = (flags & ~SHUTDOWN_SAVE) | SHUTDOWN_NOSAVE

调用 prepareForShutdown(flags) == C_OK
```
## 39. renameGenericCommand
```
void renameGenericCommand(client *c, int nx)

c->argv[1]->ptr c->argv[2]->ptr相同 什么也不做
创建一个同名的新key之前，删除旧的key
注意会设置相同的expire
```
## 40. renameCommand
```
void renameCommand(client *c)
renameGenericCommand(c, 0)
```
## 41. renamenxCommand
```
void renamenxCommand(client *c)
renameGenericCommand(c, 1)
```
## 42. moveCommand
```
void moveCommand(client *c)

server.cluster_enabled 设置时，不允许执行move操作
c->argv[2]作为dst的dbid
src与dst相同，return
lookupKeyWrite(c->db, c->argv[1]) 为空，return
lookupKeyWrite(dst, c->argv[1]) 不为空
return

都满足时 dbAdd(dst, c->argv[1], o)
expire不为-1时 setExpire(c, dst, c->argv[1], expire)
key移动成功后，free the entry in the source DB
dbDelete(src, c->argv[1])
```
## 43. scanDatabaseForReadyLists
```
void scanDatabaseForReadyLists(redisDb *db)

dbSwapDatabases的Helper function. scan所有有one or more个阻塞
的clients for B[LR]POP 或其他的list blocking 命令和信号的keys list.

key = dictGetKey(de) value = lookupKey(db, key, LOOKUP_NOTOUCH)
if (value && (value->type == OBJ_LIST || value->type == OBJ_STREAM))
signalKeyAsReady(db, key)
```
## 44. dbSwapDatabases
```
int dbSwapDatabases(int ids, int id2)

运行时交换两个数据库,注意client结构 c->db 指向给定的db
所以需要swap the underlying referenced 结构

如果至少有一个db ids超限返回C_ERR 否则返回C_OK

交换dict expires, avg_ttl，但不交换其他的dict，这是因为
希望客户端保持他们在原来的db中

然后处理blocked keys 使用scanDatabaseFirReadyLists

交换两个库，可能客户端等待的key列表已经不处在block状态了
然而出于效率的考虑，只有在dbAdd的时候才会做这个检查
故而我们需要rescan客户端阻塞的列表
```
## 45. swapdbCommand
```
void swapdbCommand(client *c)

检查参数合法，调用dbSwapDatabases
```
## 46. removeExpire
```
int removeExpire(redisDb *db, robj *key)

只有在main dict存在对应entry的expire才可以被move
否则这个key永远不会被free
dictFind(db->dict, key->ptr)
return dictDelete(db->expires, key->ptr) == DICT_OK
```
## 47. setExpire
```
void setExpire(client *c, redisDb *db, robj *key, long long when)

对特定的key设置expire when代表unix的绝对毫秒时间 when之后不可用

复用main dict的sds dictAddOrFind(db->expires,dictGetKey(kde))
dictSetSignedIntegerVal(de, when)设置过期时间
rememberSlaveKeyWithExpire(db, key)
```
## 48. getExpire
```
long long getExpire(redisDb *db, robj *key)

返回特殊key的过期时间 dictSize(db->expires) == 0 || dictFind*db->expires, key->ptr) == NULL 返回-1
还需要dictFind(db->dict, key->ptr) 保证key也在主dict内
dictGetSignedIntegerVal(de)
```
## 49. propagateExpire
```
void propagateExpire(redisDb *db, robj *key, int lazy)

把expire传播到slave和AOF文件内
当master内的key到期时，DEL操作会发送给所有的slave和AOF文件
这种方式key的过期时间是在一处中心化的

server.aof_state != AOF_OFF feedAppendOnlyFile(server.delCommand, db->id, argv,2)
replicationFeedSlaves(server.slaves, db->id, argv, 2)
```
## 50. expireIfNeeded
```
int expireIfNeeded(redisDb *db, robj *key)

主要被lookupKey族方法调用
方法的行为取决于实例的角色 slave角色不会过期key
他们只会等待著哭传递的del信息
但从库需要有一个连贯的返回 故而从库执行的read命令会表现的好像key已经过期
即使从库上还未过期，因为主库propagate DEL还没完成
在主库查找一个过期key的副作用是这样的key会从db中被驱逐
这个行为同样会传播给AOF/replication stream
key 可用返回0 过期返回1

过期时间when = getExpire(db, key)
server.loading时，不过期任何key，loading完成后再执行expire
如果处于lua调用，假设时间锁定在脚本开始执行的时间server.lua_time_start

slave，返回 now > when
否则now < when 返回0
删除过期key
```
## 51. getKeysUsingCommandTable
```
int *getKeysUsingCommandTable(struct redisCommand *cmd, robj **argv, int argc, int *numkeys)

从命令中获取key 
j = cmd->firstkey, j<= last, j += cmd->keystep
last = cmd->lastKey
last < 0 时 last = argc + last

返回keys, 个数存在*numkeys内
```
## 52. getKeysFromCommand
```
int *getKeysFromCommand(struct redisCommand *cmd, robj **argv, int argc, int *nuimkeys)

返回commnd中所有的keys
会返回array内key参数的位置
cmd->flags & CMD_MODULE_GETKEYS
moduleGetCommandKeysViaAPI(cmd, argv, argc, numkeys)
cmd->getkes_proc
cmd->getkeys_proc(cmd, argv, argc, numkeys)
否则
getKeysUsingCommandTable(cmd, argv, argc, numkeys)
```
## 53. getKeysFreeResult
```
void getKeysFreeResult(int *result)

释放getKeysFromCommand获取到的result
zfree(result)
```
## 54. zunionInterGetKeys
```
int *zunionInterGetKeys(struct redisCommand *cmd, robj **argv, int argc, int *numkeys)

extract keys from zunionstroe/zinterstore
```
## 55. evalGetKeys
```
int *evalGetKeys(struct redisCommand *cmd, robj **argv, int argc, int *numkeys)

extract keys from EVAL/EVALSHA
```
## 56. sortGetKeys
```
int *sortGetKeys(struct redisCommand *cmd, robj **argv, int argc, int *numkeys)

extract kesy from the sort command
```
## 57. migrateGetKeys
```
int *migrateGetKeys(struct redisCommand *cmd, robj **argv, int argc, int *numkeys)
```
## 58. georadiusGetKeys
```
int *georadiusGetKeys(struct redisCommand *cmd, robj **argv, int argc, int *numkeys)
```
## 59. xreadGetKeys
```
int *xreadGetKeys(struct redisCommand *cmd, robj **argv, int argc, int *numkeys)
```
## 60. slotToKeyUpdateKey
```
void slotToKeyUpdateKey(robj *key, int add)

used by Redis Cluster 为了快速找到属于特定的hash slot的keys
适用于rehash the cluster和其他我们需要了解是否有给定的hash slot内的key的场景

hashslot = keyHashSlot(key->ptr, sdslen(key->ptr))
add == 1 则 raxInsert/raxRemove
server.cluster->slots_to_keys, indexed, keylen+2
```
## 61. slotToKeyAdd
```
void slotToKeyAdd(robj *key)
slotToKeyUpdateKey(key, 1)
```
## 62. slotToKeyDel
```
void slotToKeyDel(robj *key)
slotToKeyUpdateKey(key, 0)
```
## 63. slotToKeyFlush
```
raxFree(server,cluster->slots_to_keys)
server.cluster->slots_to_keys = raxNew()
memset
```
## 64. getKeysInSlot
```
unsigned int getKeysInSlot(unsigned int hashslot, robj **keys, unsigned int count)

获取slot内的keys
调用方去减少key name的引用计数
rax 循环迭代，在keys中保存新建的stringObject
返回keys内的元素个数
```
## 65. delKeysInSlot
```
unsigned int delKeysInSlot(unsigned int hashslot)

删除特定的hash slot内的所有keys 返回被删除item个数
raxStart(&iter, server,cluster->slots_to_keys)
while (server.cluster->slots_keys_count[hashslot])
raxSeek, raxNext, *key = createStringObject,
dbDelete(&server.db[0], key)  decrRefCount(key)

raxStop(&iter)
```
## 66. countKeysInSlot
```
unsigned int countKeysInSlot(unsigned int hashslot)
return server.cluster->slot_keys_count[hashslot]
返回对应hashslot内的key总数
```
