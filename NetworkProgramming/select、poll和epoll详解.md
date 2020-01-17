# select、poll和epoll
在[Unix IO模型]()中简单介绍了什么是IO复用，现在介绍实现IO复用的几种方式：select、poll和epoll  
其中select和poll都不如epoll高效（下面会分析为什么），但是可以将poll和epoll封装成跨平台的统一接口（详见[封装poll和epoll]()），所以这里简单介绍select，重点介绍poll和epoll  

## select
select函数原型： **[1]**
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
这里很难有短小的示例代码说明select如何使用，可参考[EchoServer_select.cpp](https://github.com/liuyunian/tools-cxx/blob/master/examples/echo/EchoServer_select.cpp)，下面是几点说明：  
* 第一个参数nfds是用来指示调用select陷入内核后监视文件描述符的范围，0~(nfds-1)这个范围的文件描述符都会被监视，所以说该值应该等于当前进程中最大文件描述符+1
* fd_set本质上是一个long类型，每一位代表一个文件描述符，readfds、writefds和exceptfds都是值结果参数，把关心的文件描述符传递给内核，当有事件发生时，内核会修改这几个参数来说明哪个文件描述符有事件发生
* 与fd_set相关的几个宏定义：FD_ZERO()将fd_set所有位置0，FD_SET()将指定位置置1，FD_CLR()将指定位置置0，FD_ISSET()检测指定位置是否为1

通过简单使用select可以体会到其存在的问题：  
1. select只能告诉我们有事件发生了，但是我们还是需要遍历所关心文件描述符，查看是哪个文件描述符有事件发生了
2. 每次调用select都需要将fd_set从用户空间拷贝到内核空间，之后内核遍历完整的fd_set来确定所关心的事件，返回时再将fd_set从内核空间拷贝到用户空间，显然如果所关心的事件很多的话，开销会很大
3. 内核对fd_set大小做了限制（最大1024）且不能修改

## poll
poll函数原型：**[2]**
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
* pollfd结构中的events用于指定当前文件描述符关心的事件，而revents则返回该描述符的状态，这两个变量一个为调用值，一个为返回结果，避免了像select函数中的值-结果参数 **[3]**  

poll函数也是只能告诉我们有事件发生，但是并不能告诉我们是哪个文件描述符发生了事件，所以依旧需要遍历我们所关心的所有的文件描述符，而下面所讲的epoll方式弥补了select和poll存在的问题，提供了一个更高效的IO复用方式

## epoll
epoll是支持高并发的IO复用技术，可以让单台主机支持数万、数十万的连接，并且不会像select和poll那样随着连接数的增加，性能出现明显下降

### 使用
先来看如何使用epoll，epoll技术提供了三个函数：epoll_create、epoll_ctl、epoll_wait，函数原型如下：**[4][5][6]**
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
* epoll本身也对应一个文件描述符，也可以由其他IO复用方式监听事件是否发生
* epoll_ctl用于增加、修改、删除监听哪些文件描述符，这样避免了陷入内核时大量数据的拷贝
* epoll不但会告诉我们有事件发生，还会告诉我们哪些文件描述符产生了事件，这样就不再像select和poll需要遍历关心的文件描述符，所以当监听的文件描述符很多时，效率会大幅提升

### 原理
简单描述下epoll原理：
函数epoll_create/epoll_create1会在内核中创建一个eventpoll结构成员，该结构如下图所示：  
![eventpoll结构](/Images/np/eventpoll.png)  
我们主要关注eventpoll结构的rbr和rdlist两个成员即可，rbr指向一棵红黑树的根节点，rdlist指向一个双向链表的头结点，刚开始两个成员都为NULL  

epoll_ctl函数将传递给内核的struct epoll_event转化成一个红黑树的节点(epitem)，根据op选项对红黑树进行不同的操作（增删改），之后向内核注册有事件到来时的回调函数，该回调函数会向rdlist双向链表中插入就绪的文件描述符并唤醒epoll_wait **[7]**   

epoll_wait函数判断rdlist中有无数据，如果有数据则返回，如果没有数据则等到timeout时间到后返回  
更多epoll原理的细节可以阅读参考文献 **[7]**

### 工作模式分析
epoll有两种工作模式：默认情况下，epoll对象工作在LT模式下
* LT(Level Trigged)水平触发模式：来一个事件，如果不去处理，那么该事件之后会一直触发
* ET(Edge Trigged)边沿触发模式：来一个事件，内核只通知一次，如果不去处理的话

下面举例说明两种模式，假设事件发生时变为高电平  
**EPOLLIN事件发生的情况**   
对于监听套接字：已完成连接队列为空时可以认为是低电平，当一个客户端经过三次握手连入时，已连接队列非空，此时可以认为是高电平；调用accept()函数可以从已连接队列中取走一个连接，如果取走后队列为空，那就变为了低电平，当然如果accept()执行出错则未能从已连接队列中取走连接，那还是高电平
* 对于LT模式，高电平的情况会一直触发监听套接字的可读事件，也就是已完成连接队列中非空那么就会触发EPOLLIN。这就是为什么accept()出现EMFILE错误时会发生busy-loop
* 对于ET模式，只有低电平切换为高电平时才会触发，也就是已完成队列由空变为非空时才会触发EPOLLIN。如果触发了监听套接字的EPOLLIN事件，已连接队列中有多个客户端的连接，那么此时一定要使用accept()取走所有的连接，让其变为低电平，否则就再也不会触发监听套接字的可读事件了  

对于连接套接字：套接字对应的接收缓冲区（内核）为空时为低电平，收到数据非空是为高电平，读取完所有的数据之后，再次变为低电平
同理LT模式还是在高电平时触发可读事件，而ET模式则是在低电平切换到高电平时才触发EPOLLIN事件

**EPOLLOUT事件发生的情况**   
连接套接字对应的发送缓冲区（内核）已满时为低电平，不满时为高电平  
* 对于LT模式，连接刚接入时，发送缓冲区肯定不满，所以此时不能让epoll对象关注连接套接字的EPOLLOUT事件，否则会一直不停的触发可写事件 -- busy-loop，当发送数据时，发送缓存区已满时，此时关注连接套接字的EPOLLOUT事件，这样发送缓冲区不满时，就会触发可写事件，此时继续发送剩余的数据，发送完之后要取消关注连接套接字的可写事件
* 对于ET模式，可以一开始就关注EPOLLOUT事件，因为只有由低电平切换为高电平时才会触发，所以不用担心会差生busy-loop

值的注意的是，目前还没有有效的数据证明LT模式比ET模式效率低，所以推荐使用LT模式，原因如下 **[8]**：
1. 与poll(2)兼容，文件描述符数目较少，活动文件描述符比例较高时，epoll(4)不见得比poll(2)更高效，所以必要时可以切换poller
2. LT模式编程更容易，以往使用select(2)/poll(2)的经验都可以使用，不可能发生漏掉事件的bug
3. 读写时不必等候出现EAGAIN，可以节省系统调用次数，降低延迟

## 参考文献
* [1]. Linux man-pages：man 2 select
* [2]. Linux man-pages：man 2 poll
* [3]. 《Unix网络编程 卷1：套接字联网API》第三版，P144
* [4]. Linux man-pages：man 2 epoll_create
* [5]. Linux man-pages：man 2 epoll_ctl
* [6]. Linux man-pages：man 2 epoll_wait
* [7]. [Linux内核剖析-----IO复用函数epoll内核源码剖析](https://blog.csdn.net/Eunice_fan1207/article/details/99674021) -- Eunice_fan1207
* [8]. 《Linux多线程服务端编程 -- 使用muduo C++网络库》 -- 陈硕
