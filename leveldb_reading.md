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

        每个table文件尾部的文件信息。
        每个table文件有两个block handler, 分别为metaindex block handler、index block handler。
        每个footer encode后长度为最大值：2*BlockHandle::kMaxEncodedLength + 8（two block handles and a magic number，而两个block handler中存有offset和size数据，以Varint64形式存储，kMaxEncodedLength = 10 + 10）。

- block
  
        leveldb 数据内容的是以block存放的。

