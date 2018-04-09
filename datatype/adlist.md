# Adlist(双端链表)
## 1. listNode
```
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

定义listNode为双端链表节点
```
## 2. listIter
```
typedef struct listIter {
    listNode *next;
    int direction:
} listIter;
定义listIter为双向链表迭代器，用于记录迭代位置
```
## 3. list
```
typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
定义list为双端链表
```
## 4. listLength
```
#define listLength(l) ((l)->len)

获取list的长度
```
## 5. listFirst
```
#define listFirst(l) ((l)->head)

获取list的头节点
```
## 6. listLast
```
#define listLast(l) ((l)->tail)

获取list的尾节点
```
## 7. listPrevNode
```
#define listPrevNode(n) ((n)->prev)

获取节点n的前一个节点
```
## 8. listNextNode
```
#define listNextNode(n) ((n)->next)

获取节点n的后一个节点
```
## 9. listNodeValue
```
#define listNodeValue(n) ((n)->value)

获取节点n的value
```
## 10. listSetDupMethod
```
#define listSetDupMethod(l, m) ((l)->dup = (m))

设置链表的复制函数
```
## 11. listSetFreeMethod
```
#define listSetFreeMethod(l, m) ((l)->free = (m))

设置链表的释放空间函数
```
## 12. listSetMatchMethod
```
#define listSetMatchMethod(l, m) ((l)->match = (m))

设置链表的比较函数
```
## 13. listGetDupMethod
```
#define listGetDupMethod(l) ((l)->dup)

获取链表的复制函数
```
## 14. listGetFree
```
#define listGetFree(l) ((l)->free)

获取链表的释放空间函数
```
## 15. listGetMatchMethod
```
#define listGetMatchMethod(l) ((l)->match)

获取链表的比较函数
```
## 16. listCreate
```
list *listCreate(void)

创建新的链表,可以使用AlFreeList()释放链表，
但链表的节点的私有value需要用户使用AlFreeList后自行释放。
出错时返回NULL,否则为指向链表的指针
```
## 17. listEmpty
```
void listEmpty(list *list)
不摧毁链表结构，清除链表内所有元素
```
## 18. listRelease
```
void listRelease(list *list)

调用listEmpty，zfree,彻底释放链表占用空间
```
## 19. listAddNodeHead
```
list *listAddNodeHead(list *list, void *value)

在链表头部添加节点，失败不会改变链表
```
## 20. listAddNodeTail
```
list *listAddNodeTail(list *list, void *value)

在链表尾部添加节点，失败不会改变链表
```
## 21. listInsertNode
```
list *listInsertNode(list *list, listNode *old_node, void *value, int after)

after为非0，node插入old_node之后，否则在之前
先进行插入工作，然后检查node是否为头尾节点，
不是的话，调整与前/后节点的连接关系
```
## 22. listDelNode
```
void listDelNode(list *list, listNode *node)

调整node前后节点, zfree(node)，调整list的长度
```
## 23. listGetIterator
```
listIter *listGetIterator(list *list, int direction)

返回一个list的迭代器指针
每次调用listNext会得到下一个元素。
direction为AL_START_HEAD时，从头到尾遍历
AL_START_TAIL,从尾到头
```
## 24. listReleaseIterator
```
void listReleaseIterator(listIter *iter)

zfree(iter)
```
## 25. listRewind
```
void listRewind(list *list, listIter *li)

将li设置为 list的头节点开始遍历
```
## 26. listRewindTail
void listRewindTail(list *list, listIter *li)

将li设置为 list的尾节点开始遍历
```
## 27. listNext
```
listNode *listNext(listIter *iter)

获取迭代得到的下一个节点
```
## 28. listDup
```
list *listDup(list *orig)

复制整个链表，内存超限返回NULL
iter迭代，listSetDupMethod复制，
失败调用listRelease,返回NULL
```
## 29. listSearchKey
```
listNode *listSearchKey(list *list, void *key)

Search the list for a node matching a given key.
设定了match方法使用match方法进行匹配，否则'=='
返回第一个匹配成功的节点，失败返回NULL
```
## 30. listIndex
```
listNode *listIndex(list *list, long index)

返回从0开始的下标计数的node
index大于0从头节点开始，小于0尾节点
```
## 31. listRotate
```
void listRotate(list *list)

removing the tail node and inserting it to the head
```
## 32. listJoin
```
void listJoin(list *l, list *o)

add all the elements of the list 'o' at the end of
the list 'l'.
'o' will be set empty but otherwise valid.
```
