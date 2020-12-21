# cmake

## 常用

- CMAKE_MINIMUM_REQUIRED(VERSION 3.0)   最小版本限制
- SET(PROJECT_NAME face-search)         设置变量名称
- PROJECT(${PROJECT_NAME} C CXX)        设置项目信息,启用c和cxx工具链
- LINK_DIRECTORIES    制定链接库文件目录
- ADD_EXECUTABLE(${PROJECT_NAME} ${c_sources} ${cxx_sources})    添加编译目标
- TARGET_LINK_LIBRARIES(${PROJECT_NAME} ${CMAKE_THREAD_LIBS_INIT} uuid pthread dl faiss cassandra rdkafka cppkafka)   链接目标文件和库文件
- target_include_directories 引用目标头文件目录
- aux_source_directory(. DIR_SRCS) # 查找当前目录下的所有源文件 并将名称保存到 DIR_SRCS 变量
- add_subdirectory(math)  添加子目录
- add_library (MathFunctions ${DIR_LIB_SRCS}) 生成链接库
- INCLUDE_DIRECTORIES 头文件目录
- message 打印消息
- ADD_DEFINITIONS(-DKESTREL_CUDA) 添加宏,可以在代码中使用宏进行条件编译,宏可以作为cmake的参数
- ENV 访问环境变量 if( NOT DEFINED ENV{JAVA_HOME}) 做判断
- FILE(GLOB_RECURSE c_sources ${CMAKE_CURRENT_SOURCE_DIR}/src/*.c)  遍历制定目录服务glob表达式的文件写入c_sources变量
- find_package(CURL REQUIRED)   查找目标库,可设置必选
  ```
  FindCURL.cmake 预先由curl定义好
  默认会定义若干个变量:
  CURL_FOUND 标识是否找到
  CURL_INCLUDE_DIR include文件目录
  CURL_LIBRARY  lib文件目录
  
  ```
  ## pkgconfig
  pkgconfig 保证跨平台
  ```
  find_package(PkgConfig REQUIRED)

  pkg_check_modules(ffmpeg REQUIRED IMPORTED_TARGET libavcodec libavformat libavutil)

  target_link_libraries(${PROJECT_NAME} PRIVATE PkgConfig::ffmpeg)
  ```
  
  ## 内置变量
  - CMAKE_<LANG>_COMPILER
  - CMAKE_<LANG>_COMPILER_ID
  - CMAKE_<LANG>_COMPILER_VERSION
  - CMAKE_<LANG>_FLAGS   指定debug CMAKE_CXX_FLAGS_DEBUG 制定release CMAKE_CXX_FLAGS_RELEASE
  - CMAKE_CURRENT_SOURCE_DIR 当前cmake文件的目录

  ## if else
  https://blog.csdn.net/maizousidemao/article/details/104099776

