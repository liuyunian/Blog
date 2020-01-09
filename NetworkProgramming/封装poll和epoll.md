# 封装poll和epoll
在[select、poll和epoll详解](https://blog.csdn.net/sdjn_lyn/article/details/103844319)中提到了可以对poll和epoll进行封装，提供统一的接口。封装目的主要有两个：复用和易用，以下封装希望屏蔽poll和epoll使用时的诸多细节，提供简单易用的接口  
珠玉在前，本文所提供的代码参考自[muduo网络库](https://github.com/chenshuo/muduo)，另外代码有用到[tools-cxx](https://github.com/liuyunian/tools-cxx)中提供的base库和log库

## 框架
首先看一下整体类图：  
![poller类图](/Images/np/poller.png)  
PollPoller和EPollPoller分别对poll和epoll进行了封装，两者继承自Poller，Poller提供了统一的接口  
Channel封装了事件分发机制，与Poller是聚合关系 

## Channel
Channel对象负责文件描述符事件的分发，每个Channel对象自始至终只负责一个文件描述符的事件分发，Channel对象中保存了其负责的文件描述符的事件处理函数，事件发生时会调用响应的事件处理函数。值得注意的是：Channel不持有对应的文件描述符，即不负责close
Channel.h如下，源文件见[Channel.cpp](https://github.com/liuyunian/tools-cxx/blob/master/tools/poller/Channel.cpp) **[1]**
``` C++
#ifndef CHANNLE_H_
#define CHANNLE_H_

#include <functional>

#include "tools/base/noncopyable.h"

class Poller;

class Channel : noncopyable {
public:
  Channel(Poller *poller, int fd);
  ~Channel() = default;

  int get_fd() const {
    return m_fd;
  }

  int get_events() const {
    return m_events;
  }

  int get_revents() const {
    return m_revents;
  }

  void set_revents(int revents){
    m_revents = revents;
  }

  int get_index() const {
    return m_index;
  }

  void set_index(int index){
    m_index = index;
  }

  void enable_reading(){
    m_events |= kReadEvent;
    update();
  }

  void disable_reading(){
    m_events &= ~kReadEvent;
    update();
  }

  void enable_writing(){
    m_events |= kWriteEvent;
    update();
  }

  void disable_writing(){
    m_events &= ~kWriteEvent;
    update();
  }

  bool is_writing() const {
    return m_events & kWriteEvent;
  }
  
  typedef std::function<void()> EventCallback;
  void set_read_callback(const EventCallback &cb){
    m_readCallback = cb;
  }

  void set_write_callback(const EventCallback &cb){
    m_writeCallback = cb;
  }

  void set_error_callback(const EventCallback &cb){
    m_errorCallback = cb;
  }

  void set_close_callback(const EventCallback &cb){
    m_closeCallback = cb;
  }

  void handle_event();

  void remove();

private:
  void update();

private:
  // events
  static const int kReadEvent;
  static const int kWriteEvent;

  // revents
  static const int READ_EVENT;
  static const int WRITE_EVENT;
  static const int ERROR_EVENT;
  static const int CLOSE_EVENT;

  Poller *m_poller;   // 所属的m_poller
  const int m_fd;     // 负责的文件描述符
  int m_events;       // 关心的事件
  int m_revents;      // 发生的事件
  int m_index;        // used by Poller，既用来表征在m_pollfdList中的位置又用于区分add/update操作

  EventCallback m_readCallback;   // 可读事件回调
  EventCallback m_writeCallback;  // 可写事件回调
  EventCallback m_closeCallback;  // 连接关闭事件回调
  EventCallback m_errorCallback;  // 错误事件回调
};

#endif // CHANNLE_H_
```
* Channel类不是线程安全的，可以由持有Poller的类负责线程安全
* 每个Channel都属于一个Poller对象，但是Poller不对Channel对象的生命周期负责
* 通过调用enable_xx函数和disable_xx函数来给持有的文件描述符注册事件和取消注册事件，其中enable_xx函数回去调用update函数，该函数会将该Channel对象指针传递到Poller中，而remove函数则是从Poller中移除该Channel对象指针
* 通过set_xxx_callback函数提供事件发生后的回调函数，事件发生后调用handle_event函数，根据m_revent记录发生的事件就是事件分发

## Poller
头文件如下，源文件见[Poller.cpp](https://github.com/liuyunian/tools-cxx/blob/master/tools/poller/Poller.cpp) **[1]**
``` C++
#ifndef POLLER_H_
#define POLLER_H_

#include <vector>
#include <map>

#include "tools/base/noncopyable.h"

class Channel;

class Poller : noncopyable {
public:
  Poller() = default;
  virtual ~Poller() = default;

  typedef std::vector<Channel*> ChannelList;
  virtual ChannelList& poll(int timeoutMs) = 0;

  virtual void update_channel(Channel *channel) = 0;

  virtual void remove_channel(Channel *channel) = 0;

  static Poller* create_poller();

protected:
  ChannelList m_activeChannels;
  std::map<int, Channel*> m_channelStore;
};

#endif // POLLER_H_
```
* Poller类以接口方式存在，定义了诸多的纯虚函数由子类实现
* m_channelStore记录了都需要关注哪些文件描述符的事件，m_activeChannel则是记录哪些文件描述符发生了事件
* poll函数阻塞等待事件发生，返回发生事件的Channel列表，updata_channel函数用于添加要关注事件的Channel或者修改已有的Channel所关注的事件，remove_channel()则是移除指定的Channel，不再关注其事件
* 静态函数create_poller创建一个Poller子类对象，默认是EPollPoller对象，可以通过定义环境变量USE_POLL切换poll

## PollPoller & EPollPoller
这部分代码不再罗列，见：[PollPoller.h](https://github.com/liuyunian/tools-cxx/blob/master/tools/poller/PollPoller.h)、[PollPoller.cpp](https://github.com/liuyunian/tools-cxx/blob/master/tools/poller/PollPoller.cpp)、[EPollPoller.h](https://github.com/liuyunian/tools-cxx/blob/master/tools/poller/EPollPoller.h)、[EPollPoller.cpp](https://github.com/liuyunian/tools-cxx/blob/master/tools/poller/EPollPoller.cpp) **[1]**

## 使用示例
以下代码是用封装的Poller实现的echo服务器，其中还用到了封装的socket库，关于socket库可见博客[封装TCP socket]()，这里主要关注Poller的使用 **[2]**  
``` C++
#include <vector>
#include <map>

#include <string.h>     // memset

#include <tools/log/log.h>
#include <tools/base/Exception.h>
#include <tools/socket/ConnSocket.h>
#include <tools/socket/SocketsOps.h>
#include <tools/socket/InetAddress.h>
#include <tools/socket/ServerSocket.h>
#include <tools/poller/Channel.h>
#include <tools/poller/Poller.h>

#define LISTEN_PORT 9000
#define BUFFER_SZ 1024

ServerSocket ss(LISTEN_PORT);
Poller *poller = Poller::create_poller();
std::map<Channel*, ConnSocket> connPool;

void on_message(Channel *channelPtr){
  char buf[BUFFER_SZ] = {0};
  auto connSocketIter = connPool.find(channelPtr);
  ssize_t len = connSocketIter->second.read(buf, BUFFER_SZ);
  if(len < 0){
    LOG_WARN("read error");
  }
  else if(len == 0){
    LOG_INFO("client disconnects");

    channelPtr->remove();
    delete channelPtr;
    connPool.erase(connSocketIter);
  }
  else{
    connSocketIter->second.write(buf, len);
  }
}

void on_connection(){
  try{
    ConnSocket connSocket = ss.accept_nonblocking();
    Channel *connChannel = new Channel(poller, connSocket.get_sockfd());
    connChannel->set_read_callback(std::bind(on_message, connChannel));
    connChannel->enable_reading();
    connPool.insert({connChannel, connSocket});
  }
  catch(const Exception &e){
    LOG_WARN("accept error");
  }
}

int main(){
  ss.set_reuse_address(true);
  ss.listen();
  LOG_INFO("server is listening...");

  Channel listenChannel(poller, ss.get_sockfd());
  listenChannel.set_read_callback(on_connection);
  listenChannel.enable_reading();

  for(;;){
    Poller::ChannelList activeChannels = poller->poll(-1);
    for(auto &channelPtr : activeChannels){
      channelPtr->handle_event();
    }
  }

  return 0;
}
```
可以参考[tools-cxx中timer库](https://github.com/liuyunian/tools-cxx/tree/master/tools/timer)了解Poller的使用

## 参考文献
* [1]. 源码地址：[https://github.com/liuyunian/tools-cxx/tree/master/tools/poller](https://github.com/liuyunian/tools-cxx/tree/master/tools/poller)
* [2]. 源码地址：[https://github.com/liuyunian/tools-cxx/blob/master/examples/echo/EchoServer_poller.cpp](https://github.com/liuyunian/tools-cxx/blob/master/examples/echo/EchoServer_poller.cpp)