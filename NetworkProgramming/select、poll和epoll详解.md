# select、poll和epoll
在[Unix下的IO模型](./Unix下的IO模型.md)中简单介绍了什么是IO复用，现在介绍实现IO复用的几种方式：select、poll和epoll  
其中select和poll都不如epoll高效（下面会分析为什么），但是可以将poll和epoll封装成跨平台的统一接口（详见[封装poll和epoll](./封装poll和epoll.md)），所以这里简单介绍select，重点介绍poll和epoll  

## select
select函数原型： ***[1]***
``` C++
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

/*
 * @param nfds：待检测文件描述的个数
 * @param readfds： 测试是否可读的文件描述符的集合
 * @param writefds：测试是否可写的文件描述符的集合
 * @param exceptfds：异常条件的文件描述符的集合
 * @param timeout：等待时间，如果置为空指针则将一直等待直到一个事件发生，如果置为0则表示立即返回
 * @return 返回事件发生的文件描述符的个数，等待时间到了但是没有事件发生则返回0，出错返回-1
*/
int select(int nfds, fd_set *readfds, fd_set *writefds,
          fd_set *exceptfds, struct timeval *timeout);
```
这里很难有短小的示例代码说明select如何使用，可参考[EchoServer_select.cpp]()，下面是几点说明：  
* 第一个参数nfds是用来指示调用select陷入内核后监视文件描述符的范围，0~(nfds-1)这个范围的文件描述符都会被监视，所以说该值应该等于当前进程中最大文件描述符+1
* fd_set本质上是一个long类型，每一位代表一个文件描述符，readfds、writefds和exceptfds都是值结果参数，把关心的文件描述符传递给内核，当有事件发生时，内核会修改这几个参数来说明哪个文件描述符有事件发生
* 与fd_set相关的几个宏定义：FD_ZERO()将fd_set所有位置0，FD_SET()将指定位置置1，FD_CLR()将指定位置置0，FD_ISSET()检测指定位置是否为1

通过简单使用select可以体会到其存在的问题：  
1. select只能告诉我们有事件发生了，但是我们还是需要遍历所关心文件描述符，查看是哪个文件描述符有事件发生了
2. 每次调用select都需要将fd_set从用户空间拷贝到内核空间，之后内核遍历完整的fd_set来确定所关心的事件，返回时再将fd_set从内核空间拷贝到用户空间，显然如果所关心的事件很多的话，开销会很大
3. 内核对fd_set大小做了限制（最大1024）且不能修改

## poll
poll函数原型：***[2]***
``` C++
 #include <poll.h>

/*
 * @param fds：pollfd结构数组指针
 * @param nfds：pollfd数组大小
 * @param timeout：等待时间，单位ms。-1：一直等待直到一个事件发生，0：立即返回，>0：等待指定的毫秒数
*/ 
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

struct pollfd {
  int   fd;         // 文件描述符
  short events;     // 请求关注的事件，常用取值：POLLIN可读事件，POLLOUT可写事件，如果一个fd既关心可读事件又关心可写事件则events = POLLIN | POLLOUT
  short revents;    // 返回的事件，用来检测当前文件描述符发生了什么事件，POLLIN可读事件，POLLOUT可写事件，POLLERR发生错误，POLLHUP发生挂起，POLLNVAL描述符不是一个打开的文件
};
```
具体使用的示例代码可以参考[EchoServer_poll.cpp](https://github.com/liuyunian/tools-cxx/blob/master/examples/echo/EchoServer_poll.cpp)，以下是几点说明：  
* 使用poll函数的思路不同于select函数，poll函数通过一个pollfd数组来指定监听哪些文件描述符，以及什么事件  
* pollfd结构中的events用于指定当前文件描述符关心的事件，而revents则返回该描述符的状态，这两个变量一个为调用值，一个为返回结果，避免了像select函数中的值-结果参数 ***[3]***  

poll函数也是只能告诉我们有事件发生，但是并不能告诉我们是哪个文件描述符发生了事件，所以依旧需要遍历我们所关心的所有的文件描述符，而下面所讲的epoll方式弥补了select和poll存在的问题，提供了一个更高效的IO复用方式

## epoll
epoll是支持高并发的IO复用技术，可以让单台主机支持数万、数十万的连接，并且不会像select和poll那样随着连接数的增加，性能出现明显下降

### 使用
先来看如何使用epoll，epoll技术提供了三个函数：epoll_create、epoll_ctl、epoll_wait，函数原型如下：***[4][5][6]***
``` C++
#include <sys/epoll.h>

/*
 * @brief 打开一个epoll文件描述符
 * @param size：已经弃用，但是需要提供一个大于0的值
 * @param flags：用于设置epoll文件描述符标志，比如flags = EPOLL_CLOEXEC，表示设置close-on-exec标志；flag = NONBLOCK，表示epoll文件描述符为非阻塞的
 * @return 成功返回一个epoll文件描述符，失败返回-1
 * 两个函数功能没有什么区别，推荐统一使用epoll_create1
*/
int epoll_create(int size);
int epoll_create1(int flags);
```
``` C++
#include <sys/epoll.h>

/*
 * @brief epoll文件描述符的控制接口
 * @param epfd：要操作的epoll文件描述符
 * @param op：指定什么操作，EPOLL_CTL_ADD添加，EPOLL_CTL_MOD修改，EPOLL_CTL_DEL删除
 * @param fd：要关注的文件描述符
 * @param event：epoll_event结构指针
*/ 
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

typedef union epoll_data {
  void        *ptr;
  int          fd;
  uint32_t     u32;
  uint64_t     u64;
} epoll_data_t;

struct epoll_event {
  uint32_t     events;      // epoll事件，取值：EPOLLIN读事件；EPOLLOUT写事件；EPOLLERR发生错误；EPOLLHUP：发生挂起；EPOLLET：ET工作方式，默认是LT方式
  epoll_data_t data;        // 用户数据变量，用于回调
};
```
``` C++
#include <sys/epoll.h>

/*
 * @brief 等待IO事件
 * @param epfd：epoll文件描述符
 * @param events：epoll_event结构数组
 * @param maxevents：epll_event结构数组的最大个数
 * @param timeout：等待时间，-1：一直等待直到一个事件发生，0：立即返回，>0：等待指定的毫秒数
 * @return 有事件发生返回事件发生的个数，失败返回-1
*/
int epoll_wait(int epfd, struct epoll_event *events,
              int maxevents, int timeout);
```
示例代码参考[EchoServer_epoll.cpp](https://github.com/liuyunian/tools-cxx/blob/master/examples/echo/EchoServer_epoll.cpp)，几点说明如下： 
* 

### 原理

### 工作模式分析

## 参考文献
* [1]. Linux man-pages：man 2 select
* [2]. Linux man-pages：man 2 poll
* [3]. 《Unix网络编程 卷1：套接字联网API》第三版，P144
* [4]. Linux man-pages：man 2 epoll_create
* [5]. Linux man-pages：man 2 epoll_ctl
* [6]. Linux man-pages：man 2 epoll_wait
