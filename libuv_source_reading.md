# libuv source reading
```
tree libuv-v1.9.1 -L 1

../libuv-v1.9.1/
    .
    .
    .
    .
    .
├── include  //libuv对外提供的api
    .
    .
    .
├── libtool
    .
    .
├── Makefile
    .
    .
    .
├── samples
├── src
    ├── fs-poll.c
    ├── heap-inl.h
    ├── inet.c
    ├── queue.h
    ├── threadpool.c
    ├── unix //directory for unix plantform
    ├── uv-common.c
    ├── uv-common.h
    ├── version.c
    └── win  //directory for windows

├── test
    .
    .
````
主要代码在src include 中，src中的unix是对应unix的内部实现代码，将unix的系统调用包装成libuv的统一接口。

## 源码学习之c语言技巧
- c/c++库项目的代码组织形式

    - 对外api与内部实现分开

        libuv中include专门提供对外的api，src中提供实现。

- 对不同平台，先抽象出一层内部的api，针对不同平台，提供实现。

    如：uv__xxxxxx(....)为内部api, src/win， src/unix分别实现这些函数，在uv-common.c中统一包装成uv_xxxxxx(...)的接口。

- 类似linux内核list的浸入式容器设计。

    在src/tree.h, src/queue.h中。将提供操作的数据结构嵌入被操作的数据结构中，通过对嵌入式结构的指针等操作实现queue，rbtree，spraytree等数据结构；通过container_of的宏，计算嵌入位置的offset来取得container。
- c语言的面向对象编程。

    不同的平台，系统的api不同。同样的功能，针对不同的系统，提供不同的实现方法。具体做法是，对一个结构体，其中某种功能相关的field字段用宏表示，结构体中直接填入宏，然后每个平台的具体实现这个field

```
#define container_of(ptr, type, member) \
  ((type *) ((char *) (ptr) - offsetof(type, member)))
```

- compiler attribute

    使用了一些compile attribute来进行内存对齐，分支跳转等方面的优化。如：
    ```
    #ifndef UV__UNUSED
    # if __GNUC__
    #  define UV__UNUSED __attribute__((unused))
    # else
    #  define UV__UNUSED
    # endif
    #endif
    .
    .
    .
    #if _MSC_VER
         __declspec(align(8)) char buffer[8192];
    #else
        __attribute__ ((aligned (8))) char buffer[8192];
    #endif
    .
    .
    .
    #ifdef _MSC_VER  /* msvc */
    # define NO_INLINE __declspec(noinline)
    #else  /* gcc */
    # define NO_INLINE __attribute__ ((noinline))
    #endif
    ```
    其实这一点，libev做得更多，它直接将copy了ecb.h的代码。实际工程中，我觉得还是采用已有库中的东西比较好，平时可以看其代码学习基层原理，要是自己造轮子的话，繁琐且可靠性太低了；推荐用像之前说的ecb.h或libcork等库。

- 宏的技巧
    一些src/tree.h,src/queue.h中的技巧，就像c++的模板一样，比较复杂，利用了浸入式容器的思想，留着慢慢分析。一些比较简单的，像UV_ERRNO_MAP的宏。
    ```
    #define UV_ERRNO_MAP(XX)                                                      \
    XX(E2BIG, "argument list too long")                                         \
    XX(EACCES, "permission denied")                                             \
    XX(EADDRINUSE, "address already in use")                                    \
    XX(EADDRNOTAVAIL, "address not available")            
    .
    .
    .  
    XX(EMLINK, "too many links")                                                \
    XX(EHOSTDOWN, "host is down")                                               \
    先定义一个字段之间的对应map关系，之后通过传入XX将其扩展开。比如这像
  typedef enum {
    #define XX(code, _) UV_ ## code = UV__ ## code,
        UV_ERRNO_MAP(XX)
    #undef XX
    UV_ERRNO_MAX = UV__EOF - 1
    } uv_errno_t;
    ```
    这算是一个比较常用的技巧，像lua语言中，对指令集的定义也用到了类似的宏。通过先定义的map关系，改变传入的XX宏，可以完成两者的相互映射，批量声名或定义，enum变量的定义等功能。有时候将这个宏直接写入一个文件中，通过引入文件来使用，方便统一修改。

## 大致架构


主要对象分为两类：Handle与Request

```
/* Handle types. */
/* 生存周期较长的对象 */
typedef struct uv_loop_s uv_loop_t;
typedef struct uv_handle_s uv_handle_t;  // 所有handle对象的父类
typedef struct uv_stream_s uv_stream_t;  // tcp、tty、pipe对象的父类
typedef struct uv_tcp_s uv_tcp_t;
typedef struct uv_udp_s uv_udp_t;
typedef struct uv_pipe_s uv_pipe_t;
typedef struct uv_tty_s uv_tty_t;
typedef struct uv_poll_s uv_poll_t;
typedef struct uv_timer_s uv_timer_t;
typedef struct uv_prepare_s uv_prepare_t;
typedef struct uv_check_s uv_check_t;
typedef struct uv_idle_s uv_idle_t;
typedef struct uv_async_s uv_async_t;
typedef struct uv_process_s uv_process_t;
typedef struct uv_fs_event_s uv_fs_event_t;
typedef struct uv_fs_poll_s uv_fs_poll_t;
typedef struct uv_signal_s uv_signal_t;

