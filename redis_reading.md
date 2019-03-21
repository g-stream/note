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
	  

```
实际的底层编码类型
```
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
些部分转自https://blog.csdn.net/opensure/article/details/51375961
redis GEO实现主要包含了以下两项技术：

    1、使用geohash保存地理位置的坐标。
    2、使用有序集合（zset）保存地理位置的集合。
**geohash**
geohash的思想是将二维的经纬度转换成一维的字符串，geohash有以下三个特点：

    1、字符串越长，表示的范围越精确。编码长度为8时，精度在19米左右，而当编码长度为9时，精度在2米左右。
    2、字符串相似的表示距离相近，利用字符串的前缀匹配，可以查询附近的地理位置。这样就实现了快速查询某个坐标附近的地理位置。
    3、geohash计算的字符串，可以反向解码出原来的经纬度。

这三个特性让geohash特别适合表示二维hash值。这篇文章：GeoHash核心原理解析详细的介绍了geohash的原理，想要了解geohash实现的朋友可以参考这篇文章。
redis GEO命令实现

知道了redis使用有序集合（zset）保存地理位置数据(想了解redis有序集合的，可以参看这篇文章《有序集合对象》)，以及geohash的特性，就很容易理解redis是如何实现redis GEO命令了。细心的读者可能发现，redis没有实现地理位置的删除命令。不过由于GEO数据保存在zset中，可以用zrem来删除某个地理位置。

    geoadd命令增加地理位置的时候，会先计算地理位置坐标的geohash值，然后地理位置作为有序集合的member，geohash作为该member的score。然后使用zadd命令插入到有序集合。
    geopos命令则先根据地理位置获取geohash值，然后decode得到地理位置的坐标。
    geodist命令先根据两个地理位置各自得到坐标，然后计算两个坐标的距离。
    georadius和georadiusbymember使用相同的实现，georadiusbymember多了一步把地理位置转换成对应的坐标。然后查找该坐标和周围对应8个坐标符合距离要求的地理位置。因为geohash得到的值其实是个格子，并不是点，这样通过计算周围对应8个坐标就能解决边缘问题。由于使用有序集合保存地理位置，在对地列位置基于范围查询，就相当于实现了zrange命令，内部的实现确实与zrange命令一致，只是geo有些特别的处理，比如获得的某个地理位置，还需要计算该地理位置是否符合给定的距离访问。
    geohash则直接返回了地理位置的geohash值。

redis关于geohash使用了Ardb的geohash库geohash-int，redis使用的geohash编码长度为26位。可以精确到0.59m的精度。

## list（现已弃用，被quicklist取代）
redis内的双端链表数据结构定义。
```

typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct listIter {
    listNode *next;
    int direction;
} listIter;

typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```
## quicklist
相关数据结构：
```
/* Node, quicklist, and Iterator are the only data structures used currently. */

/* quicklistNode is a 32 byte struct describing a ziplist for a quicklist.
 * We use bit fields keep the quicklistNode at 32 bytes.
 * count: 16 bits, max 65536 (max zl bytes is 65k, so max count actually < 32k).
 * encoding: 2 bits, RAW=1, LZF=2.
 * container: 2 bits, NONE=1, ZIPLIST=2.
 * recompress: 1 bit, bool, true if node is temporarry decompressed for usage.
 * attempted_compress: 1 bit, boolean, used for verifying during testing.
 * extra: 10 bits, free for future use; pads out the remainder of 32 bits */
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;

/* quicklistLZF is a 4+N byte struct holding 'sz' followed by 'compressed'.
 * 'sz' is byte length of 'compressed' field.
 * 'compressed' is LZF data with total (compressed) length 'sz'
 * NOTE: uncompressed length is stored in quicklistNode->sz.
 * When quicklistNode->zl is compressed, node->zl points to a quicklistLZF */
typedef struct quicklistLZF {
    unsigned int sz; /* LZF size in bytes*/
    char compressed[];
} quicklistLZF;

/* quicklist is a 40 byte struct (on 64-bit systems) describing a quicklist.
 * 'count' is the number of total entries.
 * 'len' is the number of quicklist nodes.
 * 'compress' is: -1 if compression disabled, otherwise it's the number
 *                of quicklistNodes to leave uncompressed at ends of quicklist.
 * 'fill' is the user-requested (or default) fill factor. */
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned long len;          /* number of quicklistNodes */
    int fill : 16;              /* fill factor for individual nodes */
    unsigned int compress : 16; /* depth of end nodes not to compress;0=off */
} quicklist;

typedef struct quicklistIter {
    const quicklist *quicklist;
    quicklistNode *current;
    unsigned char *zi;
    long offset; /* offset in current ziplist */
    int direction;
} quicklistIter;

typedef struct quicklistEntry {
    const quicklist *quicklist;
    quicklistNode *node;
    unsigned char *zi;
    unsigned char *value;
    long long longval;
    unsigned int sz;
    int offset;
} quicklistEntry;
```
可以看到，quicklist实际上是存放有ziplist的quicklistNode的双端环形链表。
这篇文章对此进行了一个很好的总结http://zhangtielei.com/posts/blog-redis-quicklist.html。
        quicklist的结构为什么这样设计呢？总结起来，大概又是一个空间和时间的折中：
            > 双向链表便于在表的两端进行push和pop操作，但是它的内存开销比较大。首先，它在每个节点上除了要保存数据之外，还要额外保存两个指针；其次，双向链表的各个节点是单独的内存块，地址不连续，节点多了容易产生内存碎片。
            > ziplist由于是一整块连续内存，所以存储效率很高。但是，它不利于修改操作，每次数据变动都会引发一次内存的realloc。特别是当ziplist长度很长的时候，一次realloc可能会导致大批量的数据拷贝，进一步降低性能。

