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


```