/* Request types. */
typedef struct uv_req_s uv_req_t;
typedef struct uv_getaddrinfo_s uv_getaddrinfo_t;
typedef struct uv_getnameinfo_s uv_getnameinfo_t;
typedef struct uv_shutdown_s uv_shutdown_t;
typedef struct uv_write_s uv_write_t;
typedef struct uv_connect_s uv_connect_t;
typedef struct uv_udp_send_s uv_udp_send_t;
typedef struct uv_fs_s uv_fs_t;
typedef struct uv_work_s uv_work_t;
```

src/inet.c
主要实现了如下四个针对网络地址的转换函数：
static int inet_ntop4(const unsigned char *src, char *dst, size_t size);
static int inet_ntop6(const unsigned char *src, char *dst, size_t size);
static int inet_pton4(const char *src, unsigned char *dst);
static int inet_pton6(const char *src, unsigned char *dst);

src/queue.h
以宏的方式实现了浸入式的双向环形队列。多用来串连存储一些事件（handle, request）
```
#define QUEUE_DATA(ptr, type, field)                                          \
  ((type *) ((char *) (ptr) - offsetof(type, field)))\\可见其浸入式的设计

//除了常规的队列操作还实现了如下的操作。QUEUE_MOVE，用于从事件队列上摘下事件，进行处理。
#define QUEUE_SPLIT(h, q, n)                                                  \
  do {                                                                        \
    QUEUE_PREV(n) = QUEUE_PREV(h);                                            \
    QUEUE_PREV_NEXT(n) = (n);                                                 \
    QUEUE_NEXT(n) = (q);                                                      \
    QUEUE_PREV(h) = QUEUE_PREV(q);                                            \
    QUEUE_PREV_NEXT(h) = (h);                                                 \
    QUEUE_PREV(q) = (n);                                                      \
  }                                                                           \
  while (0)

#define QUEUE_MOVE(h, n)                                                      \
  do {                                                                        \
    if (QUEUE_EMPTY(h))                                                       \
      QUEUE_INIT(n);                                                          \
    else {                                                                    \
      QUEUE* q = QUEUE_HEAD(h);                                               \
      QUEUE_SPLIT(h, q, n);                                                   \
    }                                                                         \
  }                                                                           \
  while (0)


```
include/threadpool.h src/threadpool.c 
libuv内部的线程池实现，缺省是打开4个线程，用于处理一些Request，如文件操作（UV_FS），DNS操作（UV_GETADDRINFO、UV_GETNAMEINFO）及其他UV_WORK类型的操作。
```
//include/threadpool.h 
struct uv__work {
  void (*work)(struct uv__work *w);              //
  void (*done)(struct uv__work *w, int status);  //回调函数 
  struct uv_loop_s* loop;
  void* wq[2];
};

