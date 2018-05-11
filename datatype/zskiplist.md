# zskiplist 
## 1. zskiplistNode
```
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forawrd;
        unsigned int span;
    } level[];
} zskiplistNode;
```
## 2. zskiplist
```
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```
## 3. zset
```
typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```
## 4. zrangespec
```
typedef struct {
    double min, max;
    int minex, maxex;
} zrangespec;
```
## 5. zlexrangespec
```
typedef struct {
    sds min, max;
    int minex, maxex;
} zlexrangespec;
lexicographic: 词典
```
## 6. zslCreateNode
```
zskiplistNode *zslCreateNode(int level, double score, sds ele)

创建一个有特定level的zsl节点
zskiplistNode *zn, sizeof(*zn)+level*sizeof(struct zskiplistLevel)
zn->score = score
zn->ele = ele
```
## 7. zslCreate
```
zskiplist *zslCreate(void)

创建新的ziplist *zsl
zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL)
循环 zsl->header->level[j].forward = NULL
     zsl->header->level[j].span = 0

zsl->header->backward = NULL
zsl->tail = NULL
```
## 8. zslFreeNode
```
void zslFreeNode(zskiplistNode *node)

free特定ziplist的node
sdsfree(node->ele)
zfree(node)
```
## 9. zslFree
```
void zslFree(zskiplist *zsl)

free a whole skiplist, while循环
next = node->level[0].forward, zslFreeNode(node), node = next

最后zfree(zsl)
```
## 10. zslRandomLevel
```
int zslRandomLevel(void)

返回一个准备要创建的skiplist的随机level,
在1到ZSKIPLIST_MAXLEVEL间, 更高的level更不可能出现

while((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
level += 1;
```
## 11. zslInsert
```
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele)

向skiplist加入元素
i = zsl->level-1 i>=0 i--循环
rank[i] = i == (zsl->level-1) ? 0 : rank[i+1]
rank[i] += x->level[i].span
rank 为每层到头部的距离,update为每层要插入的位置

假设元素还没有在skiplist内,而score是允许重复的,重新插入相同的元素
在zslInsert过程中不会发生，因为zslInsert调用者会测试它是否已经在skiplist内

随机一个level, 若不在update内 需要对rand[i] update[i]赋值
zslCreateNode(level, score, ele)
如果level大于当前的level, 需要循环设置rank update的值
新一层都视为在header指针下,故span都为zsl->length

循环0-level, 更新update的span和forward
根据update的span值更新x->level[i].span

level < zsl->level时,循环增加level以后的update[i]->level[i].span(因为距离增加了)

返回x的地址
```
## 12. zslDeleteNode
```
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update)

删除节点, update为要更新的节点数组
循环i = 0, i<zsl->level, i++
当update[i]->level[i].forward == x时,
span += x->level[i].span - 1
forward = x->level[i].forward

即指向x的下一个节点,更新到下一个节点的节点数(两段距离相加)
判断update[i]->level[i].forward和x是否相等

不是x的上一个节点时,只需要span--

x不是最后一个节点时 x->level[0].forward->backward = x->backward
否则, zsl->tail = x->backward即可

while(zsl->level>1 && zsl->header->level[zsl->level-1].forward == NULL)
zsl->level --

当某一层为空时,减少zsl的总层数
zsl->length--
最后减少zsl的长度
```
## 13. zslDelete
```
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node)

按匹配的score和ele删除元素, 删除成功返回1, 其他情况返回0

对层数循环, 内部while查找符合的节点, 记录x->level[i].forward在update
update[0]就是最终要删除的节点的前一个节点

因为可能有许多score相同的元素,所以需要找到符合条件的x(score, x->ele)
x = x->level[0].forward,比较x的score和ele
调用zslDeleteNode(zsl,x,update)

否则的话 return 0
```
## 14. zslValueGteMin
```
int zslValueGteMin(double value, zrangespec *spec)

return spec->minex?(value>spec->min):（value>=spec->min)
```
## 15. zslValueLteMax
```
int zslValueLteMax(double value, zrangespec *spec)

return spec->maxex?(value<spec->max):(value<=spec->max)
```
## 16. zslIsInRange
```
int zslIsInRange(zskiplist *zsl, zrangespec *range)

score in range
比较x = zsl->tail
x == NULL || !zslValueGteMin(x->score, range) return 0
x = zsl->header->level[0].forward
x == NULL || !zslValueLteMax(x->score, range) return 0

return 1
```
## 17. zslFirstInRange
```
zskiplistNode *zslFirstInRange(zskiplist *zsl, zrangespec *range)

找到range内的第一个node, 如果range内不包含任何元素返回NULL

zslIsInRange(zsl, range)判断, 层数循环,while判断,out of range时
后移, 找到层数最小的!zslValueGteMin(x->level[i].forward->score,range)的x.
x = x->level[i].forward, x = x->level[0].forward
如果!zslValueLteMax(x->score, range)返回NULL
否则返回x
```
## 18. zslLastInRange
```
zskiplistNode *zslLastInRange(zskiplist *zsl, zrangespec *range)

找到range内的最后一个node
类似, 条件变为在范围内zslValueLteMax(x->level[i].forward->score,range)
```
## 19. zslDeleteRangeByScore
```
unsigned long zslDeleteRangeByScore(zskiplist *zsl, zrangespec *range, dict *dict)

删除范围内的所有节点
层数 while找到最后一个score <或<= min的x
x = x->level[0].forward
删除范围内的nodes
while, x = x->level[0].forward, 使用zslDeleteNode,dictDelete(dict, x->ele)
zslFreeNode
返回被删除的node个数
```
## 20. zslDeleteRangeByLex
```
unsigned long zslDeleteRangeByLex(zskiplist *zsl, zlexrangespec *range, dict *dict)

逻辑类似,只是范围改为字典范围
```
## 21. zslDeleteRangeByRank
```
unsigned long zslDeleteRangeByRank(zskiplist *zsl, unsigned int start
unsigned int end, dict *dict)

删除rank在start,end间的所有元素, rank从1开始
老套路找到start的位置,通过traversed字段记录span值(注意层数是从上往下)
while循环, zslDeleteNode, dictDelete, zslFreeNode连击
```
## 22. zslGetRank
```
unsigned long zslGetRank(zskiplist *zsl, double score, sds ele)

按score和ele查找元素,返回rank,rank从1开始
老套路, while内移动指针,记录rank,
注意x可能是zsl->header,所以需要检查obj是否非空
没有找到对应元素的场合,返回0
```
## 23. zslGetElementByRank
```
zskiplistNode* zslGetElementByRank(zskiplist *zsl, unsigned long rank)

按rank找到一个node, rank是1-based
```
## 24. zslParseRange
```
static int zslParseRange(robj *min, robj *max, zrangespec *spec)

populate the rangespec according to the objects min and max
如果min/max为'('的形式,表示< / >，否则<= / >=
```
## 25. zslParseLexRangeItem
```
int zslParseLexRangeItem(robj *item, sds *dest, int *ex)

(foo 为 foo 开区间, [foo 为 foo 闭区间,
- 为可能的最小串, + 为可能的最大串
```
## 26. zslFreeLexRange
```
void zslFreeLexRange(zlexrangespec *spec)

free spec
```
## 27. zslParseLexRange
```
int zslParseLexRange(robj *min, robj *max, zlexrangespec *spec)

注意range必须以( [开头,故这个range不适用于int encoded object
```
## 28. sdscmplex
```
int sdscmplex(sds a, sds b)

对sdscmp封装, 比较a,b和-inf, +inf是否相等
a == b 返回0
a == -inf b == +inf 返回-1
a == +inf b == -inf 返回 1
其他情况返回sdscmp
```
## 29. zslLexValueGteMin
```
int zslLexValueGteMin(sds value, zlexrangespec *spec)

spec->minex存在 返回 sdscmplex(value, spec->min) > 0
否则返回 sdscmplex(value, spec->min) >= 0
```
## 30. zslLexValueLteMax
```
int zslLexValueLteMax(sds value, zlexrangespec *spec)

类似
```
## 31. zslIsInLexRange
```
int zslIsInLexRange(zskiplist *zsl, zlexrangespec *range)

判断zset内是否有部分在lex range范围内
```
## 32. zslFirstInLexRange
```
zskiplistNode *zslFirstInLexRange(zskiplist *zsl, zlexrangespec *range)

找到特定lex range内的第一个node
老套路, zslLexValueGteMin, 然后判断x->level[0].forward是否为空
检查zslLexValueLteMax 确定在范围内
```
## 33. zslLastInLexRange
```
zskiplistNode *zslLastInLexRange(zskiplist *zsl, zlexrangespec *range)
```
