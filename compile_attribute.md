# gcc attributes aiding optimizations
- Constant Detection

        int __builtin_constant_p(exp)

- Hints for Branch Rrediction
    
    __builtin_expect

        #define likely(x)   __builtin_expect(!!(x), 1)
        #define unlikely(x) __builtin_expect(!!(x), 0)

- Prefetching

     __builtin_prefetch

- Align Data

        __attribute__((aligned (val)));

- Packing Data

        __attribute__((packed, aligned(val)))
- Pure Functions
    
    + Value based on parameters and global memory only
    + strlen()
    +            int __attribute__((pure)) static_pure_function([...])

可以看看ecb.h或libcork等库中的compile attribute部分，它们对一些常用的attribute做了封装。