这是内部用的结构体，用户用uv_work这个request来传入loop将requset插入loop->

/src/threadpool.c 


static uv_once_t once = UV_ONCE_INIT;
static uv_cond_t cond;
static uv_mutex_t mutex;
static unsigned int idle_threads;
static unsigned int nthreads;
static uv_thread_t* threads;
static uv_thread_t default_threads[4];
static QUEUE exit_message;
static QUEUE wq;                       //任务队列
static volatile int initialized;       //init_once 的标记



//每个线程都用这个函数初始化
static void worker(void* arg) {
  struct uv__work* w;
  QUEUE* q;

  (void) arg;

  for (;;) {         //典型的条件变量写法，固定个数的线程等在任务队列上
    uv_mutex_lock(&mutex);

    while (QUEUE_EMPTY(&wq)) {
      idle_threads += 1;
      uv_cond_wait(&cond, &mutex);
      idle_threads -= 1;
    }

    q = QUEUE_HEAD(&wq);

    if (q == &exit_message)
      uv_cond_signal(&cond);
    else {
      QUEUE_REMOVE(q);
      QUEUE_INIT(q);  /* Signal uv_cancel() that the work req is
                             executing. */
    }

    uv_mutex_unlock(&mutex);

    if (q == &exit_message)
      break;

    w = QUEUE_DATA(q, struct uv__work, wq);
    w->work(w);

    uv_mutex_lock(&w->loop->wq_mutex);
    w->work = NULL;  /* Signal uv_cancel() that the work req is done
                        executing. */
    QUEUE_INSERT_TAIL(&w->loop->wq, &w->wq);
    uv_async_send(&w->loop->wq_async);
    uv_mutex_unlock(&w->loop->wq_mutex);
  }
}

//发布任务，直接插入到任务队列wq后面
static void post(QUEUE* q) {
  uv_mutex_lock(&mutex);
  QUEUE_INSERT_TAIL(&wq, q);
  if (idle_threads > 0)
    uv_cond_signal(&cond);
  uv_mutex_unlock(&mutex);
}

