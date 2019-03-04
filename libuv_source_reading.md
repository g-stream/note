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

//src/threadpool.c 


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
