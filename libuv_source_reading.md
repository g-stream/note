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
