# ziplist
## 1. ZIPLIST OVERALL LAYOUT
```
<zlbytes> <zltail> <zllen> <entry> <entry> ... <zlend>

<uint32_t zlbytes> 保存跳跃表的总byte数，包括它本身的4byte
<uint32_t zltail> 是到表最后一个entry的offset
<uint16_t zllen> 是entry的数量,超过2^16-2时，值为2^16-1，需要去遍历ziplist
确认item的总数
<uint8_t zlend> 是一个特殊的entry，代表ziplist的结尾，
encoded as a signle byte equal to 255,其他的entry不会以255开始
```
## 2. ZIPLIST ENTRIES
```
<prevlen> <encoding> <entry-data>

每个entry会有一个前缀，具有两部分数据
1: 前一个entry的长度，这是为了能够从后向前遍历
2: entry encoding，代表entry的数据类型，有效的长度

有时encoding会代表entry数据,例如小整数
<prevlen> <encoding>

<prevlen>:
如果长度小于254bytes, 使用unsigned 8 bit int的一个byte存储
如果长度大于等于254, 会消费5bytes。
第一个byte被设置为254表示后面跟随着更大的数
后面的4byte表示前一个entry的长度

<prevlen 0-253> <encoding> <entry>

0xFE <4 bytes unsigned little endian prevleb> <encoding> <entry>

<encoding>:
当entry为string时，第一个byte的前两个bit会用来encoding的长度
当entry为integer时
```