//只调用一次，完成线程池的初始化（条件变量、互斥锁、线程数组）。
static void init_once(void) {
  unsigned int i;
  const char* val;

  nthreads = ARRAY_SIZE(default_threads);
  val = getenv("UV_THREADPOOL_SIZE");
  if (val != NULL)
    nthreads = atoi(val);
  if (nthreads == 0)
    nthreads = 1;
  if (nthreads > MAX_THREADPOOL_SIZE)
    nthreads = MAX_THREADPOOL_SIZE;

  threads = default_threads;
  if (nthreads > ARRAY_SIZE(default_threads)) {
    threads = uv__malloc(nthreads * sizeof(threads[0]));
    if (threads == NULL) {
      nthreads = ARRAY_SIZE(default_threads);
      threads = default_threads;
    }
  }

  if (uv_cond_init(&cond))
    abort();

  if (uv_mutex_init(&mutex))
    abort();

  QUEUE_INIT(&wq);

  for (i = 0; i < nthreads; i++)
    if (uv_thread_create(threads + i, worker, NULL))
      abort();

  initialized = 1;
}
```

/src/heap-inl.h
最小堆实现


/src/unix/fs.c
针对unix系统的文件操作实现，通过uv__fs_work传递给线程池来处理。将其赋给struct worker中的字段。
```
static void uv__fs_work(struct uv__work* w) {
  int retry_on_eintr;
  uv_fs_t* req;
  ssize_t r;

  req = container_of(w, uv_fs_t, work_req);
  retry_on_eintr = !(req->fs_type == UV_FS_CLOSE);

  do {
    errno = 0;

#define X(type, action)                                                       \
  case UV_FS_ ## type:                                                        \
    r = action;                                                               \
    break;

    switch (req->fs_type) {
    X(ACCESS, access(req->path, req->flags));
    X(CHMOD, chmod(req->path, req->mode));
    X(CHOWN, chown(req->path, req->uid, req->gid));
    X(CLOSE, close(req->file));
    X(FCHMOD, fchmod(req->file, req->mode));
    X(FCHOWN, fchown(req->file, req->uid, req->gid));
    X(FDATASYNC, uv__fs_fdatasync(req));
    X(FSTAT, uv__fs_fstat(req->file, &req->statbuf));
    X(FSYNC, fsync(req->file));
    X(FTRUNCATE, ftruncate(req->file, req->off));
    X(FUTIME, uv__fs_futime(req));
    X(LSTAT, uv__fs_lstat(req->path, &req->statbuf));
    X(LINK, link(req->path, req->new_path));
    X(MKDIR, mkdir(req->path, req->mode));
    X(MKDTEMP, uv__fs_mkdtemp(req));
    X(OPEN, uv__fs_open(req));
    X(READ, uv__fs_buf_iter(req, uv__fs_read));
    X(SCANDIR, uv__fs_scandir(req));
    X(READLINK, uv__fs_readlink(req));
    X(REALPATH, uv__fs_realpath(req));
    X(RENAME, rename(req->path, req->new_path));
    X(RMDIR, rmdir(req->path));
    X(SENDFILE, uv__fs_sendfile(req));
    X(STAT, uv__fs_stat(req->path, &req->statbuf));
    X(SYMLINK, symlink(req->path, req->new_path));
    X(UNLINK, unlink(req->path));
    X(UTIME, uv__fs_utime(req));
    X(WRITE, uv__fs_buf_iter(req, uv__fs_write));
    default: abort();
    }
#undef X
  } while (r == -1 && errno == EINTR && retry_on_eintr);

  if (r == -1)
    req->result = -errno;
  else
    req->result = r;

  if (r == 0 && (req->fs_type == UV_FS_STAT ||
                 req->fs_type == UV_FS_FSTAT ||
                 req->fs_type == UV_FS_LSTAT)) {
    req->ptr = &req->statbuf;
  }
}
```
uv__fs_xxx 是对文件io系统api的封装，如果系统不提供相关api，libuv还会自己实现，如用普通api实现了sendfile。

至于用户如何用该组api呢，libuv又进行了进一步封装,以chmod为例：
```

int uv_fs_chmod(uv_loop_t* loop,
                uv_fs_t* req,
                const char* path,
                int mode,
                uv_fs_cb cb) {
  INIT(CHMOD);                        //以UV_FS_CHMOD的type初始化fs request
  PATH;
  req->mode = mode;
  POST;                               //根据是否有cb函数，将request插入到线程池队列或直接运行uv__fs_work
}

#define INIT(subtype)                                                         \
  do {                                                                        \
    req->type = UV_FS;                                                        \
    if (cb != NULL)                                                           \
      uv__req_init(loop, req, UV_FS);                                         \
    req->fs_type = UV_FS_ ## subtype;                                         \
    req->result = 0;                                                          \
    req->ptr = NULL;                                                          \
    req->loop = loop;                                                         \
    req->path = NULL;                                                         \
    req->new_path = NULL;                                                     \
    req->cb = cb;                                                             \
  }                                                                           \
  while (0)

//只带一个路径信息的fs操作
#define PATH                                                                  \
  do {                                                                        \
    assert(path != NULL);                                                     \
    if (cb == NULL) {                                                         \
      req->path = path;                                                       \
    } else {                                                                  \
      req->path = uv__strdup(path);                                           \
      if (req->path == NULL) {                                                \
        uv__req_unregister(loop, req);                                        \
        return -ENOMEM;                                                       \
      }                                                                       \
    }                                                                         \
  }                                                                           \
  while (0)
