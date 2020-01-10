# 封装TCP socket
对字节序转化函数、网络地址、Socket操作函数、监听套接字和连接套接字进行了封装，部分代码参考自[muduo网络库](https://github.com/chenshuo/muduo)  
源码地址：[https://github.com/liuyunian/tools-cxx/tree/master/tools/socket](https://github.com/liuyunian/tools-cxx/tree/master/tools/socket)

## 字节序转化函数
对于字节序转化函数，Unix和Linux共同拥有的是htonl(), htons(), ntohl(), ntohs()  
另外对于Linux，在glibc 2.19版本中添加了htobe16(), htole16(), be16toh(), le16toh()等函数  
为了统一，对字节序转化函数进行了如下封装：  

| 函数 | 功能 | 参数 | 返回值 |
| :---: | :---: | :---: | :---:|
| host_to_network64 | 将64位无符号整数由本机字节序转化成网络字节序 | uint64_t | uint64_t |
| host_to_network32 | 32位无符号整数 | uint32_t | uint32_t |
| host_to_network16 | 16位无符号整数 | uint16_t | uint16_t |
| network_to_host64 | 将64位无符号整数由网络字节序转化成本机字节序 | uint64_t | uint64_t |
| network_to_host32 | 32位无符号整数 | uint32_t | uint32_t |
| network_to_host16 | 16位无符号整数 | uint16_t | uint16_t |

源码：[Endian.h](https://github.com/liuyunian/tools-cxx/blob/master/tools/socket/Endian.h)  
示例：[Endian_test.h](https://github.com/liuyunian/tools-cxx/blob/master/examples/socket/Endian_test.cpp)

## 网络地址
对于Unix和Linux，网络地址有多种，常用的有：
* 通用套接字地址结构struct sockaddr
* IPv4套接字地址结构struct sockaddr_in
* IPv6套接字地址结构struct sockaddr_in6
* ...  

这里对IPv4和IPv6地址结构进行了封装，使得构建地址、地址之间的转化更简单  
源码：[InetAddress.h](https://github.com/liuyunian/tools-cxx/blob/master/tools/socket/InetAddress.h)、[InetAddress.cpp](https://github.com/liuyunian/tools-cxx/blob/master/tools/socket/SocketsOps.cpp)  
示例：[InetAddress.cpp](https://github.com/liuyunian/tools-cxx/blob/master/examples/socket/InetAddress_test.cpp)

## Socket操作函数
对诸如socket(), bind(), listen()等函数进行了包装，包装的主要目的是简化错误处理  

| 函数 | 功能 | 返回值 |
| :---: | :---: | :---: | :---: |
| create_socket(sa_family_t)| 创建阻塞式Socket描述符 | sockfd -- int |
| create_nonblocking_socket(sa_family_t) | 创建非阻塞式Socket描述符 | sockfd -- int | 
| close(int) | -- | void | 
| shutdown_write(int) | 禁止套接字发送数据 | void | 
| read(int, void*, ssize_t) | -- | ssize_t |
| readv(int, const struct iovec*, int) | -- | ssize_t |
| write(int, const void*, ssize_t) | -- | ssize_t |
| bind | -- | void |
| listen | -- | void | 
| accept(int, struct sockaddr_in6*) | 接收连接并记录客户端的地址，返回阻塞式Socket描述符 | connfd -- int |
| accept_nonblocking(int, struct sockaddr_in6*) | 记录客户端地址，返回非阻塞式的Socket描述符 | connfd -- int |
| connect | -- | void |  

源码：[SocketsOps.h](https://github.com/liuyunian/tools-cxx/blob/master/tools/socket/SocketsOps.h)、[SocketsOps.cpp](https://github.com/liuyunian/tools-cxx/blob/master/tools/socket/SocketsOps.cpp)

## 监听套接字ServerSocket
参照Java Net中的ServerSocket，采用RAII技术对监听套接字文件描述符进行封装，使得文件描述符的生命周期与类对象的生存期关联，更方便高效的管理socket文件描述符资源

### ServerSocket.h
``` C++
#ifndef SERVERSOCKET_H_
#define SERVERSOCKET_H_

#include <arpa/inet.h>
#include "tools/socket/InetAddress.h"

class ConnSocket;

#include "tools/base/noncopyable.h"

class ServerSocket : noncopyable{
public:
  ServerSocket(int port, bool isBlocking = false, sa_family_t family = AF_INET);

  ServerSocket(const InetAddress &localAddr, bool isBlocking = false);

  ~ServerSocket();

  inline int get_sockfd() const {
    return m_sockfd;
  }

  void listen();

  ConnSocket accept();

  ConnSocket accept_nonblocking();

  void set_reuse_address(bool on);

  void set_reuse_port(bool on);

private:
  InetAddress m_localAddr;
  const int m_sockfd;
};

#endif // SERVERSOCKET_H_
```

### ServerSocket.cpp
``` C++
#include <string.h>     // memset

#include "tools/base/Exception.h"
#include "tools/socket/ServerSocket.h"
#include "tools/socket/ConnSocket.h"
#include "tools/socket/SocketsOps.h"
#include "tools/socket/InetAddress.h"

ServerSocket::ServerSocket(int port, bool isBlocking, sa_family_t family) : 
  m_sockfd(isBlocking ? sockets::create_socket(family)
                      : sockets::create_nonblocking_socket(family)),
  m_localAddr(port, family)
{}

ServerSocket::ServerSocket(const InetAddress &localAddr, bool isBlocking) : 
  m_sockfd(isBlocking ? sockets::create_socket(localAddr.get_family())
                      : sockets::create_nonblocking_socket(localAddr.get_family())),
  m_localAddr(localAddr)
{}

ServerSocket::~ServerSocket(){
  sockets::close(m_sockfd);
}

void ServerSocket::listen(){
  sockets::bind(m_sockfd, m_localAddr.get_sockaddr());
  sockets::listen(m_sockfd);
}

ConnSocket ServerSocket::accept(){
  struct sockaddr_in6 addr;
  memset(&addr, 0, sizeof(addr));
  int connfd = sockets::accept(m_sockfd, &addr);
  if(connfd < 0){
    throw Exception("accept error");
  }

  InetAddress remoteAddr(addr);
  return ConnSocket(connfd, remoteAddr);
}

ConnSocket ServerSocket::accept_nonblocking(){
  struct sockaddr_in6 addr;
  memset(&addr, 0, sizeof(addr));
  int connfd = sockets::accept_nonblocking(m_sockfd, &addr);
  if(connfd < 0){
    throw Exception("accept_nonblocking error");
  }

  InetAddress remoteAddr(addr);
  return ConnSocket(connfd, remoteAddr);
}

void ServerSocket::set_reuse_address(bool on){
  sockets::set_reuse_address(m_sockfd, on);
}

void ServerSocket::set_reuse_port(bool on){
  sockets::set_reuse_port(m_sockfd, on);
}
```
* 提过了两种构造方法，一种是只指定Port并采用本机任一IP地址；另一种是使用InetAddress构造，可以指定IP
* accept()和accept_nonblocking()返回ConnSocket对象，该类是可复制的，下面会介绍其实现。另外这两个函数出错时会抛出异常，所以在使用时应做异常处理

## 连接套接字ConnSocket
同样也是采用RAII方式对连接套接字文件描述符进行封装  
鉴于在实际使用时需要传递ConnSocket，所以将其实现为可复制的，内部采用计数方式，每复制一次，计数加一，析构之后计数减一，当计数由1变为0时才会close文件描述符  

### ConnSocket.h
``` C++
#ifndef CONNSOCKET_H_
#define CONNSOCKET_H_

#include <sys/types.h>

#include "tools/base/copyable.h"
#include "tools/base/noncopyable.h"
#include "tools/socket/InetAddress.h"

class ConnSocket;

class SocketGuard : noncopyable {
private:
  friend class ConnSocket;
  SocketGuard(int sockfd);
  ~SocketGuard();

  const int m_sockfd;
  int m_count;
};

class ConnSocket : copyable{
public:
  ConnSocket();

  ConnSocket(int sockfd, InetAddress addr);

  ConnSocket(const ConnSocket &sock);

  ~ConnSocket();

  ConnSocket& operator=(const ConnSocket &sock);

  inline int get_sockfd() const {
    return m_guard->m_sockfd;
  }

  inline InetAddress get_remote_address() const {
    return m_remoteAddr;
  }

  ssize_t read(void *buf, ssize_t count);

  ssize_t readv(const struct iovec *iov, int iovcnt);

  ssize_t write(const void *buf, ssize_t count);

  void shutdown_write();

  void set_keep_alive(bool on);

  void set_no_delay(bool on);

private:
  SocketGuard *m_guard;
  InetAddress m_remoteAddr;
};

#endif // CONNSOCKET_H_
``` 

### ConnSocket.cpp
``` C++
#include <string.h> // memset

#include "tools/socket/ConnSocket.h"
#include "tools/socket/SocketsOps.h"

SocketGuard::SocketGuard(int sockfd) : 
  m_sockfd(sockfd),
  m_count(1)
{}

SocketGuard::~SocketGuard(){
  sockets::close(m_sockfd);
}

ConnSocket::ConnSocket() : 
  m_guard(nullptr),
  m_remoteAddr(){}

ConnSocket::ConnSocket(int sockfd, InetAddress addr) :
  m_guard(new SocketGuard(sockfd)),
  m_remoteAddr(addr){}

ConnSocket::ConnSocket(const ConnSocket &sock) : 
  m_guard(sock.m_guard),
  m_remoteAddr(sock.m_remoteAddr)
{
  ++ m_guard->m_count;
}

ConnSocket::~ConnSocket(){
  if(m_guard != nullptr){
    -- m_guard->m_count;
    if(m_guard->m_count == 0){
      delete m_guard;
    }
  }
}

ConnSocket& ConnSocket::operator=(const ConnSocket &sock){
  this->~ConnSocket();

  m_guard = sock.m_guard;
  ++ m_guard->m_count;
  m_remoteAddr = sock.m_remoteAddr;

  return *this;
}

ssize_t ConnSocket::read(void *buf, ssize_t count){
  memset(buf, 0, count);
  return sockets::read(m_guard->m_sockfd, buf, count);
}

ssize_t ConnSocket::readv(const struct iovec *iov, int iovcnt){
  return sockets::readv(m_guard->m_sockfd, iov, iovcnt);
}

ssize_t ConnSocket::write(const void *buf, ssize_t count){
  return sockets::write(m_guard->m_sockfd, buf, count);
}

void ConnSocket::shutdown_write(){
  sockets::shutdown_write(m_guard->m_sockfd);
}

void ConnSocket::set_keep_alive(bool on){
  sockets::set_keep_alive(m_guard->m_sockfd, on);
}

void ConnSocket::set_no_delay(bool on){
  sockets::set_keep_alive(m_guard->m_sockfd, on);
}
```

ServerSocket和ConnSocket的使用示例可以参考博客[多版本EchoServer]()