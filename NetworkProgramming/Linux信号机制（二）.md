## 什么是信号集？
收到一个信号之后执行信号处理函数，在执行信号处理函数的过程中如果又来了一个相同的信号，那么这个信号将会被阻塞，直到信号处理函数执行完之后，再响应被阻塞的信号，注意如果信号被阻塞期间又收到了该信号，那么多个信号的处理会被合并为1次  
比如正在执行SIGUSR1的信号处理函数，此时又收到了一个SIGUSR1信号，那么该信号将会被阻塞，直到信号处理函数执行完之后，再次执行该函数。如果在阻塞期间又收到了SIGUSER1信号，接下来只再调用一次SIGUSR1的信号处理函数  

信号集就是用来记录当前收到了哪个信号，会把当前信号的标志位置成“正在处理”，如果此时再收到该信号，那么信号就阻塞等待  
使用数据类型sigset_t表示信号集，在Linux中该类型是一个32位无符号整数，这是因为在Linux中定义了32种信号，每一个信号用32位无符号整型变量中的一位来标志，如果该位置为1，那么表示正在处理该信号，如果置为0表示可以处理该信号。注意：有的操作系统，信号个数多于32，此时就不能用32位整数表示一个信号集了

## 操作信号集的函数
函数原型如下：**[1]**
``` C
#include <signal.h>

// @brief 清空信号集，全部位置0
int sigemptyset(sigset_t *set);

// 全部位置1
int sigfillset(sigset_t *set);

// 将signo信号对应位置1
int sigaddset(sigset_t *set, int signum);

// 将signo信号对应位置0
int sigdelset(sigset_t *set, int signum);

// 判断该信号集中是否包含指定的信号
int sigismember(const sigset_t *set, int signum);
```
再次强调信号集的作用，信号集一般用于一个进程对收到信号的过滤  
如果使用sigemptyset()初始化信号集（信号集的所有位置0），此时可以接收到所有的信号  
如果使用sigfillset()初始化信号集（信号集的所有位置1），此时屏蔽所有信号，当然SIGKILL和SIGQUIT信号是不能屏蔽的  
sigaddset()和sigdelset()的意义在于指定屏蔽某信号或者接收某信号  

## sigprocmask函数
该函数就是用来检测和修改当前进程的信号屏蔽字（使用信号集作为信号屏蔽字），函数原型如下：**[2]**
``` C
/*
 * @param how 如何修改当前信号屏蔽字
    * SIG_SETMASK: 直接将第二个参数的信号集进行关联
    * SIG_BLOCK: 当前要设置的信号集与进程目前关联的信号集取并集
    * SIG_UNBLOCK: 取交集
 * @param newSet 要关联的信号集，如果newSet是个空指针（表示只是要获取当前进程的进程屏蔽字），此时how的值无意义
 * @param oldSet 存放当前进程的信号屏蔽字
 * @return 执行成功返回0，否则返回-1
 */
int sigpromask(int how, const sigset_t* newSet, sigset_t *oldSet);
```
注意：sigprocmask是仅用在单线程的进程环境，对于多线程的进程使用pthread_sigmask()
示例：
``` C++
#include <stdio.h>
#include <signal.h>
#include <stdlib.h> // exit
#include <unistd.h> // sleep


void sig_quit(int signo){
    printf("收到了SIGQUIT信号!\n");
}

int main(){
  signal(SIGQUIT, sig_quit);                  // 注册SIGQUIT信号处理函数，SIGQUIT信号可以通过Ctrl+\触发

  sigset_t newSet, oldSet;                    // 定义两个信号集变量
  sigemptyset(&newSet);                       // 清空信号集，全部位置0
  sigaddset(&newSet, SIGQUIT);                // 将SIGQUIT对应的位置1，表示要屏蔽SIGQUIT信号

  sigprocmask(SIG_BLOCK, &newSet, &oldSet);
  printf("进程关联的信号集设置为newSet\n");

  if(sigismember(&newSet, SIGQUIT)){
    printf("SIGQUIT信号被屏蔽了，按ctrl+\\测试\n");
  }

  printf("sleep 10s\n");
  sleep(10);
  printf("sleep end\n");

  sigprocmask(SIG_SETMASK, &oldSet, NULL);
  printf("进程关联的信号集重新设置为oldSet，SIGQUIT信号没有被屏蔽，按ctrl+\\测试\n");
  printf("sleep 10s\n");
  sleep(10);

  return 0;
}
```

## sigaction函数
该函数用来查看和设置某一信号的处理函数。由于signal函数存在兼容性问题，所以实际开发中使用sigaction函数代替signal函数。函数原型如下：**[3]**
``` C++
#include <signal.h>

/*
 * @param signum: 指定要查看和设置的哪个信号的处理函数
 * @param act: 如果非空，表示修改信号的处理函数
 * @param oact: 如果非空，可以通过该参数得到信号之前的处理函数
 * @return 成功返回0，否则返回-1
 */
int sigaction(int signum, const struct sigaction *act,
              struct sigaction *oldact);

struct sigaction {
  void     (*sa_handler)(int);                        // 信号处理函数地址，或者SIG_IGN、SIG_DFL
  void     (*sa_sigaction)(int, siginfo_t *, void *); // 替代的信号处理函数，如果sa_flags为SA_SIGINFO则执行该处理函数
  sigset_t   sa_mask;                                 // 执行信号处理函数时的信号屏蔽字
  int        sa_flags;                                // 选项
  void     (*sa_restorer)(void);                      // The sa_restorer field is not intended for application use
};
```
示例代码在介绍完sigsuspend函数后一并给出

## sigsuspend函数
该函数可以简单理解为阻塞等待信号发生，函数原型如下：**[4]**
``` C++
#include <signal.h>

/*
 * @param mask：先将mask替换当前进程的信号屏蔽字，之后阻塞进程等待信号发生
 * @return 总是返回-1，可以通过errno值判断是否出错，正常情况errno为EINTR
 */
int sigsuspend(const sigset_t *mask);
```
示例：
``` C++
#include <stdio.h>  // printf
#include <string.h> // memset
#include <signal.h> // signal 

void handler_sigusr1(int sig){
  printf("收到了SIGUSR1信号\n");  
}

int main(){
  sigset_t set;
  sigemptyset(&set);

  struct sigaction act;
  memset(&act, 0, sizeof(act));
  act.sa_handler = handler_sigusr1;
  act.sa_mask = set;

  if(sigaction(SIGUSR1, &act, NULL) < 0){
    perror("sigaction error");
  }

  while(1){
    sigsuspend(&set);
  }

  return 0;
}
```

介绍上述这些函数的目的是为了封装信号机制做知识铺垫，如何使用C++封装简单易用的信号机制，请见博客[封装信号机制]()

## 参考文献
* [1]. Linux man-pages：man 3 sigemptyset
* [2]. Linux man-pages：man 2 sigprocmask
* [3]. Linux man-pages：man 2 sigaction
* [4]. Linux man-pages：man 2 sigsuspend