//带两个路径信息的fs操作
#define PATH2                                                                 \
  do {                                                                        \
    if (cb == NULL) {                                                         \
      req->path = path;                                                       \
      req->new_path = new_path;                                               \
    } else {                                                                  \
      size_t path_len;                                                        \
      size_t new_path_len;                                                    \
      path_len = strlen(path) + 1;                                            \
      new_path_len = strlen(new_path) + 1;                                    \
      req->path = uv__malloc(path_len + new_path_len);                        \
      if (req->path == NULL) {                                                \
        uv__req_unregister(loop, req);                                        \
        return -ENOMEM;                                                       \
      }                                                                       \
      req->new_path = req->path + path_len;                                   \
      memcpy((void*) req->path, path, path_len);                              \
      memcpy((void*) req->new_path, new_path, new_path_len);                  \
    }                                                                         \
  }                                                                           \
  while (0)

#define POST                                                                  \
  do {                                                                        \
    if (cb != NULL) {                                                         \
      uv__work_submit(loop, &req->work_req, uv__fs_work, uv__fs_done);        \
      return 0;                                                               \
    }                                                                         \
    else {                                                                    \
      uv__fs_work(&req->work_req);                                            \
      return req->result;                                                     \
    }                                                                         \
  }                                                                           \
  while (0)


```


/src/unix/core.c
loop运行核心：
```
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int ran_pending;

  r = uv__loop_alive(loop);
  if (!r)
    uv__update_time(loop);

  while (r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop);                 //循环开始时更新时间
    uv__run_timers(loop);                  //运行注册的定时器
    ran_pending = uv__run_pending(loop);   //运行loop->pending_queue中的回调函数
    uv__run_idle(loop);                    //运行注册的idle任务
    uv__run_prepare(loop);                 //运行注册的prepare任务  （这里idle、prepare与check实现方式其实没有差别，由loop-watcher.c中的同一套宏生成）

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);

    uv__io_poll(loop, timeout);            //io多路复用不同的系统调用不同的底层api实现
    uv__run_check(loop);
    uv__run_closing_handles(loop);         //对于loop->close_handles链表上的各handle将其unref、从loop->handle_queue中移除最后调用handle->close_cb回调函数

    if (mode == UV_RUN_ONCE) {
      /* UV_RUN_ONCE implies forward progress: at least one callback must have
       * been invoked when it returns. uv__io_poll() can return without doing
       * I/O (meaning: no callbacks) when its timeout expires - which means we
       * have pending timers that satisfy the forward progress constraint.
       *
       * UV_RUN_NOWAIT makes no guarantees about progress so it's omitted from
       * the check.
       */
      uv__update_time(loop);
      uv__run_timers(loop);
    }

    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }

  /* The if statement lets gcc compile it to a conditional store. Avoids
   * dirtying a cache line.
   */
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
}

```

各handle close的方式： 

```
1. 通过调用，handle->flag中set UV_CLOSING位，关闭相关资源，并将handle加到loop->closeing_handles链表头上
void uv_close(uv_handle_t* handle, uv_close_cb close_cb) {
  assert(!(handle->flags & (UV_CLOSING | UV_CLOSED)));

  handle->flags |= UV_CLOSING;
  handle->close_cb = close_cb;

  switch (handle->type) {
  case UV_NAMED_PIPE:
    uv__pipe_close((uv_pipe_t*)handle);
    break;

  case UV_TTY:
    uv__stream_close((uv_stream_t*)handle);
    break;

  case UV_TCP:
    uv__tcp_close((uv_tcp_t*)handle);
    break;

  case UV_UDP:
    uv__udp_close((uv_udp_t*)handle);
    break;

  case UV_PREPARE:
    uv__prepare_close((uv_prepare_t*)handle);
    break;

  case UV_CHECK:
    uv__check_close((uv_check_t*)handle);
    break;

  case UV_IDLE:
    uv__idle_close((uv_idle_t*)handle);
    break;

  case UV_ASYNC:
    uv__async_close((uv_async_t*)handle);
    break;

  case UV_TIMER:
    uv__timer_close((uv_timer_t*)handle);
    break;

  case UV_PROCESS:
    uv__process_close((uv_process_t*)handle);
    break;

  case UV_FS_EVENT:
    uv__fs_event_close((uv_fs_event_t*)handle);
    break;

  case UV_POLL:
    uv__poll_close((uv_poll_t*)handle);
    break;

  case UV_FS_POLL:
    uv__fs_poll_close((uv_fs_poll_t*)handle);
    break;

  case UV_SIGNAL:
    uv__signal_close((uv_signal_t*) handle);
    /* Signal handles may not be closed immediately. The signal code will */
    /* itself close uv__make_close_pending whenever appropriate. */
    return;

  default:
    assert(0);
  }

  uv__make_close_pending(handle);
}

