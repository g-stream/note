
bnode.cc
bnode.h

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
总之，主要的是kv pair array 里的数据，前面attached的meta array用来快速找到对应的pair entry。
bnode是对这一结构的进一步封装，加上了所在b树中的深度、引用计数、数据的持久化（包括数据的dump undump方法、node所在的blocks的id、对应文件中的offset等信息）、b树结点的slipt/merge操作等。

bnodecache.cc
bnodecache.h
其中定义了几个相关类：
IndexBlkMeta —— Index Block Meta structure that is suffixed at the end of every raw Bnode when written to file.（写入文件时附在每个block的最后面，大小为16 bytes放在）。新版本支持的功能，有nextbid可以不连续的存放block。
```
struct IndexBlkMeta {
    IndexBlkMeta() {
        nextBid = BLK_NOT_FOUND;
        sbBmpRevnumHash = 0;
        memset(reserved, 0x0, 5);
        marker = BLK_MARKER_BNODE;
    }

    void set(bid_t _bid, uint64_t _sb_revnum) {
        nextBid = _endian_encode(_bid);
        sbBmpRevnumHash = _sb_revnum & 0xffff;
        sbBmpRevnumHash = _endian_encode(sbBmpRevnumHash);
        marker = BLK_MARKER_BNODE;
    }

    void decode() {
        nextBid = _endian_decode(nextBid);
        sbBmpRevnumHash = _endian_decode(sbBmpRevnumHash);
    }

    bid_t nextBid;
    uint16_t sbBmpRevnumHash;
    uint8_t reserved[5];
    uint8_t marker;
};
```
BnodeCacheShard:管理bnode cache（dirty or clean）的类，实现了lru功能。
```

/**
 * Shard Bnodecache instance
 */
class BnodeCacheShard {
public:
    BnodeCacheShard(size_t _id)
        : id(_id)
    {
        spin_init(&lock);
        list_init(&cleanNodes);
    }

    ~BnodeCacheShard() {
        for (auto entry : allNodes) {
            delete entry.second;
        }
        spin_destroy(&lock);
    }

private:
    friend class BnodeCacheMgr;
    friend class FileBnodeCache;

    size_t getId() const {
        return id;
    }

    bool empty() {
        // Caller should grab the shard lock before calling this function
        return list_empty(&cleanNodes) && dirtyIndexNodes.empty();
    }

    // Shard id
    size_t id;
    // Lock to synchronize access to cleanNodes, dirtyIndexNodes, allNodes
    spin_t lock;
    // LRU list of clean index nodes
    struct list cleanNodes;
    // Tree map of dirty index nodes
    std::map<cs_off_t, Bnode*> dirtyIndexNodes;
    // Hashtable of all the btree nodes belonging to this shard
    std::unordered_map<cs_off_t, Bnode*> allNodes;
};
```
FileBnodeCache:管理一个文件对应的shards cache
```
/**
 * File Bnodecache instance
 */
class FileBnodeCache {
public:
    FileBnodeCache(std::string fname, FileMgr* file, size_t num_shards);

    ~FileBnodeCache() { }

    /* Fetch filename that the bnode cache instance is associated with */
    const std::string& getFileName(void) const;

    /* Fetch the filemgr instance for the bnode cache */
    FileMgr* getFileManager(void) const;

    /* Fetch the Reference count of the file bnode cache instance */
    uint32_t getRefCount(void) const;

    /* Fetch the number of victims selected for eviction */
    uint64_t getNumVictims(void) const;

    /* Fetch the number of items in the bnode cache */
    uint64_t getNumItems(void) const;

    /* Fetch the total number of items written to the bnode cache */
    uint64_t getNumItemsWritten(void) const;

    /* Get the last time of access for this bnode cache */
    uint64_t getAccessTimestamp(void) const;

    /* Get the number of shards for this bnode cache */
    size_t getNumShards(void) const;

    /* Update last time of access of this bnode cache */
    void setAccessTimestamp(uint64_t timestamp);

    /**
     * Check if a file bnode cache is empty of not.
     * @return true If a file bnode cache is empty
     */
    bool empty();

    /* Acquire the locks of all shards within this bnode cache */
    void acquireAllShardLocks();

    /* Release the locks of all shards within this bnode cache */
    void releaseAllShardLocks();

    /* Sets eviction in progress flag */
    bool setEvictionInProgress(bool to);

private:
    friend class BnodeCacheMgr;

    const std::string filename;
    // File manager instance for the file
    FileMgr* curFile;
    // Shards of the bnode cache for a file
    std::vector<std::unique_ptr<BnodeCacheShard>> shards;

    std::atomic<uint32_t> refCount;
    std::atomic<uint64_t> numVictims;
    std::atomic<uint64_t> numItems;
    std::atomic<uint64_t> numItemsWritten;
    std::atomic<uint64_t> accessTimestamp;

    // Flag if eviction is running on this victim
    std::atomic<bool> evictionInProgress;
};
```
BnodeCacheMgr：单例类，管理一个db中所有的FileBnodeCache。
blockcache.cc
blockcache.h

