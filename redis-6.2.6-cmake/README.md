## 背景

- redis 项目使用 Makefile 编译
- 旧版本 clion(2018.3)使用 cmake 构建 c 项目, 导入 redis 源无法直接 debug, 可升级 clion 版本或者添加 CMakeLists.txt 文件, 使用 cmake 构建项目, 就可以愉快的 debug 啦
- 新版本 clion(2021.3)已经同时支持 Makefile 和 cmake, 无需 CMakeLists.txt 即可 debug, 后面的内容可跳过

## 环境

- 操作系统: macOS 或者 ubuntu
- gcc, cmake
- redis 源码(https://download.redis.io/releases/redis-6.2.6.tar.gz)

## 转换项目为 cmake

1. deps/hdr_histogram 添加 CMakeLists.txt

```cmake
add_library(hdr_histogram hdr_alloc.c hdr_histogram.c)
```

2. deps/linenoise 添加 CMakeLists.txt

```cmake
add_library(linenoise linenoise.c)
```

3. deps/lua 添加 CMakeLists.txt

```cmake
set(LUA_SRC
        src/lapi.c src/lcode.c src/ldebug.c src/ldo.c src/ldump.c src/lfunc.c
        src/lgc.c src/llex.c src/lmem.c
        src/lobject.c src/lopcodes.c src/lparser.c src/lstate.c src/lstring.c
        src/ltable.c src/ltm.c
        src/lundump.c src/lvm.c src/lzio.c src/strbuf.c src/fpconv.c
        src/lauxlib.c src/lbaselib.c src/ldblib.c src/liolib.c src/lmathlib.c
        src/loslib.c src/ltablib.c
        src/lstrlib.c src/loadlib.c src/linit.c src/lua_cjson.c
        src/lua_struct.c
        src/lua_cmsgpack.c
        src/lua_bit.c
        )
add_library(lua STATIC ${LUA_SRC})
```

4. deps/ 添加 CMakeLists.txt

```cmake
add_subdirectory(hdr_histogram)
add_subdirectory(hiredis)
add_subdirectory(linenoise)
add_subdirectory(lua)
```

5. 根目录添加 CMakeLists.txt

```cmake
# cmake最低版本要求
cmake_minimum_required(VERSION 3.13)

#项目信息
project(redis6.2.6_debug)

#debug模式编译
set(CMAKE_BUILD_TYPE Debug CACHE STRING "set build type to debug")

# O2 第二级别优化
## -Wall：开启警告提示
# -std=c99：C99标准
add_definitions(
       "-O2"
       "-Wall"
       "-std=c99"
       "-pedantic"
)

#GCC编译选项（设置CMAKE_C_FLAGS变量）
# set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O2")
# # -std=c++11, -std=c99
# set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c99 -pedantic -DREDIS_STATIC=''")
# set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -W -Wno-missing-field-initializers")

#可执行文件的输出目录
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/)

#安装路径
set(CMAKE_INSTALL_PREFIX ${PROJECT_SOURCE_DIR}/target/)

#依赖库目录
#Redis源码编译需要依赖的自定义库的路径，在主目录deps下
set(DEPS_PATH ${CMAKE_CURRENT_SOURCE_DIR}/deps)

#依赖共享库, 根据原Rerdis中的Makefile，编译源码需要依赖的系统共享库
set(SHARED_LIBS -lm -ldl -lpthread)

#在编译Redis之前需要执行src目录下的 mkreleasehdr.sh 脚本生成 release.h
execute_process(COMMAND sh ${CMAKE_CURRENT_SOURCE_DIR}/src/mkreleasehdr.sh WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/src)

#include directories
include_directories(
        ${DEPS_PATH}/hdr_histogram
        ${DEPS_PATH}/hiredis
        ${DEPS_PATH}/lua/src
        ${DEPS_PATH}/linenoise
)

# 添加需要编译子目录
add_subdirectory(deps)

link_directories(${DEPS_PATH}/hdr_histogram)
link_directories(${DEPS_PATH}/hiredis)
link_directories(${DEPS_PATH}/lua/src)
link_directories(${DEPS_PATH}/linenoise)

# redis-server 来源src/Makefile的REDIS_SERVER_OBJ 修改.o为.c
set(REDIS_SERVER_LIST src/adlist.c src/quicklist.c src/ae.c src/anet.c src/dict.c src/server.c src/sds.c src/zmalloc.c src/lzf_c.c src/lzf_d.c src/pqsort.c src/zipmap.c src/sha1.c src/ziplist.c src/release.c src/networking.c src/util.c src/object.c src/db.c src/replication.c src/rdb.c src/t_string.c src/t_list.c src/t_set.c src/t_zset.c src/t_hash.c src/config.c src/aof.c src/pubsub.c src/multi.c src/debug.c src/sort.c src/intset.c src/syncio.c src/cluster.c src/crc16.c src/endianconv.c src/slowlog.c src/scripting.c src/bio.c src/rio.c src/rand.c src/memtest.c src/crcspeed.c src/crc64.c src/bitops.c src/sentinel.c src/notify.c src/setproctitle.c src/blocked.c src/hyperloglog.c src/latency.c src/sparkline.c src/redis-check-rdb.c src/redis-check-aof.c src/geo.c src/lazyfree.c src/module.c src/evict.c src/expire.c src/geohash.c src/geohash_helper.c src/childinfo.c src/defrag.c src/siphash.c src/rax.c src/t_stream.c src/listpack.c src/localtime.c src/lolwut.c src/lolwut5.c src/lolwut6.c src/acl.c src/gopher.c src/tracking.c src/connection.c src/tls.c src/sha256.c src/timeout.c src/setcpuaffinity.c src/monotonic.c src/mt19937-64.c)

#redis-cli
set(REDIS_CLI_LIST src/anet.c src/adlist.c src/dict.c src/redis-cli.c src/zmalloc.c src/release.c src/ae.c src/crcspeed.c src/crc64.c src/siphash.c src/crc16.c src/monotonic.c src/cli_common.c src/mt19937-64.c)

# redis-benchmark
set(REDIS_BENCHMARK_LIST src/ae.c src/anet.c src/redis-benchmark.c src/adlist.c src/dict.c src/zmalloc.c src/release.c src/crcspeed.c src/crc64.c src/siphash.c src/crc16.c src/monotonic.c src/cli_common.c src/mt19937-64.c)


#生成可执行文件
add_executable(redis-server ${REDIS_SERVER_LIST})
add_executable(redis-benchmark ${REDIS_BENCHMARK_LIST})
add_executable(redis-cli ${REDIS_CLI_LIST})

# 将目标文件与库文件进行链接
target_link_libraries(redis-server hdr_histogram hiredis lua ${SHARED_LIBS})
target_link_libraries(redis-benchmark hdr_histogram hiredis ${SHARED_LIBS})
target_link_libraries(redis-cli hdr_histogram linenoise hiredis ${SHARED_LIBS})

#add_custom_command不直接执行，target依赖成立后执行
ADD_CUSTOM_COMMAND(
        TARGET redis-server
        COMMAND cp ${CMAKE_CURRENT_SOURCE_DIR}/redis-server  ${CMAKE_CURRENT_SOURCE_DIR}/redis-sentinel
        COMMAND cp ${CMAKE_CURRENT_SOURCE_DIR}/redis-server  ${CMAKE_CURRENT_SOURCE_DIR}/redis-check-rdb
        COMMAND cp ${CMAKE_CURRENT_SOURCE_DIR}/redis-server  ${CMAKE_CURRENT_SOURCE_DIR}/redis-check-aof
)

#执行make install执行以下安装程序
install(TARGETS redis-server DESTINATION bin)
install(TARGETS redis-benchmark DESTINATION bin)
install(TARGETS redis-cli DESTINATION bin)
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/redis-server DESTINATION bin RENAME redis-check-rdb)
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/redis-server DESTINATION bin RENAME redis-check-aof)
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/redis-server DESTINATION bin RENAME redis-sentinel)
```