void uv__make_close_pending(uv_handle_t* handle) {
  assert(handle->flags & UV_CLOSING);
  assert(!(handle->flags & UV_CLOSED));
  handle->next_closing = handle->loop->closing_handles;
  handle->loop->closing_handles = handle;
}

2. loop循环中通过uv__run_closing_handles(loop)。对于loop->close_handles链表上的各handle将其unref、从loop->handle_queue中移除最后调用handle->close_cb回调函数
我们可以看到其close_cb是一起调用的不知道这里是否有运行效率的考虑。

static void uv__finish_close(uv_handle_t* handle) {
  /* Note: while the handle is in the UV_CLOSING state now, it's still possible
   * for it to be active in the sense that uv__is_active() returns true.
   * A good example is when the user calls uv_shutdown(), immediately followed
   * by uv_close(). The handle is considered active at this point because the
   * completion of the shutdown req is still pending.
   */
  assert(handle->flags & UV_CLOSING);
  assert(!(handle->flags & UV_CLOSED));
  handle->flags |= UV_CLOSED;

  switch (handle->type) {
    case UV_PREPARE:
    case UV_CHECK:
    case UV_IDLE:
    case UV_ASYNC:
    case UV_TIMER:
    case UV_PROCESS:
    case UV_FS_EVENT:
    case UV_FS_POLL:
    case UV_POLL:
    case UV_SIGNAL:
      break;

    case UV_NAMED_PIPE:
    case UV_TCP:
    case UV_TTY:
      uv__stream_destroy((uv_stream_t*)handle);
      break;

    case UV_UDP:
      uv__udp_finish_close((uv_udp_t*)handle);
      break;

    default:
      assert(0);
      break;
  }

  uv__handle_unref(handle);
  QUEUE_REMOVE(&handle->handle_queue);

  if (handle->close_cb) {
    handle->close_cb(handle);
  }
}

```

/src/unix/linux-core.c
看一下封装了epoll的loop

```