bnodemgr.cc
bnodemgr.h
管理一个文件中所有的bnode维护cleanNodes与dirtyNodes两个子集，实现了cleannode到dirtynode的操作（建新node，将原有文件中的cleannode加上stale标记），b+树node的读写等操作。


docio.cc
docio.h
实际的数据库文件存储格式。
```
struct docio_object {
    struct docio_length length;
    timestamp_t timestamp;
    void *key;
    union {
        fdb_seqnum_t seqnum;
        uint64_t doc_offset;
    };
    void *meta;
    void *body;
};

    struct docio_length {
        keylen_t keylen;
        uint16_t metalen;
        uint32_t bodylen;
        uint32_t bodylen_ondisk;
        uint8_t flag;
        uint8_t checksum;
    };
```

staleblock.cc
staleblock.h
db的b+树在动态的操作过程中会如果某些kvpair被delete，对应的db文件中的相应block也会被标记成stale，统一由StaleDataManager管理，其负责相关区间的合并，当前的收集当前的staleinfo并以system doc的形式写入文件中。
```
    /*
     * << stale block system doc structure >>
     * [previous doc offset]: 8 bytes (0xffff.. if not exist)
     * [previous header BID]: 8 bytes (0xffff.. if not exist)
     * [KVS info doc offset]: 8 bytes (0xffff.. if not exist)
     * [Default KVS seqnum]:  8 bytes
     * [# items]:             4 bytes
     * ---
     * [position]:            8 bytes
     * [length]:              4 bytes
     * ...
     */
```

btree_kv.cc
btree_kv.h
btree_str_kv.cc
btree_str_kv.h
btree_fast_str_kv.cc
btree_fast_str_kv.h
btree_var_kv_ops.h
有关bnode上的kv对的操作。
```

/**
 * === node->data structure overview ===
 *
 * [offset of key 1]: sizeof(key_len_t) bytes
 * [offset of key 2]: ...
 * ...
 * [offset of key n]: ...
 * [offset of key n+1]: points to the byte offset right after the end of n-th entry
 * [key 1][value 1]
 * [key 2][value 2]
 * ...
 * [key n][value n]
 *
 * Note that the maximum node size is limited to 2^(8*sizeof(key_len_t)) bytes
 */

/**
 * === Variable-length key structure ===
 *
 * B+tree key (8 bytes): pointer to the address that the actual key is stored.
 *
 * Actual key:
 * <-- 2 --><-- key len -->
 * [key len][  key string ]
 */
```


btree.cc
btree.h
btree_new.cc
btree_new.h
两个版本的b树实现。

btreeblock.cc
btreeblock.h



file_handle.cc
file_handle.h
filemgr.cc
filemgr.h

commit_log.cc
commit_log.h
compaction.cc
compaction.h
compactor.cc
compactor.h
fdb_internal.h

hbtrie.cc
hbtrie.h
internal_types.h
iterator.cc
iterator.h
kv_instance.cc
kvs_handle.cc
kvs_handle.h
transaction.cc
wal.cc
wal.h
superblock.cc
superblock.h