# intset
## 1. intset
```
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```
## 2. 宏
```
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (siezof(int64_t))
```
## 3. _intsetValueEncoding
```
static uint8_t _intsetValueEncoding(int64_t v)

返回给定值需要的encoding
```
## 4. static int64_t _intsetGetEncoded
```
static int64_t _intsetGetEncoded(intset *is, int pos, uint8_t enc)

根据enc返回在pos位置的值-> (is->contents)开始 长度根据enc确定
```
## 5. _intsetGet
```
static int64_t _intsetGet(intset *is, int pos)

return _intsetGetEncoded(*is, pos, intrev32ifbe(is->encoding))
```
## 6. _insetSet
```
static void _intsetSet(intset *is, int pos, int64_t value)

在指定位置 按is->encoding存储value
```
## 7. intsetNew
```
intset *intsetNew(void)

创建新的intset 默认encoding为intrev32ifbe(INTSET_ENC_INT16)
```
## 8. intsetResize
```
static intset *intsetResize(intset *is, uint32_t len)

resize，大小为sizeof(intset) + len * intrev32ifbe(is->encoding)
```
## 9. intsetSearch
```
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos)

pos可为空
查找给的的value, 找到返回1，pos指向其位置
未找到返回0 pos指向它可以被插入的位置

intset中的数字是按顺序排列的，使用二分查找确定位置
```
## 10. intsetUpgradeAndAdd
```
static intset *intsetUpgradeAndAdd(intset *is, int64_t value)

将intset更新为更大的encoding 插入给定的value
value<0 设置在intset的开头 否则设置在intset的结尾
按length-- 从后向前在is中写入数据(可能是同一位置的内存，从前向后会覆盖)
```
## 11. intsetMoveTail
```
static void intsetMoveTail(intset *is, uint32_t from, uint32_t to)

移动from开始的intset尾部数据到to
```
## 12. intsetAdd
```
intset *intsetAdd(intset *is, int64_t value, uint8_t *success)

在intset中插入一个值
如果value的enc大于is->encoding，则需要对intset进行upgrade
如果search到了该值 返回
否则 resize 移动尾部以腾出空间
设置值并改变is->length
```
## 13. intsetRemove
```
intset *intsetRemove(intset *is, int64_t value, int *success)

从intset中删除值
删除直接使用movetail,再resize
```
## 14. intsetFind
```
uint8_t intsetFind(intset *is, int64_t value)

判断一个值是否在这个set中
value的enc在is->encoding范围内,则返回intsetSearch(is, value, NULL)
```
## 15. intsetRandom
```
int64_t intsetRandom(intset *is)

返回一个随机的数字(根据is->length)
```
## 16. intsetGet
```
uint8_t intsetGet(intset *is, uint32_t pos, int64_t *value)

不在intset中返回0,否则返回1,
*value = _intsetGet(is, pos);
```
## 17. intsetLen
```
uint32_t intsetLen(const intset *is)

返回intrev32ifbe(is->length)
```
## 18. intsetBlobLen
```
size_t intsetBlobLen(intset *is)

返回 sizeof(intset) + intrev32ifbe(is->length) * intrev32ifbe(is->encoding)
intset blob size in bytes
```
