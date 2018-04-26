# zipmap
## 1. 概述
```
zipmap是 'string' => 'string'格式的hash表
例如"foo" => "bar", "hello" => "world"会存储为
<zmlen><len>"foo"<len><free>"bar"<len>"hello"<len><free>"world"

<zmlen>为1byte，包含zipmap的当前大小
当zipmap的长度大于254时，这个值不再使用
zipmap的长度需要遍历去确定

<len>表示接下的string的长度(key或者value)
<len>长度为encode在single value或者5 bytes value
如果第一个byte的值在0-253，那么是signle-byte长度
254，那么接下来会跟4bytes的无符号整型(host byte order)
255用来标记hash的结尾

<free>为string后未使用的byte数量, 产生自修改给定key对应的value
总是一个unsigned 8 bit number，因为如果一个更新操作需要变更更多
字节，zipmap会进行reallocate确保它尽可能的小

上面的hash最小化压缩为
"\x02\x03foo\x00bar\x05hello\x05\x00world\xff"

查找的复杂度为O(n),n表示zipmap中的元素个数而不是byte数
这会降低查找的成本
```
## 2. 宏
```
ZIPMAP_BIGLEN 254
ZIPMAP_END 255

ZIPMAP_VALUE_MAX_FREE 4
定义了free的最大值
the max number of trailing bytes in a value

ZIPMAP_LEN_BYTES(_l) (((_l) < ZIPMAP_BIGLEN) ? 1 : sizeof(unsigned int)+1)
返回interger value l 对应长度encode所需的bytes
```
## 3. zipmapNew
```
unsigned char *zipmapNew(void)

创建一个空的zipmap
两字节，zm[0] = 0;(长度)
        zm[1] = ZIPMAP_END;
```
## 4. zipmapDecodeLength
```
static unsigned int zipmapDecodeLength(unsigned char *p)
decode the encoded length pointer by 'p'

使用memrev32ifbe(&len)的方式转换
```
## 5. zipmapEncodeLength
```
static unsigned int zipmapEncodeLength(unsigned char *p, unsigned int len)

encode length l and writing it in p 如果p为空只返回encode l所需的byte数量
len小于254,写在一个字节即可
否则需要memcpy
```
## 6. zipmapLookupRaw
```
statis unsigned char *zipmapLookupRaw(unsigned char *zm, unsigned char *key, unsigned int klen,
unsgined int *totlen)

查找匹配的key 返回指向zipmap内的指针 未找到对应key返回NULL
返回NULL时，totlen会设置为整个zipmap的size 保证调用函数可以
reallocate空间去设置更多的entry

循环，l = zipmapDecodeLength(p)
llen为l的实际长度，根据l和llen去判断是否相等
不等p += llen + l（跳过key的长度）
然后需要对p在执行zipmapDecode得到l
p += zipmapEncodeLength(NULL, l);
free = p[0]
p += l + 1 + free;

+1是为了跳过free本身占用的byte

传进来的totlen不为空时， totlen = (unsigned int)(p - zm) + 1;
```
## 7. zipmapRequiredLength
```
static unsigned long zipmapRequiredLength(unsigned int klen, unsigned int vlen)

返回k v 需要的总长度
注意需要k长度 v长度 vfree三个额外字节
计算长度所需字节可能需要增加
```
## 8. zipmapRawKeyLength
```
static unsigned int zipmapRawKeyLength(unsigned char *p)
返回一个key占用的总长度
encoded length + payload
```
## 9. zipmapRawValueLength
```
static unsigned int zipmapRawValueLength(unsigned char *p)
返回一个value占用的长度
(encoded length + signle byte free count + payload)
```
## 10. zipmapRawEntryLength
```
static unsigned int zipmapRawEntryLength(unsigned char *p)
返回指向key的p对应的entry的长度
entry = key + associated value + trailing free space
```
## 11. zipmapResize
```
static inline unsigned char *zipmapResize(unsigned char *zm, unsigned int len)
realloc之后设置zm[len-1] = ZIPMAP_END，返回zm
```
## 12. zipmapSet
```
unsigned char *zipmapSet(unsigned char *zm, unsigned char *key, unsigned int klen,
unsigned char *val, unsigned int vlen, int *update)

将key设置为给定的value，不存在key则新建
update指针如果非空，更新key被设为1，否则设为0

首先确定所需的长度，如果没有找到key则resize zipmap

找到的时候 判断p对应的free空间是否足够，不足够时也需要resize
之后对p后面的字段执行memmove

检查空闲空间和需要空间大小，空闲空间过大则在进行memmove
最后 p += zipmapEncodeLength(p, klen) memcpy(p, key, klen); p += klen
写入key
类似的方式写入value
```
## 13. zipmapDel
```
unsigned char *zipmapDel(unsigned char *zm, unsigend char *key, unsigned int klen, int *deleted)
删除特定的key,设置deleted的时候 未找到key设置为0否则为1
memmove, resize
```
## 14. zipmapRewind
```
unsigned char *zipmapRewind(unsigned char *zm)
return zm+1
```
## 15. zipmapNext
```
unsigned char *zipmapNext(unsigned char *zm, unsigned char **key,
    unsigned int *klen, unsigned char **value, unsigned int *vlen)

用来遍历完整的zipmap 调用方法:
unsigned char *i = zipmapRewind(my_zipmap);
while ((i = zipmapNext(i, &key, &klen, &value, &vlen)) != NULL) {
    pritnf("%d bytes key at $p\n", klen, key);
    printf("%d bytes value as $p\n, vlen, value);
}

通过zm +对应的key value长度获取对应位置的值
```
## 16. zipmapGet
```
int zipmapGet(unsigned char *zm, unsigned char *key, unsigned int klen,
        unsigned char **value, unsigned int *vlen)

检索一个key，并存储value的长度在vlen 值在value中
未检索到返回0，检索到返回1
```
## 17. zipmapExists
```
int zipmapExists(unsigned char *zm, unsigned char *key, unsigned int klen)

返回zipmapLookupRaw(zm, key, klen, NULL)是否为空
```
## 18. zipmapLen
```
unsigned int zipmapLen(unsigned char *zm)

返回zipmap内的entry总数，如果zm[0]小于ZIPMAP_BIGLEN
返回zm[0]
否则需要进行遍历
```
## 19. zipmapBlobLen
```
size_t zipmapBlobLen(unsigned char *zm)

返回raw size in bytes of a zipmap
方便在其他位置序列化zipmap(使用c的下标形式)
```
