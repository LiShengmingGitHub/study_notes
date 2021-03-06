# 信号概念
信号, 就是软件中断, 用于很多异步事件模型中. 每个信号都有一个名字, 以SIG开头, 如SIGALRM是闹钟信号. 在<signal.h>中定义, 不存在编号为0的信号, 有很多条件产生信号: 

1. 用户按下终端键时如按`CRTL + C`, 产生`SIGINT`信号.
1. 硬件异常. 如引用无用内存(`SIGSEGV`), 除数为0
1. 用户调用kill(1)发送给其它进程. 如我们控制台输入: kill -9 1234, 则会发送`SIGKILL`信号给1234的进程.
1. 进程调用kill(2)
1. 当检测到某种软件条件发生时, 通知有关进程产生信号. 如`SIGURG`(网络上接收到一个紧急数据名, 像TCP协议中的URG标志报文); `SIGALRM`到时间后也会产生一个信号.

对信号的处理有一般有如下三种方式:

1. 忽略此信号. 但有两种信息是不能忽略的,如`SIGKILL`和`SIGSTOP`. 就像我们要强制停止一个进程时, 输入`kill -9 PID`就会产生`SIGKILL`, 这个信号不能忽略.
1. 捕捉信息. 就像事件通知模型, 当发生这种信息, 我们调用一个用户函数来处理我们定义的一些动作. 有两个信号是不能捕捉并定义事件处理的, 就是上面说的`SIGKILL`和`SIGSTOP`. 这种处理方式是最常用的, 比如我们捕捉`SIGCHLD`来捕捉子进程终止时, 父进程触发事件处理函数, 可以调用waitpid来获得子进程的PID和退出状态, 可以做自己想做的事,也可以防止僵尸进程.
1. 默认系统动作.

有很多异常信号发生时都会产生core文件, 如除以0时. 
```c
int main(int argc, char *argv[]){
    float a = 10 / 0;
    printf("%f\n", a);
    return 0;
}
```
运行退出后, 会发现产生了一个`core`文件(我在ubuntu下要用`ulimit -c unlimited`打开core开关), 用`gdb --core=core`分析这个core:
```c
Core was generated by `./a.out'.
Program terminated with signal 8, Arithmetic exception.
```
可以看到是是由编号为`8`的信号导致退出的,描述为`Arithmetic exception`, 通过`kill -l`可以看到为`8`的信号为:`SIGFPE`, 就是除以0为错的信号.

# signal函数
```c
#include <signal.h>
void (*signal(int signo, void (*)(int)))(int); // 或成功返回之前的处理函数, 出错返回SIG_ERR
```
第一个参数是信号, 第二参数一个函数指针, 该函数的参数是一个整数(信号), 返回为void. 这个原型定义太复杂了, 我们可以用`typedef`改写下:
```c
typedef void Sigfunc(int);

// 现在可以这样定义原型
Sigfunc *signal(int, Sigfunc *);
```
对于typedef的这种高级用法, 参考: [c typedef](http://stackoverflow.com/questions/1591361/understanding-typedefs-for-function-pointers-in-c-examples-hints-and-tips-ple)

# 信号丢失
在早期版本中, 信号是不可靠的, 所谓不可靠, 即是会丢失. 因为每次注册捕捉信号函数, 只会执行一次, 所以要每次触发后, 要再次注册绑定信号处理函数.
```c
int sig_int_flag;
main(){
  int sig_int(); // 信号处理函数
  ...
  signal(SIGINT, sig_int); // 注册信号处理函数
  ...
  while(sig_int_flag == 0){ //检查是否触发过
    pause();
  }
}

sig_int(){
  signal(SIGINT, sig_int); // 重新注册
  sig_int_flag = 1; // 设置标识为触发过
}
```
如果观察这段程序会有一个问题, 如果在执行while的检查条件为true,并在执行pause()之前, 现在触发了一个信号, 然后在sig_int函数中改变标识sig_int_flag, 然后继续执行pause(), 现在可以永远阻塞在这里了(假设这个程序只会收到过一次信号). 出现这种问题的原理就是这个信号丢失了.

# 可重入函数
我们知道, 信号产生是异步的, 如果在程序正在执行的时候收到信号并处理, 在处理完之后再继续执行主程. 这里就有个问题要考虑, 在调用信号处理程序内部处理时, 会不会破坏之后继续执行的主程数据结构. 比如malloc就会有这个问题. 还有,在信号处理程序中可能会产生errorno变量值, 这也会改变主程中刚产生的值.

# kill 和 raise函数
```c
#include <signal.h>
int kill(pid_t pid, int signo);
int raise(int signo);
// 成功返回0, 出错返回-1
```
这两个函数功能是一样的, 唯一不同的是raise是给调用进程自身发送信号. 所以 `kill(signo) == raise(getpid(), signo)`

# alarm 和 pause 函数
```c
#include <unistd.h>
unsigned int alarm(unsigned int seconds); // 0 或者以前设置的时间余留秒数
```
`alarm`将会产生一个SIGALARM信号.
`pause`函数使当前进程挂起直到捕捉到一个信号.
```c
#include <unistd.h>
int pause(void); // 返回值-1, 并将errorno设置为EINTR
```

# 信号集
如果要表示多个信号, 用`sigset_t`类型表示. 关于`sigset_t`的函数有如下这些:
```c
#include <signal.h>
int sigemptyset(sigset_t *set);
int sigfillset(sigset_t *set);
int sigaddset(sigset_t *set, int signo);
int sigdelset(sigset_t *set, int signo);
int sigismember(const sigset_t *set, int signo);
```
这些函数看名称就知道是看什么用的, 基本上就是集合的运算, 具体实现就是位运算(要该类型支持所有信号总数).

# sigprocmask函数
```c
#include <signal.h>
int sigprocmask(int how, const sigset_t *restrict set, sigset *restrict oset); // 成功返回0, 出错返回-1
```
其中how的值可能为:

how | 说明
--- | --- | ---
SIG_BLOCK | 添加屏蔽信号|
SIG_UNBLOCK | 删除屏蔽信号
SIG_SETMASK | 重置屏蔽信号


























