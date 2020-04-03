# RDMA-Redis

## Build with cmake: ##

    - mkdir build
    - cd build
    - cmake ..  
    - make all
	
## Quick Start ##

### Server ###

    - cd build
    - sudo ./redis-server
	
### Client ###

    - cd build
    - sudo ./redis-benchmark

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


### To be solved ###

很明显，write_with_imm一方面不能使用inline的优势，另一方面它是two-sided的verb，性能不如one-sided的write verb，因此利用write来进行client到server的通信会进一步提升性能，但设计也会相应的更为复杂。
主要的问题就在于write不需要server端post recv，也就不会产生cqe，从而导致server端无法感知client请求的到来，也就不能立即处理请求。

目前考虑的方案是采取Accelerating Redis with RDMA Over InfiniBand文章中的方法，将server端的接收信息的buffer分成多个chunk，每个chunk对应于一个client，同时client的请求信息的格式也需要重新定制，
包括：request head, command and ending flag，其中的ending flag可以用来判断请求信息的到来。

### 代码详细分析###

首先，元语一定使用的是WRITE_WITH_IMM，远端内存信息也在QP建立连接时传输完成了（struct cm_con_data_t），这些换成WRITE元语
之后一样适用。接下来主要考虑接收数据的buffer在代码中的位置，经过分析，redisServer结构体中的resources结构体中的buff变量是
接收数据的buffer。

