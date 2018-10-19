## 信号处理

信号对系统来说是非常重要的一部分，这部分内容，我们关注的重点在于编程应用。信号的实现细节非常复杂。简单来说，信号提供了一个异步处理机制。一个进程不必实时检测某些事件的发生，而是做其他处理，当某些事件发生时，内核会向进程发送信号，进程会响应这个信号。除了信号这个词语以外，可能你还会见到过异常和中断这两个术语。关于信号，有这样几点，是需要知道的：

1. 软件层面，程序只是产生信号请求，真正向进程发送信号的是内核。
2. 硬件的错误被称为异常，其他一些硬件信号被称为中断。这是硬件层面的信号。
3. 内核定义了一组信号常量表示不同的信号事件。
4. 系统内核有默认的信号处理方式，程序也可以自定义信号的处理。
5. 无论是硬件层面还是软件层面的信号，都是由内核统一处理，提供统一的接口。

信号的处理方式有三种：

1. 忽略该信号
2. 捕获并处理
3. 系统默认的处理方式

信号的名称都是 SIG加上名字缩写的宏定义。头文件是signal.h，Linux的头文件结构中，signal.h并没有直接定义信号，而是引入了bits/signum.h文件。命令 kill -l 可以查看信号的名称以及数字。man 7 signal给出了详细的说明。

信号的产生来自于这几种形式：

1. 用户键盘输入
2. 硬件产生信号被内核捕获
3. 系统内核直接发送给进程
4. 程序通过系统调用发送信号

通过kill -l发现，没有32，33。从SIGRTMIN开始，是SIGRTMIN+数字，而SIGRTMIN是34，没有32，33是因为glibc的线程库使用了这两个信号。前31个信号是普通的信号，从SIGRTMIN开始，后面的是运行时信号。我们这里先不关注运行时信号，简单来说，运行时信号也称为可靠信号，因为普通的信号可能会丢失。运行时信号多个信号会排队，相同的多个信号到达可以被合并成一个。

### 处理SIGINT以及SIGTERM信号

SIGINT信号是用户在终端输入Ctrl+C产生信号，用于强制中断进程。SIGINT信号可以被程序捕获。

SIGTERM是kill命令默认发送的信号，默认动作是中断进程。SIGTERM信号也可以被程序捕获。

用于注册信号处理方式的函数是signal：

| 头文件 | \#include <signal.h>                                         |
| ------ | ------------------------------------------------------------ |
| 原型   | typedef  void(*sighandler_t) (int);<br>sighandler_t signal(int signum, sighandler_t handler); |
| 参数   | signum：注册要处理的信号；handler：信号处理函数              |
| 返回值 | 成功以后返回的是指向之前的信号处理函数的函数指针，出错返回SIG_ERR |

这个函数看起来非常复杂，但是使用起来很简单。

以下代码忽略SIGINT和SIGTERM的信号：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>

