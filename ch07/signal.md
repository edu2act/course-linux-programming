## 信号处理

信号对系统来说是非常重要的一部分，这部分内容，我们关注的重点在于编程应用。信号的实现细节非常复杂。简单来说，信号提供了一个异步处理机制。一个进程不必实时检测某些事件的发生，而是做其他处理，当某些事件发生时，内核会向进程发送信号，进程会响应这个信号。除了信号这个词语以外，可能你还会见到过异常和中断。有这样几点，是需要知道的：

1. 软件层面，程序只是产生信号请求，生成信号的是内核。
2. 硬件的错误被称为异常，其他一些硬件信号被称为中断。这是硬件层面的信号。
3. 内核定义了一组信号常量表示不同的信号事件。
4. 系统内核有默认的信号处理方式，程序员也可以自定义信号的处理。

信号的处理方式有三种：

1. 忽略该信号
2. 捕获并处理
3. 系统默认的处理方式

信号的名称都是 SIG加上名字缩写的宏定义。头文件是signal.h，Linux的头文件结构中，signal.h并没有直接定义信号，而是引入了bits/signum.h文件。命令 kill -l 可以查看信号的名称以及数字。man 7 signal给出了详细的说明。

产生信号的请求来自于这几种形式：

1. 用户键盘输入
2. 硬件产生信号
3. 系统内核直接发送给进程
4. 程序通过系统调用发送信号

通过kill -l发现，没有32，33。从SIGRTMIN开始，是SIGRTMIN+数字，而SIGRTMIN是34，没有32，33是因为glibc的线程库使用了这两个信号。前31个信号是普通的信号，从SIGRTMIN开始，后面的是运行时信号。我们这里先不关注运行时信号，简单来说，运行时信号也称为可靠信号，因为普通的信号可能会丢失。

### 处理SIGINT以及SIGTERM信号

SIGINT信号是用户在终端输入Ctrl+C产生信号，用于强制中断进程。SIGINT信号可以被程序捕获。

SIGTERM是kill命令默认发送的信号，默认动作是中断进程。SIGTERM信号也可以被程序捕获。

以下代码忽略SIGINT和SIGTERM的信号：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>

int main(int argc , char *argv[]) {

    signal(SIGINT, SIG_IGN);
    signal(SIGTERM, SIG_IGN);

    printf("pid: %d\n",getpid());

    while (1) {
        printf("signal test\n");
        sleep(1);
    }

    return 0;
}
```

注册自己的函数处理信号：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>

void handle_sig(int sig) {
    printf("signal number: %d\n", sig);
    switch (sig) {
        case SIGINT:
            printf("get signal: SIGINT\n");
            exit(0);
            break;
        case SIGTERM:
            printf("get signal: SIGTERM\n");
            exit(0);
            break;
        default:;
    }
    
}

int main(int argc , char *argv[]) {

    signal(SIGINT, handle_sig);
    signal(SIGTERM, handle_sig);

    printf("pid: %d\n",getpid());

    while (1) {
        printf("signal test\n");
        sleep(1);
    }

    return 0;
}
```





### 处理时钟信号



### 向进程发送信号



### 最小shell加入信号处理

