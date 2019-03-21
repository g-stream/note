# infrastructure of forestdb
# 基础设施的封装
1. arch.h
针对不同的平台，提供了统一的spinlock、mutex、thread、malloc、malloc_align接口。
2. atomic.h
用模板类的方式实现了对于atomic的setIfXxx操作，运用了cas的手法。如下
```
template <typename T>
void atomic_setIfBigger(std::atomic<T> &obj, const T &newValue) {
    T oldValue = obj.load();
    while (newValue > oldValue) {
        if (obj.compare_exchange_strong(oldValue, newValue)) {
            break;
        }
        oldValue = obj.load();
    }
}
```
实现值引用计数的管理类，并在其上实现单线程的引用计数指针 class SingleThreadedRCPtr
封装出一套read-write lock。
3. breakpad_dummy.cc
4. breakpad.h
5. breakpad_linux.cc
6. breakpad_win32.cc
上面四者应该是debug用，日后再研究 。
7. sync_object.h
一般condition_variable与mutex是放一起使用的，这里直接把它们合在一个类SyncObject里了，代码如下：
```
class SyncObject : public std::mutex {
public:
    SyncObject() {
    }

    ~SyncObject() {
    }

    void wait(UniqueLock& lock) {
        cond.wait(lock);
    }

    void wait_for(UniqueLock& lock,
                  const double secs) {
        cond.wait_for(lock, std::chrono::milliseconds(int64_t(secs * 1000.0)));
    }

    void wait_for(UniqueLock& lock,
                  const uint64_t nanoSecs) {
        cond.wait_for(lock, std::chrono::nanoseconds(nanoSecs));
    }

    void notify_all() {
        cond.notify_all();
    }

    void notify_one() {
        cond.notify_one();
    }

private:
    std::condition_variable cond;

    DISALLOW_COPY_AND_ASSIGN(SyncObject);
};
```
8. fdb_errors.cc
是fdb_errors.h的反向map，由err_code返回错误的描述信息。
9. filemgr_ops.cc
10. filemgr_ops.h
11. filemgr_ops_linux.cc
12. filemgr_ops_windows.h
libforestdb/fdb_types.h中定义了如下的文件操作类型，在如上文件中，根据不同的平台linux or windows完成了这一类型的填空。
```
typedef struct filemgr_ops {
    fdb_fileops_handle (*constructor)(void *ctx);
    fdb_status (*open)(const char *pathname, fdb_fileops_handle *fops_handle,
                       int flags, mode_t mode);
    fdb_ssize_t (*pwrite)(fdb_fileops_handle fops_handle, void *buf, size_t count,
                          cs_off_t offset);
    fdb_ssize_t (*pread)(fdb_fileops_handle fops_handle, void *buf, size_t count,
                         cs_off_t offset);
    int (*close)(fdb_fileops_handle fops_handle);
    cs_off_t (*goto_eof)(fdb_fileops_handle fops_handle);
    cs_off_t (*file_size)(fdb_fileops_handle fops_handle,
                          const char *filename);
    int (*fdatasync)(fdb_fileops_handle fops_handle);
    int (*fsync)(fdb_fileops_handle fops_handle);
    void (*get_errno_str)(fdb_fileops_handle fops_handle, char *buf, size_t size);
    voidref (*mmap)(fdb_fileops_handle fops_handle, size_t length, void **aux);
    int (*munmap)(fdb_fileops_handle fops_handle, void *addr, size_t length, void *aux);

    // Async I/O operations
    int (*aio_init)(fdb_fileops_handle fops_handle, struct async_io_handle *aio_handle);
    int (*aio_prep_read)(fdb_fileops_handle fops_handle,
                         struct async_io_handle *aio_handle, size_t aio_idx,
                         size_t read_size, uint64_t offset);
    int (*aio_submit)(fdb_fileops_handle fops_handle,
                      struct async_io_handle *aio_handle, int num_subs);
    int (*aio_getevents)(fdb_fileops_handle fops_handle,
                         struct async_io_handle *aio_handle, int min,
                         int max, unsigned int timeout);
    int (*aio_destroy)(fdb_fileops_handle fops_handle,
                       struct async_io_handle *aio_handle);

    int (*get_fs_type)(fdb_fileops_handle src_fd);
    int (*copy_file_range)(int fs_type, fdb_fileops_handle src_fops_handle,
                           fdb_fileops_handle dst_fops_handle, uint64_t src_off,
                           uint64_t dst_off, uint64_t len);
    void (*destructor)(fdb_fileops_handle fops_handle);
    void *ctx;
} fdb_filemgr_ops_t;
```
可见io操作还用了较新的aio api。