void uv__io_poll(uv_loop_t* loop, int timeout) {
  /* A bug in kernels < 2.6.37 makes timeouts larger than ~30 minutes
   * effectively infinite on 32 bits architectures.  To avoid blocking
   * indefinitely, we cap the timeout and poll again if necessary.
   *
   * Note that "30 minutes" is a simplification because it depends on
   * the value of CONFIG_HZ.  The magic constant assumes CONFIG_HZ=1200,
   * that being the largest value I have seen in the wild (and only once.)
   */
  static const int max_safe_timeout = 1789569;
  static int no_epoll_pwait;
  static int no_epoll_wait;
  struct uv__epoll_event events[1024];
  struct uv__epoll_event* pe;
  struct uv__epoll_event e;
  int real_timeout;
  QUEUE* q;
  uv__io_t* w;
  sigset_t sigset;
  uint64_t sigmask;
  uint64_t base;
  int have_signals;
  int nevents;
  int count;
  int nfds;
  int fd;
  int op;
  int i;

  if (loop->nfds == 0) {
    assert(QUEUE_EMPTY(&loop->watcher_queue));
    return;
  }

  while (!QUEUE_EMPTY(&loop->watcher_queue)) {
    q = QUEUE_HEAD(&loop->watcher_queue);
    QUEUE_REMOVE(q);
    QUEUE_INIT(q);

    w = QUEUE_DATA(q, uv__io_t, watcher_queue);
    assert(w->pevents != 0);
    assert(w->fd >= 0);
    assert(w->fd < (int) loop->nwatchers);

    e.events = w->pevents;
    e.data = w->fd;

    if (w->events == 0)
      op = UV__EPOLL_CTL_ADD;
    else
      op = UV__EPOLL_CTL_MOD;

    /* XXX Future optimization: do EPOLL_CTL_MOD lazily if we stop watching
     * events, skip the syscall and squelch the events after epoll_wait().
     */
    if (uv__epoll_ctl(loop->backend_fd, op, w->fd, &e)) {
      if (errno != EEXIST)
        abort();

      assert(op == UV__EPOLL_CTL_ADD);

      /* We've reactivated a file descriptor that's been watched before. */
      if (uv__epoll_ctl(loop->backend_fd, UV__EPOLL_CTL_MOD, w->fd, &e))
        abort();
    }

    w->events = w->pevents;
  }

  sigmask = 0;
  if (loop->flags & UV_LOOP_BLOCK_SIGPROF) {
    sigemptyset(&sigset);
    sigaddset(&sigset, SIGPROF);
    sigmask |= 1 << (SIGPROF - 1);
  }

  assert(timeout >= -1);
  base = loop->time;
  count = 48; /* Benchmarks suggest this gives the best throughput. */
  real_timeout = timeout;

  for (;;) {
    /* See the comment for max_safe_timeout for an explanation of why
     * this is necessary.  Executive summary: kernel bug workaround.
     */
    if (sizeof(int32_t) == sizeof(long) && timeout >= max_safe_timeout)
      timeout = max_safe_timeout;

    if (sigmask != 0 && no_epoll_pwait != 0)
      if (pthread_sigmask(SIG_BLOCK, &sigset, NULL))
        abort();

    if (no_epoll_wait != 0 || (sigmask != 0 && no_epoll_pwait == 0)) {
      nfds = uv__epoll_pwait(loop->backend_fd,
                             events,
                             ARRAY_SIZE(events),
                             timeout,
                             sigmask);
      if (nfds == -1 && errno == ENOSYS)
        no_epoll_pwait = 1;
    } else {
      nfds = uv__epoll_wait(loop->backend_fd,
                            events,
                            ARRAY_SIZE(events),
                            timeout);
      if (nfds == -1 && errno == ENOSYS)
        no_epoll_wait = 1;
    }

    if (sigmask != 0 && no_epoll_pwait != 0)
      if (pthread_sigmask(SIG_UNBLOCK, &sigset, NULL))
        abort();

    /* Update loop->time unconditionally. It's tempting to skip the update when
     * timeout == 0 (i.e. non-blocking poll) but there is no guarantee that the
     * operating system didn't reschedule our process while in the syscall.
     */
    SAVE_ERRNO(uv__update_time(loop));

    if (nfds == 0) {
      assert(timeout != -1);

      timeout = real_timeout - timeout;
      if (timeout > 0)
        continue;

      return;
    }

    if (nfds == -1) {
      if (errno == ENOSYS) {
        /* epoll_wait() or epoll_pwait() failed, try the other system call. */
        assert(no_epoll_wait == 0 || no_epoll_pwait == 0);
        continue;
      }

      if (errno != EINTR)
        abort();

      if (timeout == -1)
        continue;

      if (timeout == 0)
        return;

      /* Interrupted by a signal. Update timeout and poll again. */
      goto update_timeout;
    }

    have_signals = 0;
    nevents = 0;

    assert(loop->watchers != NULL);
    loop->watchers[loop->nwatchers] = (void*) events;
    loop->watchers[loop->nwatchers + 1] = (void*) (uintptr_t) nfds;
    for (i = 0; i < nfds; i++) {
      pe = events + i;
      fd = pe->data;

      /* Skip invalidated events, see uv__platform_invalidate_fd */
      if (fd == -1)
        continue;

      assert(fd >= 0);
      assert((unsigned) fd < loop->nwatchers);

      w = loop->watchers[fd];

      if (w == NULL) {
        /* File descriptor that we've stopped watching, disarm it.
         *
         * Ignore all errors because we may be racing with another thread
         * when the file descriptor is closed.
         */
        uv__epoll_ctl(loop->backend_fd, UV__EPOLL_CTL_DEL, fd, pe);
        continue;
      }

      /* Give users only events they're interested in. Prevents spurious
       * callbacks when previous callback invocation in this loop has stopped
       * the current watcher. Also, filters out events that users has not
       * requested us to watch.
       */
      pe->events &= w->pevents | POLLERR | POLLHUP;

      /* Work around an epoll quirk where it sometimes reports just the
       * EPOLLERR or EPOLLHUP event.  In order to force the event loop to
       * move forward, we merge in the read/write events that the watcher
       * is interested in; uv__read() and uv__write() will then deal with
       * the error or hangup in the usual fashion.
       *
       * Note to self: happens when epoll reports EPOLLIN|EPOLLHUP, the user
       * reads the available data, calls uv_read_stop(), then sometime later
       * calls uv_read_start() again.  By then, libuv has forgotten about the
       * hangup and the kernel won't report EPOLLIN again because there's
       * nothing left to read.  If anything, libuv is to blame here.  The
       * current hack is just a quick bandaid; to properly fix it, libuv
       * needs to remember the error/hangup event.  We should get that for
       * free when we switch over to edge-triggered I/O.
       */
      if (pe->events == POLLERR || pe->events == POLLHUP)
        pe->events |= w->pevents & (POLLIN | POLLOUT);

      if (pe->events != 0) {
        /* Run signal watchers last.  This also affects child process watchers
         * because those are implemented in terms of signal watchers.
         */
        if (w == &loop->signal_io_watcher)
          have_signals = 1;
        else
          w->cb(loop, w, pe->events);

        nevents++;
      }
    }

    if (have_signals != 0)
      loop->signal_io_watcher.cb(loop, &loop->signal_io_watcher, POLLIN);

    loop->watchers[loop->nwatchers] = NULL;
    loop->watchers[loop->nwatchers + 1] = NULL;

    if (have_signals != 0)
      return;  /* Event loop should cycle now so don't poll again. */

    if (nevents != 0) {
      if (nfds == ARRAY_SIZE(events) && --count != 0) {
        /* Poll for more events but don't block this time. */
        timeout = 0;
        continue;
      }
      return;
    }

    if (timeout == 0)
      return;

    if (timeout == -1)
      continue;

update_timeout:
    assert(timeout > 0);

    real_timeout -= (loop->time - base);
    if (real_timeout <= 0)
      return;

    timeout = real_timeout;
  }
}


