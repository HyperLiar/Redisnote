# Sds
## 1. what is sds
```
typedef char *sds;
```
## 2. member in sds
```
uint32_t len            /* used length */
uint32_t alloc          /* excluding the header and null terminator */
unsigned char flags     /* 3 lsb of type, 5 unused bits */
char buf[]              /* string */
```
## 3. sdslen
```
static inline size_t sdslen(const sds s)

return len
```
## 4. sdsavail
```
static inline size_t sdsavail(const sds s)

return alloc - len
```
## 5. sdssetlen
```
static inline void sdssetlen(sds s, size_t newlen)

change len to newlen
```
## 6. sdsinclen
```
static inline void sdsinclen(sds s, size_t inc)

add inc to len
```
## 7. sdsalloc
```
static inline size_t sdsalloc(const sds s)

return alloc
```
## 8. sdssetalloc
```
static inline void sdssetalloc(sds s, size_t newlen)

change allow to newlen
```
## 9. sdsHdrSize
```
static inline int sdsHdrSize(char type)

return sizeof(sds of the type)
```
