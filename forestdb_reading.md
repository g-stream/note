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
