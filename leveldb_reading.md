# leveldb source analysis - basic classes and operation

- Iterator
    定义在include的iterator.h中，是一个抽象类，指定了一组 iterator的接口，如Seek,SeekToFirst等操作。要说的是这里面还有个RegisterCleanup方法，能向iterator中注册CleanupFunction，arg1,arg2三元组构成的算是析构函数吧，在iterator析构时被调用。
    里面有个CleanupNode cleanup_head_;成员，注册的函数在这里串成单链表。
    
- block
    - block的结构：

            |entry0|entry1|...|restarts arrays|num_of_restarts|trailer)
        - restarts arrays(store the offset of every restart point,以uint32存储)

            所以通过restart来跳过从头遍历。

        - trailer长度为5bytes|type(cmopressed?)(1 byte)|crc(uint32)|

        - num_of_restarts,以uint32存储

        
    - entry的结构：

            |shared_bytes(varint32)|no_shared(varint32)|value_length(varint32)|unshared key data|value data|
        trailer的结构：
        |type (char)是否进行了压缩|crc (uint32)|

    - block iter的实现：
            
        Next： 存当前entry的长度，加上当前entry的入口即为下个entry的地址。

        prev：从小于当前entry offset的最大restart point开始往后Next到当前entry的前一个entry。

        其他操作也类似，运用restart point加快遍历操作。
        

- footer
    - footer的结构：
            
            |metaindex_block_handle|index_block_handle|padding_bytes|magic|
        每个table文件尾部的文件信息。

        每个table文件有两个block handler, 分别为metaindex block handler、index block handler。

        每个footer encode后长度为固定值：2*BlockHandle::kMaxEncodedLength + 8（two block handles and a magic number，而两个block handler中存有offset和size数据，以Varint64形式存储，kMaxEncodedLength = 10 + 10，为了达到最大长度，两个block handler存完后添加padding bytes，最后再加上8字节长的magic number）。

- block
  
    在sstable内，leveldb数据内容的是以block存放的，最后添上metaindex、dataindex与footer。

- user_key and internal_key

    leveldb中对userkey附加了一些信息作为internal_key。首先是，leveldb中要删除key时，不会直接找到key存储的位置删除，而是通过插入一个标记有kTypeDeletion的key，告知这个key已经被删除了，当往下层merge操作时再对其进行删除操作。但这也会引发问题，对于两个sst中同样的key，如何确定哪个是最新的值？leveldb中，对每个entry还加入了sequence number，这是一个增的uint56,seq较大的那个entry即为最新的值。

    所以internal_key == |user_key|sequence_number|Ktype(kTypeValue/kTypeDeletion)|, 后两者，kType占用8bits，sequence_number 占用56bits，两者共占用8bytes。存储时，这8bytes附在user_key后面，成为internal_key的data。

    比较internal_key时，先比较其user_key的大小，如果一样大，则比较其sequence_number的大小。这里可以看到把sequence_number放到kType前面的好处，我们把sequence_number与kType合起来的那个8bytes当作uint64来比较就好了。

- log 
    - header

            |checksum(4bytes)|length(2bytes)|record_type(1byte)|
        record_type有如下几种：  kZeroType，kFullType，kFirstType，kMiddleType，kLastType。


通过这些数据格式，我们应该能看出leveldb底层的文件是以什么样的格式存放数据的。相关的代码主要放在table文件夹下，与db的log_writer.h、log_writer.cc中。

每个sst文件中放置一串block文件段，后附有metaindex、blockindex计录其offset与size。sst的inerator具体实现在two_level_iterator.h/cc中大致思想类似去图书馆中找书，先找到对应的书架，后从书架中找到对应的书。sstable文件的元信息封装成FileMetaData定义在version_edit.h中。
```
struct FileMetaData {
  int refs;
  int allowed_seeks;          // Seeks allowed until compaction
  uint64_t number;
  uint64_t file_size;         // File size in bytes
  InternalKey smallest;       // Smallest internal key served by table
  InternalKey largest;        // Largest internal key served by table

  FileMetaData() : refs(0), allowed_seeks(1 << 30), file_size(0) { }
};
```

leveldb会生成的文件类型：
```
enum FileType {
  kLogFile,
  kDBLockFile,
  kTableFile,
  kDescriptorFile,
  kCurrentFile,
  kTempFile,
  kInfoLogFile  // Either the current one, or an old one
};
```

- kLogFile：日志文件：[0-9]+.log。

    leveldb会先将数据写入log中，然后再写入memtable。前缀数字为FileNumber。
- kDBLockFile： 文件锁。

    一个db只能同时有一个db实例，故加锁实现主动保护。
- kTableFile： sstable文件：[0-9]+.sst

    层级的sstable数据文件
- kDescriptorfile： db元信息文件：MANIFEST-[0-9]+ 

    每当db中的状态改变（VersionSet），会将这次改变追加到descriptor文件中。后缀数字为FileNumber。
- kTempfile： 临时文件：[0-9]+.dbtmp 

    对db做修复时，会产生临时文件。前缀为FileNumber。
- kInfoLogfile： db运行时打印的日志文件 LOG

    db运行时，打印的info日志保存在LOG中。每次重新运行，如果已经存在LOG文件，会将LOG文件重名成LOG.old


## memtable.h/cc

实现了内存中的sst，内部是一个[skip list](https://en.wikipedia.org/wiki/Skip_list)。
大致原理是，一个随机高度的递增链表。每个结点带有一个高度值，每个高度有一个指针指向下一个高度大于该指针所在高度的结点的同高度的结点，如没有下个结点，则指向NULL。查找时从第一个节点的最高指针开始，如果下个结点比被查找元素小则向前，否则下降高度重复。插入时，随机生成高度，维护所有高度的指针。


## snapshot.h
快照功能，代码很简单，只是一个保存sequence的链表。leveldb中每个键值对在插入时，use_key都会被算成internal_key。internal_key == userkey+sequence+valuetype(normal or delete)。sequence是一个递增的数，每插入一次加一。故可以以此表示一个快照，恢复时滤过大于保存值的kv对即可。

## WriteBatch
对批量的write操作封装成WriteBatch。它会将userkey连同SequnceNumber和VAlueType先做encode，然后做decode，将数据insert到指定的Handler(memtable)上面。

## Compact (db_impl.cc version_set.cc)
db中有一个compact后台进程，负责将memtable持久化成sstable，以及均衡gkwhdb中各level的sstable。Compact进程会优先将已经写満的memtable dump成level-0的sstable（不会合并相同key或者清理已经删除的key）。然后，根据设计的策略选取level-n以及level-n+1中有key-range overlap的几个sstable进行merge(期间会合并相同的key以及清理删除的key)，最后生成若干个level-n+1的sstable。随着数据不断的写入和compact的进行，低level的sstable不娄向高level迁移。所有level中，只有level-0中的sstable因为是由memtable直接dump生成，文件之间key会有overlap。