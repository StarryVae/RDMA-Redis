# RDMA-Redis

## Build with cmake: ##

    - mkdir build
    - cd build
    - cmake ..  
    - make all

## Communication design details ##

### Server ###

* redis.c/main()：初始化RDMA资源，建立QP连接，创建线程processEventOnce()
* redis.c/processEventOnce()：其实就是正常RDMA编程中的poll cqe的循环线程，监听客户端发来的请求，并调用processEvents()函数进行请求的解析处理
* networking.c/processEvents()：调用processInputBuffer_() -> processCommand_()
* redis.c/processCommand_()：解析客户端请求的command，调用addReply()相关函数返回客户端信息
* networking.c/addReply()：这个函数就是调用RDMA的post_send发送数据

### Client ###

* redis-benchmark.c/main()：初始化RDMA资源，建立QP连接，根据运行提供的参数执行benchmark()函数
* redis-benchmark.c/benchmark()：创建client，调用processEvents()函数
* redis-benchmark.c/processEvents()：该函数其实就是创建线程processEventOnce()
* redis-benchmark.c/processEventOnce()：最终的工作线程负责调用RDMA的post_send发送客户端请求

### RDMA parameters ###

* Connection type：RC
* verb：WRITE_WITH_IMM
* inline：no
* poll strategy：busy polling




