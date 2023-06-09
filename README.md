# 简介
该项目仅用于本人学习，不存在商业用途·

# 执行流程
首先初始化整个项目的前置参数：在 `config.cpp` 中能找到对应的参数，也可以通过 `config.cpp` 调整整个项目的设置。

之后流程：
## WebServer::log_write() 初始化日志模块
日志模块分为同步操作和异步操作。

在日志模块中采用懒汉单例模式。

* 同步：不创建异步线程写日志，直接使用线程池中的线程写日志
* 异步：创建异步线程写日志，通过log的成员函数接口   `Log::flush_log_thread()` , 在该接口中调用 `Log::async_write_log()` 不断地从阻塞队列中取出日志string写入日志文件

## 数据库初始化
在 `WebServer::sql_pool()` 通过数据库连接池类获取数据连接池的指针 `m_connPool`, 调用 `connection_pool::init(...)进行初始化`。

然后调用 `http_conn::initmysql_result(...)` 获取mysql中users表中所有的结果映射在map中。

## 线程池初始化
特判一下线程池中的线程数量是否小于0，以及最大线程的数量是否小于0，如果是的话直接抛出 `std::excepection()`

然后直接创建线程池数组，得到指针 `m_threads` , 并判断线程池是否创建成功

对每个分配的空间创建一条线程，并将每条创建好的线程和主线程进行分离（为了不主线程收子线程的影响而导致阻塞或）

## 选择触发模式
项目中采用的是epoll多路复用所以默认的是LT模式。

## 设置监听socket
`socket(PF_INET, SOCK_STREAM, 0)` 创建socket文件, 得到socket文件描述符

通过 `sockaddr_in` 绑定主机的ip地址以及项目使用的端口号。

通过 `setsockopt(...)` 设置端口可以重用，并使用 `bind(...)` 绑定socket描述符，以及`sockaddr_in` 结构体，但是要转化为 `sockaddr`类型。

然后初始化 `utils` 类，初始化时间单位；

注册epoll内核事件表 `m_epollfd = epoll_create(5)` ，得到epoll事件的文件描述符 `m_epollfd` 。将`epoll_fd`添加到untils类时间链表中。创建一对无名的套接字`m_pipefd`用于进程之间的通信，线程之间使用这一对套接字进行信号的传递。

其中函数`epoll_create(5)`中参数5的作用是告诉内核创建epoll事件表的大小为5.

## 运行
运行采用的是reactor模式，主线程不断接收客户端的连接处理线程，让线程池中的线程进行业务处理最后返回主线程发送响应消息。

使用epoll的多路复用观测事件的发生，
主线程通过在while中不断地轮询事件地发生，并且在`epoll_wait()`中被阻塞，直到有事件发生，返回事件发生的个数以及对应事件的文件描述符。

* 如果是监听的socket文件描述符 `m_listenfd` 说明是新的客户端连接请求。则调用对应的处理函数对新来到的连接调用`accpet()`接收，得到文件描述符 `connfd`，并且注册到timer（定时器中）。
* 如果得到的事件是epoll中的`EPOLLRDHUP`、`EPOLLHUP`、`EPOLLERR`的其中一个，表示服务端关闭了连接，此时调用`deal_timer()`移除对应的定时器。<br><br>其中`EPOLLRDHUP`事件是由于服务器接收到对端正常关闭连接的请求触发，`EPOLLHUP`由于对端socket的文件描述符被挂断，服务器自己断开连接； `EPOLLERR` 由于对端的socket文件描述符发生错误，服务器断开连接。

* 当前处理的文件描述符和之前创建的通信套接字`m_pipefd`中读到的一样且事件类型是`EPOLLIN`则进入信号处理函数`dealwithsignal()`：<br>

在 `dealwithsignal()` 函数中通过 `recv()`函数从缓冲区中接收另外先线程的信号存放在`signal[1024]`中，`ret()`返回接收的字节数，如果接收的字节数小于等于0直接返回false；反之则对每个字节的的信号进行处理，如果 `signal[i] == SIGLRM`将time_out设置为true，**(和定时器相关)**；如果 `signal[i] == SIGTERM` 则将stop_server设置为true, 停止服务器。

* 如果 `event[i].events` 是 `EPOLLIN`类型的事件说明要处理客户端上的信号，调用`dealwiththread()`<br>

在`dealwiththread()`函数中:
`sockfd`是要处理客户端socket的文件描述符，在reactor模式下，先调整定时器重置当前客户端socket的计时。如果检测到读事件则调用 `append()` 将改事件放入请求队列中


