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
