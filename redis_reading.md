# redis date types and implementation detials

Redis is not a plain key-value store, it is actually a data structures server, supporting different kinds of values. What this means is that, while in traditional key-value stores you associated string keys to string values, in Redis the value is not limited to a simple string, but can also hold more complex data structures. The following is the list of all the data structures supported by Redis, which will be covered separately in this tutorial:

-    Binary-safe strings.
-    Lists: 
    > Collections of string elements sorted according to the order of insertion. They are basically linked lists.
-    Sets: collections of unique, unsorted string elements.
-    Sorted sets
    > Similar to Sets but where every string element is associated to a floating number value, called score. The elements are always taken sorted by their score, so unlike Sets it is possible to retrieve a range of elements (for example you may ask: give me the top 10, or the bottom 10).
-    Hashes
    > Hashes are maps composed of fields associated with values. Both the field and the value are strings. This is very similar to Ruby or Python hashes.
-    Bit arrays (or simply bitmaps)
    > it is possible, using special commands, to handle String values like an array of bits: you can set and clear individual bits, count all the bits set to 1, find the first set or unset bit, and so forth.
-    HyperLogLogs
    > this is a probabilistic data structure which is used in order to estimate the cardinality of a set. Don't be scared, it is simpler than it seems... See later in the HyperLogLog section of this tutorial.
-    Streams
    > append-only collections of map-like entries that provide an abstract log data type. 

## Binary-safe strings

```
typedef char *sds;

/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};

#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
#define SDS_TYPE_MASK 7
#define SDS_TYPE_BITS 3
#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));
#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))
#define SDS_TYPE_5_LEN(f) ((f)>>SDS_TYPE_BITS)
```
以不同的sdshdr表示不同长度的字符串，对一个sds，实际上分配对应sdshdr空间加上一个data段的buf的长度的空间，并以sdshdr中的data段地址开始存字符串（结尾存放'\0'，以保证与c字符串的兼容，strcpy等;同时，len信息的存在保证了它二进制安全，能存放图片，视频等二进制文件），用data开始的字段表示这个sds。sdshdr中带有长度信息，相较于c字符串，它以O(1)复杂度求len，并且有了长度信息后更安全，防止越界错误。
sdshdr中还保存free表示buf可用空间。通过buf的预分配，一些小修改只需要运用剩下的buf就行，优化操作。

## object




```
/* A redis object, that is a type able to hold a string / list / set */

/* The actual Redis Object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */

/* The "module" object type is a special one that signals that the object
 * is one directly managed by a Redis module. In this case the value points
 * to a moduleValue struct, which contains the object value (which is only
 * handled by the module itself) and the RedisModuleType struct which lists
 * function pointers in order to serialize, deserialize, AOF-rewrite and
 * free the object.
 *
 * Inside the RDB file, module types are encoded as OBJ_MODULE followed
 * by a 64 bit module type ID, which has a 54 bits module-specific signature
 * in order to dispatch the loading to the right module, plus a 10 bits
 * encoding version. */
#define OBJ_MODULE 5    /* Module object. */
#define OBJ_STREAM 6    /* Stream object. */
	  

/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
  * is set to one of this fields for this object. */
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

## HyperLogLog

该算法用来估计一个集合的基数，属于概率算法，返回一个近似的结果。但是大节省了内存。其的原理大致来说就是，对一次抛硬币的伯努力事件，以第一次出现反面的时候抛硬币的次数k来估算共抛了2^k次。为了减少误差，多次抛硬币的过程，分别计下其第一次出现反面时抛的次数，以次数的最大值来估计。

dense:
`11000000|222211|33333322|55444444|....`
连续存储6bits长的uint。
对6bit uint的在char数组中的index，用如下公式求得：
> b0 = 6 * pos / 8
offset，用如下公式：
> fb = 6 * pos % 8
怎么操作6 bits的register，redis设计了一套宏操作：
```



## GEO