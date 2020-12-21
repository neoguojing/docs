# cmake

## 常用

CMAKE_MINIMUM_REQUIRED(VERSION 3.0)   最小版本限制
SET(PROJECT_NAME face-search)         设置变量名称
PROJECT(${PROJECT_NAME} C CXX)        设置项目信息
ADD_EXECUTABLE(${PROJECT_NAME} ${c_sources} ${cxx_sources})    添加编译目标
TARGET_LINK_LIBRARIES(${PROJECT_NAME} ${CMAKE_THREAD_LIBS_INIT} uuid pthread dl faiss cassandra rdkafka cppkafka)   



