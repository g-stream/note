# reading leveldb
- Iterator
    定义在include的iterator.h中，是一个抽象类，指定了一组 iterator的接口，如Seek,SeekToFirst等操作。要说的是这里面还有个RegisterCleanup方法，能向iterator中注册CleanupFunction，arg1,arg2三元组构成的算是析构函数吧，在iterator析构时被调用。
    里面有个CleanupNode cleanup_head_;成员，注册的函数在这里串成单链表。
    
- block
    - block的结构：
        |entry0|entry1|...|restarts arrays(store the offset of every restart point,以uint32存储)|num_of_restarts,以uint32存储|trailer(不计入长度？？？长度为5bytes)
        所以通过restart来跳过从头遍历。
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
  
        leveldb 数据内容的是以block存放的。

- user_key and internal_key
    leveldb中对userkey附加了一些信息作为internal_key。首先是，leveldb中要删除key时，不会直接找到key存储的位置删除，而是通过插入一个标记有kTypeDeletion的key，告知这个key已经被删除了，当往下层merge操作时再对其进行删除操作。但这也会引发问题，对于两个sst中同样的key，如何确定哪个是最新的值？leveldb中，对每个entry还加入了sequence number，这是一个增的uint56,seq较大的那个entry即为最新的值。
    所以internal_key == |user_key|sequence_number|Ktype(kTypeValue/kTypeDeletion)|, 后两者，kType占用8bits，sequence_number 占用56bits，两者共占用8bytes。存储时，这8bytes附在user_key后面，成为internal_key的data。
    比较internal_key时，先比较其user_key的大小，如果一样大，则比较其sequence_number的大小。这里可以看到把sequence_number放到kType前面的好处，我们把sequence_number与kType合起来的那个8bytes当作uint64来比较就好了。

- log 
    - header
        结构：
        |checksum(4bytes)|length(2bytes)|record_type(1byte)|
        record_type有如下几种：  kZeroType，kFullType，kFirstType，kMiddleType，kLastType。


通过这些数据格式，我们应该能看出leveldb底层的文件是以什么样的格式存放数据的。相关的代码主要放在table文件夹下，与db的log_writer.h、log_writer.cc中。
每个sst文件中放置一串block文件段，后附有metaindex、blockindex计录其offset与size。sst的inerator具体实现在two_level_iterator.h/cc中大致思想类似去图书馆中找书，先找到对应的书架，后从书架中找到对应的书。


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

memtable.h/cc
实现了内存中的sst，内部是一个[skip list](https://en.wikipedia.org/wiki/Skip_list)。
大致原理是，一个随机高度的递增链表。每个结点带有一个高度值，每个高度有一个指针指向下一个高度大于该指针所在高度的结点的同高度的结点，如没有下个结点，则指向NULL。查找时从第一个节点的最高指针开始，如果下个结点比被查找元素小则向前，否则下降高度重复。插入时，随机生成高度，维护所有高度的指针。


snapshot.h
快照功能，代码很简单，只是一个保存sequence的链表。leveldb中每个键值对在插入时，use_key都会被算成internal_key。internal_key == userkey+sequence+valuetype(normal or delete)。sequence是一个递增的数，每插入一次加一。故可以以此表示一个快照，恢复时滤过大于保存值的kv对即可。