int main(int argc , char *argv[]) {

    //忽略SIGINT和SIGTERM信号
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

//要处理信号的函数必须是接收一个int类型的参数，返回值为void
void handle_sig(int sig) {
    printf("signal number: %d\n", sig);
    /*
       	在实际的应用场景中可以做一些清理工作，比如，
       	关闭打开的文件，释放申请的内存，写入日志等。
    */
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

    //注册函数捕获SIGINT和SIGTERM并处理
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

这两个程序是最简单的信号处理方式的演示，基本的逻辑就是这样的，捕获信号以后的处理根据具体场景去编写代码。



### 处理时钟信号

先来看一段程序并猜测它的运行结果：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>

int main(int argc, char *argv[])
{
    alarm(2); //设置2秒后，内核会向进程发送一个SIGALRM信号
    pause(); //挂起进程，直到SIGALRM信号到达
    printf("I am a programmer\n");

    return 0;
}
```

通过程序的整体逻辑以及注释来看，似乎是程序挂起等待2秒后，输出一条信息。但是测试的结果是程序等待2秒后直接退出，在本机测试的结果是，响应了一条信息：Alarm clock。

发生这种情况的原因是由于SIGALRM信号默认是杀死进程的，除非程序注册了函数用于处理SIGALRM信号。调用alarm相当于设置了一个闹钟，到时间后通知程序。

以下是alarm函数的原型：

```c
#include <unistd.h>
unsigned int alarm(unsigned int seconds);
```

函数的返回值是前一个设置闹钟的剩余秒数，如果返回0表示没有闹钟设置。比如代码：

```c
unsigned int sd = alarm(5);
printf("%u\n",sd);
```

输出是0，因为在alarm(5)之前没有设置闹钟。

考虑以下程序，会出现什么结果？

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>

int main(int argc, char *argv[])
{
    unsigned int sd = alarm(2); //设置2秒后，内核会向进程发送一个SIGALRM信号
    
    sd = alarm(0);
    
    printf("%u\n", sd); //sd=?
    
    pause(); //挂起进程，直到SIGALRM信号到达
    
    printf("I am a programmer\n");//何时输出？

    return 0;
}
```

sd的值是2，由于alarm(0)取消了闹钟设置。所以pause()会一直等待时钟信号的到来，注意，我们没有在程序里注册处理SIGALRM信号，所以"I am a programmer"不会输出，因为进程在收到SIGALRM信号之后会退出。但是这个时候如何给进程一个SIGALRM信号？

>程序运行之后，会一直等待时钟信号，通过ps命令发现进程的PID是2662，这是在机器上的结果，你要确定你的进程运行时的PID。命令kill -l查看SIGALRM的数值是14，所以只需要运行命令：
>
>kill  -14   2662
>
>进程会因为收到SIGALRM信号而退出

接下来的示例程序每秒输出一次时间。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <time.h>

void alarm_do(int sig) {
    time_t tm = time(NULL);
    
    //把时间格式化成字符串并输出
    printf("%s\n", ctime(&tm));
}

int main(int argc, char *argv[]) {

    signal(SIGALRM, alarm_do); //设置信号处理函数
    while(1) {
        
        //每隔一秒执行alarm_do
        alarm(1);
        pause();
    }

    return 0;
}
```

需要处理定时任务的程序要设置时钟信号的处理方式，比如两个服务进程，一个负责处理网络请求，另一个检测请求列表中是否存在超时未传输数据，是否在指定时间内发送验证消息等情况。

### 向进程发送信号

Linux系统的kill命令可以向其他进程发送信号，这种功能的实现依赖于系统的一个调用：kill。和kill命令同名。kill的调用方式很简单：

| 头文件 | \#include <sys/types.h><br>\#include <signal.h> |
| ------ | ----------------------------------------------- |
| 原型   | int kill(pid_t pid, int sig);                   |
| pid    | 要发送信号的进程ID                              |
| sig    | 信号值                                          |
| 返回值 | 成功返回0，出错返回-1                           |

如果进程的PID是2562，要向它发送SIGTERM信号，只需要kill(2562, SIGTERM);。注意如果2562是root用户的进程，其他普通用户调用会失败，因为不具备权限。

进程向自己发送信号：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <sys/types.h>

void handle_sig(int sig) {
    printf("get signal: %d\n",sig);
    exit(0);
}

int main(int argc, char *argv[]) {
    
    signal(SIGTERM, handle_sig); //设置信号处理函数

    printf("waiting signal...\n");
    sleep(1);

    kill(getpid(), SIGTERM); // 向自己发送SIGTERM信号
    
    sleep(5);

    return 0;
}
```

以上是简单的示例，在调用kill发送信号以后，尽管有sleep(5);但是handle_sig在输出信息后执行了exit(0);退出程序。

pid参数不仅仅可以传递进程的PID，它可以是以下形式：

| pid参数 | 说明                                                         |
| ------- | ------------------------------------------------------------ |
| 大于0   | 信号传递给指定的PID                                          |
| 等于0   | 信号传递给当前调用进程的组内所有进程，比如进程有3个子进程，这3个子进程都会收到信号。 |
| -1      | 信号传递给当前调用进程具备权限的所有进程（init进程除外）。   |
| 小于-1  | 信号将会发送给组ID是 -pid 的进程。                           |

kill命令的简单实现：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <sys/types.h>


int main(int argc, char *argv[]) {

    if (argc < 2) {
        dprintf(2, "error: usage killone [SIGNAL] [PID]\n");
        return -1;
    }

    int sig = SIGTERM; //默认发送SIGTERM信号
    int pid = 0;
    //参数处理
    for(int i=1; i<argc; i++) {
        if (strncmp(argv[i], "--signal=",9)==0) { //检测是否指定了信号
            sig = atoi(argv[i]+9);
            if (sig<0 || sig >64) {
                sig = SIGTERM;
            }
        } else { //记录要发送信号的PID
            pid = atoi(argv[i]);
        }
    }

    //如果pid事进程自己的或者是<=1则不合法
    if (pid == getpid() || pid <= 1) {
        dprintf(2, "Error: pid illagel\n");
        return -1;
    }

    if (kill(pid, sig)==-1) {
        perror("kill");
        return -1;
    }

    return 0;
}
```

关于kill的更多用法会在进程控制一章讲述。



**!可靠信号作为选修内容，这里要说明的是，可靠信号的机制兼容旧的信号模式，实际上signal的实现已经是基于可靠信号的处理方式来实现的，对接口调用来说，signal依旧封装成原来的调用方式，这样可以兼容原来的代码。**



### *可靠信号

Linux的信号机制继承自Unix的设计方式，早期的Unix的信号机制功能比较弱，那时候使用信号就像是捕鼠器，使用一次之后，捕获到信号就要再设置，比如处理SIGINT，SIGTERM，SIGALRM信号的函数，要设计成以下方式：

```c
//早期的信号处理函数的使用方式
void handle_sig(int sig) {
    /*
    	每次捕获信号以后还要重新设置
    */
 	signal(SIGINT, handle_sig);
    signal(SIGTERM, handle_sig);
    signal(SIGALRM, handle_sig);
    
    switch (sig) {
            //...
            //...
    }
    
}
```

设置信号是需要时间的，如果操作复杂，可能还涉及到多个程序，这时候，如果还来不及重新设置，新的信号又到达了，这时候就会执行默认操作，程序可能因此中断，数据可能不能同步。

原始的信号机制还存在另一个问题：用于设置信号处理的函数只能是void  handle_sig(int sig);的形式，并且当信号发生的时候，系统也仅仅是通知了进程，获取到的信息就是哪个信号被捕获。信号的来源是不清楚的。

针对这些问题，很多组织都开发出了不同的解决方案，然而被广泛支持并且比较好的是POSIX的模型，新的方案中，原来的信号处理模式依旧被支持。最新的方案使用sigaction函数进行信号的设置。其具体信息如下：

| 头文件 | #include<signal.h>                                           |
| ------ | ------------------------------------------------------------ |
| 原型   | int sigaction(int sig, const struct sigaction * act, struct sigaction *oldact); |
| 参数   | sig       信号的值<br>act       新设置的信号处理结构<br>oldact  保存旧的处理结构 |
| 返回值 | 成功设置返回0，出错返回-1                                    |

struct sigaction结构信息如下：

```c
struct sigaction {
    void (*sa_handler)(int);
    void (*sa_sigaction) (int, siginfo_t *, void *);
    sigset_t sa_mask;
    int sa_flags;
}
```

sa_handler用于旧的处理模式，sa_sigaction用于新的处理模式，那系统如何知道你要使用新的模式还是旧的模式？

这需要使用sa_flags来设置。sa_flags一些选项如下：

| 选项         | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| SA_RESTHAND  | 当信号处理函数被调用时重置，就是采用捕鼠器模式               |
| SA_NOCLDWAIT | 当子进程退出时不会产生僵尸进程                               |
| SA_RESTART   | 一般在执行系统调用返回之前，信号不会被发送，但是针对一些慢速的设备的操作除外，比如硬盘，U盘的读写。这时候系统调用会返回-1，此选项说明不返回而是重新系统调用。 |
| SA_SIGINFO   | 设置这个位表示使用sa_sigaction否则使用sa_handler             |

sa_mask用于指定在处理一个信号的时候要阻塞那些信号，这可以防止数据被损毁。

现在来看一个完整的示例：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
#include <signal.h>
#include <fcntl.h>
#include <time.h>

#define DATA_BUF_LEN    1024

//要处理信号的函数
void handle_sig(int sig) {
    printf("get signal: %d\n", sig);
    sleep(1);
    time_t tm = time(NULL);
    printf("done : %s\n", ctime(&tm));
}

int main(int argc, char *argv[]) {

    sigset_t blocked;
    
    struct sigaction sighandle;
    
    char buffer[DATA_BUF_LEN+1];
    
    sigemptyset(&blocked); //初始化要设置的sa_mask所有位

    sigaddset(&blocked, SIGQUIT); //添加阻塞SIGQUIT信号，Ctrl+\会产生此信号

    sighandle.sa_mask = blocked; //设置好的值给sa_mask

    /*
    	设置处理函数，sa_flags没有设置SA_SIGINFO，
    	所以采用旧的处理模式
    */
    sighandle.sa_handler = handle_sig; 

    if (sigaction(SIGINT, &sighandle, NULL) == -1) {
        perror("sigaction");
        return -1;
    }

    while (1) {
        fgets(buffer, DATA_BUF_LEN, stdin);
        printf("    %s", buffer);
    }

    return 0;
}
```

程序运行后，快速连续按Ctrl+C，Ctrl+\\。会发现所有的Ctrl+C处理完之后才会处理Ctrl+\\产生的SIGQUIT信号，此信号会产生核心转储文件（core dumped，进程在内存中的映像，会把这些数据存储到文件，用于调试使用）。
