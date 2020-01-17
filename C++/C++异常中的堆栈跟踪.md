# C++异常中的堆栈跟踪
C++异常中的堆栈跟踪就是当程序抛出异常时，能够把抛出异常的语句所在的文件名、函数以及其它上层函数信息都打印出来。  
堆栈跟踪意义重大：在实际的生产过程中，发现代码中bug要比解决bug更加费事费力，而发生异常时的堆栈信息可以帮助我们更快的发现程序中的问题，有助于加快开发。  

C++不像java，在异常类中有提供堆栈跟踪功能，C++标准库中的std::exception及其子类中并没有提供堆栈跟踪功能，所以我们需要给C++中的异常类添加上可以查看异常发生时堆栈信息的功能。  
在具体的实现之前，我们先来看些前提知识：  

## 获取堆栈信息
首先应该考虑的是我们该如何获取到堆栈信息？在Linux平台上提供了如下API用来获取和查看堆栈信息 **[1]**: 
``` C++
#include <execinfo.h>

int backtrace(void **buffer, int size);

char **backtrace_symbols(void *const *buffer, int size);

void backtrace_symbols_fd(void *const *buffer, int size, int fd);
```
backtrace函数用于获取堆栈的地址信息，backtrace_symbols函数把堆栈地址翻译成我们易识别的字符串， backtrace_symbols_fd函数则把字符串堆栈信息输出到文件中 **[2]**，这几个函数详细的介绍、使用和注意事项详见参考文献 **[2]**。

我们来简单使用下backtrace，看获取到的堆栈信息是什么样的：
``` C++
#include <iostream>
#include <execinfo.h>

void test_backtrace(){
  const int maxFrames = 100;                            // 堆栈返回地址的最大个数
  void *frame[maxFrames];                               // 存放堆栈返回地址
  int nptrs = backtrace(frame, maxFrames);
  char **strings = backtrace_symbols(frame, nptrs);   // 字符串数组
  if(strings == nullptr){
    return;
  }

  for(int i = 0; i < nptrs; ++ i){
    std::cout << strings[i] << '\n';
  }

  free(strings);
}

int main(){
  test_backtrace();

  return 0;
}
```
编译执行如下：
```
$ g++ -o test.out -rdynamic backtrace.cpp
$ ./test.out
./test.out(_Z14test_backtracev+0x38) [0x7f4819400bb2]
./test.out(main+0x9) [0x7f4819400c5a]
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xe7) [0x7f4818691b97]
./test.out(_start+0x2a) [0x7f4819400a9a]
```
以“./test.out(_Z14test_backtracev+0x38) [0x7f4819400bb2]”为例解析：./test.out是可执行文件名，括号中隐约可以看出是test_backtrace函数名+偏移地址，后面中括号中是堆栈返回地址  
下面解释括号中“奇怪的函数名”和如何将其还原

## mangle和demangle
C++程序在编译时，程序中的标识符（变量名、函数名）会被编译器修改，转换成C++ ABI(Application Binary Interface, 应用二进制接口)标识符，这个过程称为mangle；相反由C++ ABI标识符转换成源码中标识符的过程称为demangle **[3]**。
所以“_Z14test_backtracev”这样奇怪的函数名其实是C++ ABI标识符，根据编译器g++的命名规则解析如下：_Z开头表示函数，后面14表示函数名长度，test_backtrace则是真的函数名，最后的v表示返回值是void  

使用<cxxabi.h>中提供的abi::__cxa_demangle()进行demangle
``` C++
#include <cxxabi.h>
char* abi::__cxa_demangle(const char *mangled_name, char *output_buffer, size_t *length, int *status)
```
将上面test_backtrace()补充成如下：
``` C++
void test_backtrace(){
  const int maxFrames = 100;                            // 堆栈返回地址的最大个数
  void *frame[maxFrames];                               // 存放堆栈返回地址
  int nptrs = backtrace(frame, maxFrames);
  char **strings = backtrace_symbols(frame, nptrs);     // 字符串数组
  if(strings == nullptr){
    return;
  }

  size_t len = 256;           
  char *demangled = static_cast<char*>(::malloc(len));  // 用于存放demangle之后的结果
  for(int i = 0; i < nptrs; ++ i){
    char *leftPar = nullptr;                            // 左括号
    char *plus = nullptr;                               // 加号
    for(char *p = strings[i]; *p; ++ p){                // 找到左括号和加号的位置，两者之间的内容是需要demangle的
      if(*p == '(')
        leftPar = p;
      else if(*p == '+')
        plus = p;
    }

    *plus = '\0';
    int status = 0;
    char *ret = abi::__cxa_demangle(leftPar+1, demangled, &len, &status);
    *plus = '+';
    if(status != 0){
      std::cout << strings[i] << '\n';
    }
    else{
      demangled = ret;
      std::string stack;
      stack.append(strings[i], leftPar+1);
      stack.append(demangled);
      stack.append(plus);
      std::cout << stack << '\n';
    }
  }

  free(demangled);
  free(strings);
}
```
编译执行如下：
```
$ g++ -o test.out -rdynamic backtrace.cpp
$ ./test.out
./test.out(test_backtrace()+0x39) [0x7fb9c34013a3]
./test.out(main+0x9) [0x7fb9c340165f]
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xe7) [0x7fb9c2671b97]
./test.out(_start+0x2a) [0x7fb9c340128a]
```
可见成功将C++ ABI标识符转换成了源码中的标识符了，有了上述的预备知识之后，我们就可以着手封装一个具有堆栈跟踪的异常类了

