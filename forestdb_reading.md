# reading forestdb source code

question:
- what's direct IO, which will bypass the OS page cache.
- the differernce between index id and sequence num;
- 

## doc struction 
```
struct fdb_doc_struct {
    /**
     * key length.
     */
    size_t keylen;
    /**
     * metadata length.
     */
    size_t metalen;
    /**
     * doc body length.
     */
    size_t bodylen;
    /**
     * actual doc size written on disk.
     */
    size_t size_ondisk;
    /**
     * Pointer to doc's key.
     */
    void *key;
    /**
     * Sequence number assigned to a doc.
     */
    fdb_seqnum_t seqnum;
    /**
     * Offset to the doc (header + key + metadata + body) on disk.
     */
    uint64_t offset;
    /**
     * Pointer to doc's metadata.
     */
    void *meta;
    /**
     * Pointer to doc's body.
     */
    void *body;
    /**
     * Is a doc deleted?
     */
    bool deleted;
    /**
     * Flags for miscellaneous doc properties.
     */
     uint32_t flags;
    /**
     * Use the seqnum set by user instead of auto-generating.
     */
#define FDB_CUSTOM_SEQNUM 0x01
} fdb_doc;
```
like leveldb,in forestdb doc have a flag indicating whether the doc has been deleted.

# /include/libforestdb
 - fdb_errors.h 
定义一些错误码，按linux编程的习惯，成功返回0值，失败返回负值。
 - fdb_types.h
 定义一些常用的类型，用uint类型包装一些flag、mode、statu、option，并定义一些值。
 - forestdb.h
 c风格的db api。

# /src

- arch.h
 针对不同的平台，封装一套统一的api供db使用。主要对不同平台下的多线程编程进行了封装，（其中包括spin lock、 thread、condition variable、 mutex）
 - atomic.h
 原子化操作的一些实现。利用atomic的compare_exchange_strong实现了atomic_setIfBigger、atomic_setIfLess。用于引用计数的RCValue、单线程引用计数指针SingleThreadedRCPtr、读写锁。
 - atomicqueue.h
 线程安全的队列实现，通过操作前后加解锁实现。
 - avltree.h、avltree.cc
 avl树，浸入式容器。用_get_entry宏取得被浸入的类型。
 - bgflusher.h、bgflusher.cc

- blockcache.h、blockcache.cc
定义了两个类型：FileBlockCache、BlockCacheManager
BlockCacheItem类：包含了被管理的block的bid、addr、flag（block是否为脏块，是否immutable或free to use）、score(用于cache？？？)信息
BlockCacheShard类：管理一个<bid, BlockCacheItem*>的hash_map（allBlocks:all cached blocks），两个<bid, BlockCacheItem*>的tree_map(dirtyDataBlocks:for dirty data blocks、 dirtyIndexBlocks: for dirty index Blocks), 一个列表cleanBlocks：for clean blocks。

- bnode.h、bnode.cc
BsaItem用于Bnode(BsArray)和Btree的键值对，用于array中，带array中的offset值，value可以是与key一起存储的二进制，也可以是指针，用isValueChildPtr区分。
BsaKvMeta kvPos（Position of key-value pair.）
        isPtr（Boolean flag indicates if corresponding key-value pair contains the pointer to a dirty inner child node (true),  otherwise false.）
        isDirtyChildTree（  Boolean flag indicates if corresponding key-value pair  contains the pointer to a dirty root node of next level child  tree (true), otherwise false. Since the pointer to the next root  node is treated as a (wrapped) value in B+tree level, 'isPtr'  value will not be set to true together.）

BsArray：
 * | <--                  arrayCapacity                --> |
 * |      |  <---        kvDataSize      --->  |           |
 * +------+-------+-------+-----------+--------+-----------+
 * | meta |  KV 0 |  KV 1 |    ...    | KV n-1 |   empty   |
 * +------+-------+-------+-----------+--------+-----------+
 * meta 为KvMeta的array。
  * kvMeta[i].kvPos: location of KV i, excluding the memory region for 'meta'.
 *                  'kvPos' value starts from zero.
 *   e.g.)
 *   arrayBaseOffset = 15
 *   kvMeta[0].kvPos = 0
 *   kvMeta[1].kvPos = 20
 * then, 'KV 0' is located at dataArray+15,
 * and   'KV 1' is located at dataArray+15+20.
 *
 * kvMeta[i].isPtr: boolean flag that indicates if
 *                  KV i contains pointer (true) or binary data (false).

 # btree.h/cc btree_new.h/cc
这应该是两个版本的b树，新版本中的b树实现有什么不同之处呢？


btree:
BtreeNodeAddr 四个属性:offset、 ptr、isDirty、isEmpty
                 offset       isDirty    isEmpty   ptr
pointer value     -             true        false   value
empty pointer     BLK_NOT_FOUND  false       true    -
binary data value  value offset   false       false  -
offset、ptr只用到一个，故写在一个union中

BtreeKvPair: key value keylen valuelen
BtreeKey: data length

btree_new:
BtreeV2Meta：元信息
多了个NodeActionType用于add、remove、replace操作。