# 功能性函数等
common.h
checksum.cc
checksum.h
encryption_aes.cc
encryption_bogus.cc
encryption.cc
encryption.h
forestdb_endian.h
hash_functions.cc
hash_functions.h
utils/bitwise_utils.h
utils/crc32.cc
utils/crc32.h
utils/crypto_primitives.h
utils/debug.cc
utils/debug.h
utils/gettimeofday_vs.cc
utils/gettimeofday_vs.h
utils/iniparser.cc
utils/iniparser.h
utils/memleak.cc
utils/memleak.h
utils/partiallock.cc
utils/partiallock.h
utils/system_resource_stats.cc
utils/system_resource_stats.h
utils/time_utils.cc
utils/time_utils.h
utils/timing.cc
utils/timing.h

# 数据库配置
config.cmake.h
configuration.cc
configuration.h
forestdb对其数据库与kvs_handle有对应的配置项，可以查看相应代码，代码的注释比较详细。
version.cc
version.h

# 基础的数据结构
atomicqueue.h
TODO
avltree.cc
avltree.h
list.cc
list.h
avltree与list均为经典的侵入式设计
memory_pool.cc
memory_pool.h
实现是一个`vector<uint_8*>`存放着预分配的内存块地址，与一个`queue<int>`存放空闲的内存块的编号
ringbuffer.h
模板类，为一个定长的T类型数组，存放最多数组长度个T类型元素。如越界，则从头开始存放并标记wrap。
hash.cc
hash.h
一个hash的框架，建表时可以传入不同的nbuckets、hash_func、hash_cmp_func。对于hash冲突，提供了两种解决方法，list与avltree，两者通过_HASH_TREE这个宏定义，在编译时选择。