uint64_t uv__hrtime(uv_clocktype_t type) {
  static clock_t fast_clock_id = -1;
  struct timespec t;
  clock_t clock_id;

  /* Prefer CLOCK_MONOTONIC_COARSE if available but only when it has
   * millisecond granularity or better.  CLOCK_MONOTONIC_COARSE is
   * serviced entirely from the vDSO, whereas CLOCK_MONOTONIC may
   * decide to make a costly system call.
   */
  /* TODO(bnoordhuis) Use CLOCK_MONOTONIC_COARSE for UV_CLOCK_PRECISE
   * when it has microsecond granularity or better (unlikely).
   */
  if (type == UV_CLOCK_FAST && fast_clock_id == -1) {
    if (clock_getres(CLOCK_MONOTONIC_COARSE, &t) == 0 &&
        t.tv_nsec <= 1 * 1000 * 1000) {
      fast_clock_id = CLOCK_MONOTONIC_COARSE;
    } else {
      fast_clock_id = CLOCK_MONOTONIC;
    }
  }

  clock_id = CLOCK_MONOTONIC;
  if (type == UV_CLOCK_FAST)
    clock_id = fast_clock_id;

  if (clock_gettime(clock_id, &t))
    return 0;  /* Not really possible. */

  return t.tv_sec * (uint64_t) 1e9 + t.tv_nsec;
}

```