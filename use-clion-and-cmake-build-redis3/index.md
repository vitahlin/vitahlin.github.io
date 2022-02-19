# macOS上用Clion CMake编译redis 3.0



## 相关版本说明

- **redis：3.0.0**
- **macOS：11.2.3**
- **CMake：3.19.6**
- **Clion：2020.3.2**

## 克隆redis3.0源代码

去Github上面下载3.0版本的代码：[https://codeload.github.com/redis/redis/zip/3.0](https://codeload.github.com/redis/redis/zip/3.0)
解压后将目录`redis-3.0`重新命名为`redis` 。

## 创建对应的CMakeLists.txt

使用`cmake` 编译`redis` 源码的话需要创建对应的`CMakeLists.txt` 文件，创建完成后，直接在`redis` 目录执行命令`cmake .` 即可。

### redis/deps/hiredis/CMakeLists.txt

```c
add_library(hiredis STATIC
        hiredis.c net.c dict.c net.c sds.c async.c
        )
```

### redis/deps/linenoise/CMakeLists.txt

```c
add_library(linenoise linenoise.c)
```

### redis/deps/lua/CMakeLists.txt

```c
set(LUA_SRC
        src/lapi.c src/lcode.c src/ldebug.c src/ldo.c src/ldump.c src/lfunc.c
        src/lgc.c src/llex.c src/lmem.c src/lobject.c src/lopcodes.c src/lparser.c
        src/lstate.c src/lstring.c src/ltable.c src/ltm.c src/lundump.c src/lvm.c
        src/lzio.c src/strbuf.c src/fpconv.c src/lauxlib.c src/lbaselib.c src/ldblib.c
        src/liolib.c src/lmathlib.c src/loslib.c src/ltablib.c src/lstrlib.c src/loadlib.c
        src/linit.c src/lua_cjson.c src/lua_struct.c src/lua_cmsgpack.c src/lua_bit.c
        )
add_library(lua STATIC ${LUA_SRC})
```

### redis/deps/CMakeLists.txt

```c
add_subdirectory(hiredis)
add_subdirectory(linenoise)
add_subdirectory(lua)
```

### redis/CMakeLists.txt

```c
cmake_minimum_required(VERSION 3.0 FATAL_ERROR)

project(redis3.0 VERSION 3.0)

set(CMAKE_BUILD_TYPE "Debug")

get_filename_component(REDIS_ROOT "${CMAKE_CURRENT_SOURCE_DIR}" ABSOLUTE)

add_subdirectory(deps)

set(SRC_SERVER src/redis.c
        ## 解决mac下编译报错问题
        ## src/ae_epoll.c src/ae_evport.c src/ae_select.c src/ae_kqueue.c
        src/adlist.c src/ae.c src/anet.c src/aof.c src/asciilogo.h src/bio.c src/bitops.c src/blocked.c
        src/cluster.c src/config.c src/crc64.c src/crc16.c src/db.c src/debug.c src/dict.c src/endianconv.c
        src/hyperloglog.c src/intset.c src/latency.c src/lzf_c.c src/lzf_d.c src/memtest.c src/multi.c
        src/networking.c src/notify.c src/object.c src/pqsort.c src/pubsub.c src/rand.c src/rdb.c src/release.c
        src/replication.c src/rio.c src/scripting.c src/sds.c src/sentinel.c src/setproctitle.c src/sha1.c
        src/slowlog.c src/sort.c src/sparkline.c src/syncio.c src/t_hash.c src/t_list.c src/t_set.c src/t_string.c
        src/t_zset.c src/util.c src/ziplist.c src/zipmap.c src/zmalloc.c
        )

set(EXECUTABLE_OUTPUT_PATH src)
link_directories(deps/linenoise deps/lua/src deps/hiredis)
add_executable(redis-server ${SRC_SERVER})

target_include_directories(redis-server
        PRIVATE ${REDIS_ROOT}/deps/linenoise
        PRIVATE ${REDIS_ROOT}/deps/hiredis
        PRIVATE ${REDIS_ROOT}/deps/lua/src
        )
target_link_libraries(redis-server
        PRIVATE pthread
        PRIVATE dl
        PRIVATE m
        PRIVATE lua
        PRIVATE linenoise
        PRIVATE hiredis
        )

set(CLIENT_SRC src/redis-cli.c
        src/anet.c src/sds.c src/adlist.c src/zmalloc.c
        src/release.c src/anet.c src/ae.c src/crc64.c
        )
add_executable(redis-cli ${CLIENT_SRC})

target_include_directories(redis-cli
        PRIVATE ${REDIS_ROOT}/deps/linenoise
        PRIVATE ${REDIS_ROOT}/deps/hiredis
        PRIVATE ${REDIS_ROOT}/deps/lua/src
        )
target_link_libraries(redis-cli
        PRIVATE pthread
        PRIVATE m
        PRIVATE linenoise
        PRIVATE hiredis
        )
```

## 错误处理

### fatal error: 'sys/epoll.h' file not found

在macOS下编译会出现这个错误，是因为epoll是linux独有的，macOS上并没有`sys/epoll.h`头文件，因此编译出错。但macOS上使用`kqueue`代替了`epoll`，这里可以直接注释相关代码。

redis/CMakeLists.txt中注释下列文件：

```bash
##        src/ae_epoll.c
##        src/ae_evport.c
##        src/ae_select.c
##        src/ae_kqueue.c
```

### fatal error: 'release.h' file not found

在redis目录下执行脚本即可，用于生成缺失的 `release.h` 文件。

```shell
cd src/
sh mkreleasehdr.sh
```

## Clion导入项目

直接打开项目，选择 **Open as：CMake project**，打开后就可以直接运行了， `redis-serve` 即redis服务端， `redis-cli` 即客户端。

