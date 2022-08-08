参考资料：https://www.bilibili.com/video/BV1Jq4y1p7Rw?p=6

## 查看redis数据结构及其底层编码方法

### 查看数据结构

命令：`TYPE {key}`

案例：

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/img/image-20220808204129543.png" alt="image-20220808204129543" style="zoom:67%;" />

### 查看底层编码方式

这里的编码方式其实就是外表数据结构的底层实现

命令：`Object Encoding {key}`

案例：

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/img/image-20220808204355909.png" alt="image-20220808204355909" style="zoom: 60%;" />

可以看到key "name" 对应的值 "mike" 是一个字符串类型的值，其底层实现为 "embstr"。

### 源码层面上看

来到server.h，对应的结构体为 `redisObject` ：

```c
typedef struct redisObject {
    unsigned type:4;		// <-- 类型值
    unsigned encoding:4;	// <--底层编码值
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

type类型对应的值在 `server.h` 中（5种基本类型）：

```c
/* The actual Redis Object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */
```

encoding底层类型值也在 `server.h` 中（11种底层类型）：

```c
/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
/*
 * 对象编码。某些类型的对象（如字符串和哈希）可以在内部以多种方式表示。
 * 对象的“编码”字段设置为此对象的此字段之一。
 * */
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
```

各个数据结构对应的底层实现参考小林coding图（其实不太完全）：

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/img/image-20220808205548819.png" alt="image-20220808205548819" style="zoom: 50%;" />