## ziplist
×**ziplist的内存结构**
> <zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>
各个段位均是小端模式
  <uint32_t zlbytes> 是unsigned integer list字节数
  <uint32_t zltail> 到最后一个元素的offset  This allows
  <uint16_t zllen> list中元素个数，如果个数大于于ss-l2^16-2则设为2^16-1,这时需要遍历才能得到元素个
  <uint8_t zlend>  a single byte equal to 255 表示list结尾。

<entry> =
  - <prevlen> <encoding>
  - <prevlen from 0 to 253> <encoding> <entry>
  - 0xFE <4 bytes unsigned little endian prevlen> <encoding> <entry>
<encoding> =
  - |00pppppp| - 1 byte
       > String value with length less than or equal to 63 bytes (6 bits).
       > "pppppp" represents the unsigned 6 bit length.
  - |01pppppp|qqqqqqqq| - 2 bytes
       > String value with length less than or equal to 16383 bytes (14 bits).
       > IMPORTANT: The 14 bit number is stored in big endian.
  - |10000000|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt| - 5 bytes
       > String value with length greater than or equal to 16384 bytes. Only the 4 bytes following the first byte represents the length
       > up to 32^2-1. The 6 lower bits of the first byte are not used and
       are set to zero.
       > IMPORTANT: The 32 bit number is stored in big endian.
  - |11000000| - 3 bytes
       > Integer encoded as int16_t (2 bytes).
  - |11010000| - 5 bytes
       > Integer encoded as int32_t (4 bytes).
  - |11100000| - 9 bytes
       > Integer encoded as int64_t (8 bytes).
  - |11110000| - 4 bytes
       > Integer encoded as 24 bit signed (3 bytes).
  - |11111110| - 2 bytes
       > Integer encoded as 8 bit signed (1 byte).
  - |1111xxxx| - (with xxxx between 0000 and 1101) immediate 4 bit integer.
       > Unsigned integer from 0 to 12. The encoded value is actually from 1 to 13 because 0000 and 1111 can not be used, so 1 should be subtracted from the encoded 4 bit value to obtain the right value.
  - |11111111| - End of ziplist special entry.
可以看到redis用了prelen使之能逆向遍历元素。

## hash

```
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;

typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;

/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;

/* If safe is set to 1 this is a safe iterator, that means, you can call
 * dictAdd, dictFind, and other functions against the dictionary even while
 * iterating. Otherwise it is a non safe iterator, and only dictNext()
 * should be called while iterating. */
typedef struct dictIterator {
    dict *d;
    long index;
    int table, safe;
    dictEntry *entry, *nextEntry;
    /* unsafe iterator fingerprint for misuse detection. */
    long long fingerprint;
} dictIterator;
```
redis的hash表为了保证rehash过程减少堵塞，采用了多次rehash的操作，具体实现在用两个dictht存放数据，不用rehash时，dictht[0]的rehashidx为-1,rehash时，从dictth[0]中的rehashidx号桶开始，把桶中链表上的元素，往dictht[1]中加（桶的个数总是2的倍数个，故只要原hash值与sizemask作与运算就是新idx）。rehash函数指定还传入一个int n，用于指定每次rehash时，从原dictht中移出几个有元素的桶。最后，如果dictht没元素了，则用dicthd[1]换下dicthd[0]。

插入时，根据是否在rehashing，将元素插到1号或0号dictht中。查找时，同样，先从dictht[0]开始找，如果在rehashing再找dictht[1]。

## zipmap
zipmap在内存中的格式：
`<zmlen><len>"foo"<len><free>"bar"<len>"hello"<len><free>"world"`
zmlen: 1 byte长，如果<254的值，则值本身就是zipmap的键值对个数，如果是>=254，则长度不定，要遍历整个zipmap才能知道
len  : 1 byte至5 byte，如果第一个byte < 254则第一个byte就表示长度，如果是254，则后面的4个bytes表示一个int（大端）。如果255表示这个zipmap的结尾。
free : 表示修改map后的留空大小。一个byte大小。

redis的底层数据结构大致在内存消耗与性能之间做了一个很好的平衡，也给了用户一些配置选项，以进行针对性的优化。