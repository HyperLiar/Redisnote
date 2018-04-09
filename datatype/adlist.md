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