# 后台任务
bgflusher.cc
bgflusher.h
实现了一个管理后台回写数据的单例类BgFlusher。BgFlusher管理一个openFiles的avltree。打开的文件以openfiles_elem的形式，串在openFiles的avltree上，一组线程对其进行flush操作。
```
struct openfiles_elem {
    char filename[FDB_MAX_FILENAME_LEN];
    FileMgr *file;
    fdb_config config;
    uint32_t register_count;
    bool background_flush_in_progress;
    ErrLogCallback *log_callback;
    struct avl_node avl;
};
```
初始化时的操作，打开多个线程，每个运行友员函数bgflusher_thread，其调用成员函数void * BgFlusher::bgflusherThread()。
```
BgFlusher * BgFlusher::createBgFlusher(struct bgflusher_config *config)
{
    BgFlusher *tmp = bgflusherInstance.load();
    if (tmp == nullptr) {
        LockHolder l_lock(bgfLock);
        tmp = bgflusherInstance.load();
        if (tmp == nullptr) {
            tmp = new BgFlusher(config->num_threads);
            bgflusherInstance.store(tmp);
            // We must create threads only after the singleton instance is ready
            for (size_t i = 0; i < config->num_threads; ++i) {
                thread_create(&tmp->bgflusherThreadIds[i], bgflusher_thread,
                              NULL);
            }
        }
    }
    return tmp;
}

void *bgflusher_thread(void *voidargs) {
    BgFlusher *bgf = BgFlusher::getBgfInstance();
    fdb_assert(bgf, bgf, NULL);
    return bgf->bgflusherThread();
}

void * BgFlusher::bgflusherThread()
{
    fdb_status fs;
    struct avl_node *a;
    FileMgr *file;
    struct openfiles_elem *elem;
    ErrLogCallback *log_callback = NULL;

    while (1) {
        uint64_t num_blocks = 0;

        UniqueLock l_lock(bgfLock);
        a = avl_first(&openFiles);//取第一个元素
        while(a) {//遍历所有注册的文件，进行处理，无效的file或file正在被处理，会直接找下一个的file
            filemgr_open_result ffs;
            elem = _get_entry(a, struct openfiles_elem, avl);
            file = elem->file;
            if (!file) {//无fileMgr的元素就删除它
                a = avl_next(a);
                avl_remove(&openFiles, &elem->avl);
                free(elem);
                continue;
            }

            if (elem->background_flush_in_progress) {//如果有别的线程正在处理这个file，则跳过它。
                a = avl_next(a);
            } else {
                elem->background_flush_in_progress = true;//开始处理这个文件，先标记自己的所有权
                log_callback = elem->log_callback;
                ffs = FileMgr::open(file->getFileName(), file->getOps(),
                                    file->getConfig(), log_callback);
                fs = (fdb_status)ffs.rv;
                l_lock.unlock();//下面的因为已经做了标记，并打开了文件，下面的代码可以放心操作，不用锁保护了，让出锁。
                if (fs == FDB_RESULT_SUCCESS) {
                    num_blocks += file->flushImmutable(log_callback);
                    FileMgr::close(file, false, file->getFileName(), log_callback);

                } else {
                    fdb_log(log_callback, fs,
                            "Failed to open the file '%s' for background flushing\n.",
                            file->getFileName());
                }
                l_lock.lock();//加锁并标记处理完成，文件不被我占有。
                elem->background_flush_in_progress = false;
                a = avl_next(&elem->avl);
                if (bgflusherTerminateSignal) {
                    return NULL;
                }
            }
        }
        l_lock.unlock();

        mutex_lock(&syncMutex);
        if (bgflusherTerminateSignal) {//有结束信号则退出
            mutex_unlock(&syncMutex);
            break;
        }
        if (!num_blocks) {
            thread_cond_timedwait(&syncCond, &syncMutex,
                                  (unsigned)(bgFlusherSleepInSecs * 1000));
        }
        if (bgflusherTerminateSignal) {
            mutex_unlock(&syncMutex);
            break;
        }
        mutex_unlock(&syncMutex);
    }
    return NULL;
}

```

workload.h
定义了任务的工况，读多还是写多还是平衡。
taskable.h  
提供一个Taskable接口
```
class Taskable {
public:
    /*
        Return a name for the task, used for logging
    */
    virtual const std::string& getName() const = 0;

    /*
        Return a 'group' ID for the task.

        The use-case here is to allow the lookup of all tasks
        owned by this taskable.

        The address of the object is a safe GID.
    */
    virtual task_gid_t getGID() const = 0;

    /*
        Return the workload priority for the task
    */
    virtual bucket_priority_t getWorkloadPriority() const = 0;

    /*
        Set the taskable object's workload priority.
    */
    virtual void setWorkloadPriority(bucket_priority_t prio) = 0;

    /*
        Return the taskable object's workload policy.
    */
    virtual WorkLoadPolicy& getWorkLoadPolicy() = 0;

    /*
        Called with the time spent queued
    */
    virtual void logQTime(type_id_t taskType, hrtime_t enqTime) = 0;

    /*
        Called with the time spent running
    */
    virtual void logRunTime(type_id_t taskType, hrtime_t runTime) = 0;

protected:
    virtual ~Taskable() {}
};
```
task_type.h
task_priority.cc
task_priority.h
定义了不同任务的类型与其优先级，目前只有bgflusher与compactor。
```
// Priorities for Read-only IO tasks

// Priorities for Auxiliary IO tasks

// Priorities for Read-Write IO tasks
const Priority Priority::CompactorPriority(COMPACTOR_ID, 2);
const Priority Priority::BgFlusherPriority(BGFLUSHER_ID, 1);

// Priorities for NON-IO tasks
```
globaltask.cc
globaltask.h
taskqueue.cc
taskqueue.h
executorthread.cc
executorthread.h
executorpool.cc
executorpool.h
futurequeue.h
tasks.h