## 封装具有堆栈跟踪功能的异常类
### Exception.h
``` C++
#ifndef EXCEPTION_H_
#define EXCEPTION_H_

#include <stdexcept>

class Exception : public std::runtime_error
{
public:
  Exception(const std::string &what);
  ~Exception() noexcept override = default;

  std::string stack_trace() const noexcept {
    return m_stack;
  }

private:
  void fill_stack_trace();

private:
  std::string m_stack;
};

#endif // EXCEPTION_H_
```

### Exception.cpp
``` C++
#include <cxxabi.h>     // __cxa_demangle
#include <execinfo.h>   // backtrace backtrace_symbols

#include "Exception.h"

Exception::Exception(const std::string &what) : 
  std::runtime_error(what)
{
  fill_stack_trace();
}

void Exception::fill_stack_trace(){
  const int maxFrames = 200;                            // 返回堆栈框架的最大个数
  void *frame[maxFrames];                               // 存放堆栈框架的的返回地址
  int nptrs = ::backtrace(frame, maxFrames);
  char **strings = ::backtrace_symbols(frame, nptrs);   // 字符串数组
  if(strings == nullptr){
    return;
  }

  size_t len = 256;           
  char *demangled = static_cast<char*>(::malloc(len));  // 用于存放demangled之后的结果
  for(int i = 1; i < nptrs; ++ i){
    char *leftPar = nullptr;                            // 左括号
    char *plus = nullptr;                               // 加号
    for(char *p = strings[i]; *p; ++ p){                // 找到左括号和加号的位置，两者之间的内容是需要demangle的
      if(*p == '(')
        leftPar = p;
      else if(*p == '+')
        plus = p;
    }

    *plus = '\0';
    int status = 0;
    char *ret = abi::__cxa_demangle(leftPar+1, demangled, &len, &status);
    *plus = '+';
    if(status != 0){
      m_stack.append(strings[i]);
    }
    else{
      demangled = ret;
      m_stack.append(strings[i], leftPar+1);
      m_stack.append(demangled);
      m_stack.append(plus);
    }

    m_stack.push_back('\n');
  }

  free(demangled);
  free(strings);
}
```
我们自定义的Exception类继承std::runtime_error，这样就对外提供了两个接口：what()用来查看异常信息，stace_trace()用来查看堆栈信息

### 测试
``` C++
#include <iostream>
#include <tools/base/Exception.h>

class ExceptionTest{
public:
  ExceptionTest(){
    throw Exception("hello world");
  }
};

void test_func(){
  ExceptionTest b;
}

int main(){
  try{
    test_func();
  }
  catch(Exception &e){
    std::cout << e.what() << '\n';
    std::cout << e.stack_trace() << std::endl;
  }
  
  return 0;
}
```

编译执行结果如下：
```
$ g++ -o Exception_test -rdynamic Exception_test.cpp
$ ./Exception_test
hello world
./Exception_test(Exception::Exception(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&)+0x4e) [0x7f7e06a020f4]
./Exception_test(ExceptionTest::ExceptionTest()+0x5d) [0x7f7e06a01fe5]
./Exception_test(test_func()+0x23) [0x7f7e06a01ddd]
./Exception_test(main+0x1d) [0x7f7e06a01e11]
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xe7) [0x7f7e05c71b97]
./Exception_test(_start+0x2a) [0x7f7e06a01cda]
```

## 参考文献
* [1]. Linux man-pages：man 3 backtrace
* [2]. [【C】使用backtrace获取堆栈信息](https://blog.csdn.net/iEearth/article/details/49763481) -- aidear_evo
* [3]. [mangle和demangle](https://www.cnblogs.com/robinex/p/7892795.html) -- 巴黎河畔