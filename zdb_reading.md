# libzdb
[libzdb](https://github.com/asmuth/zdb) is a lightweight embedded database:



    Data is organized into tables and rows; tables have a strict schema
    Column-oriented internal storage layout for efficient compression and I/O
    Allows reading data without copying it (zero-copy)
    Supports non-blocking I/O via io_submit/aio
    Implemented as a lightweight C/C++ library
    Safe to use from multiple threads and processes

```
../zdb
├── bindings
├── build
├── CHANGELOG
├── CMakeLists.txt
├── core
    ├── cursor.cc
    ├── cursor.h
    ├── database.cc
    ├── database.h
    ├── lock.cc
    ├── lock.h
    ├── metadata.cc
    ├── metadata.h
    ├── op_commit.cc
    ├── op_insert.cc
    ├── op_load.cc
    ├── op_meta.cc
    ├── page_buffer.cc
    ├── page_buffer.h
    ├── page.cc
    ├── page.h
    ├── page_index.cc
    ├── page_index.h
    ├── page_map.cc
    ├── page_map.h
    ├── table.h
    ├── tsdb.cc
    ├── tsdb.h
    ├── tuple.h
    ├── util
    ├── varint.cc
    ├── varint.h
    └── zdb.h
├── examples
├── LICENSE
├── README.md
├── test
├── tools
```
# core/zdb.h
定义了db的操作。
- cursor_advise_t指针的操作类型？？？？
- enum 类型：zdb_type_t，zdb_err_t
    
    可见db支持的类型为如下几种：
    ```
    typedef enum {
    ZDB_BOOL = 1,
    ZDB_UINT32 = 2,
    ZDB_UINT64 = 3,
    ZDB_INT32 = 4,
    ZDB_INT64 = 5,
    ZDB_FLOAT32 = 6,
    ZDB_FLOAT64 = 7,
    ZDB_STRING = 8
    } zdb_type_t;
    ```
- const int ZDB_OPEN_XXX 打开方式的flag？
- db操作
    - open, close, commit
    - table add\delete
    - column add\delete\id
    - put 表中加tuple
    - 各种lookup_int64/float32.....
    - tuple alloc\free和各种add_int32/float32....
    - tuple 各种get
    - cursor_init/close
    - cursor_use/next/tell/advise
    - 各种cursor_get
    - 各种cursor_seek
针对c语言，用
```
#ifdef __cplusplus
extern "C" {
#endif


#ifdef __cplusplus
}
#endif
```
包裹了接口

# core/varint.h, core/varint.c
读写变长整形的函数。
# core/tuple
zdb_tuple 类定义了对tuple add和get各种类型的方法
# core/lock
包装了pthread的读写锁
# core/
# core/
# core/
# core/
# core/
# core/
# core/
# core/
# core/