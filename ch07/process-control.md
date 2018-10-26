## 进程控制

进程控制是一个比较困难的部分，因为要控制进程的运行，就要理解进程的运行方式，系统对进程的调度方面的一些基础知识，进程之间的信号机制。



### 创建子进程

fork系统调用可用于创建子进程，fork的调用很简单，fork没有参数，出错的话返回值是-1，成功执行会在父进程返回子进程的PID，子进程返回0。

| 头文件 | \#include <sys/types.h><br>\#include <unistd.h>          |
| ------ | -------------------------------------------------------- |
| 原型   | pid_t  fork(void);                                       |
| 参数   | 无参数                                                   |
| 返回值 | 出错返回-1，否则在父进程返回子进程的pid，在子进程返回0。 |

开始接触到fork可能会有些疑惑，实际上，在fork执行过程中，一个新的进程就被系统创建了，新的进程完全拥有父进程的数据，但是会在父进程调用fork的代码出继续执行，此时，父进程中的fork调用会返回一个值，就是子进程的pid，在子进程中返回0。由于返回值的不同，代码可以通过判断返回值控制父进程和子进程执行不同的逻辑。

创建子进程示例：

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

int main(int argc, char *argv[]) {
    
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return 1;
    }
	/*
		根据返回值的不同，父进程和子进程可以执行不同的代码
    */
    if(pid > 0) { //父进程执行，getpid用于获取进程自己的PID
        printf("Parent: %d fork child [%d]\n", getpid(), pid);
    } else { //子进程执行
        printf("Child: %d\n",getpid());
    }

    return 0;
}
```

实际上Linux/Unix的内核为了提高性能，并不会马上就把所有的数据都复制出来，这会导致大量性能损耗，fork创建的子进程往往并不需要父进程的全部信息，所以多个进程共享同样物理空间，只有当子进程更改数据时才会复制对应段的数据。

对于用户来说，这些都是透明的，你只需要按照fork的逻辑去理解并编码。其他的内核会帮你解决。

### 等待子进程退出

之前的程序在fork调用之后，两个进程就交给系统进行调度，在执行过程中，两个进程就是交替运行。

现在要让父进程等待子进程退出后再退出，这需要一个系统调用：wait。

| 头文件 | \#include <sys/types.h><br>#include <sys/wait.h> |
| ------ | ------------------------------------------------ |
| 原型   | pid_t  wait(int *wstatus);                       |
| 参数   | 指针：int*，保存子进程的退出状态码               |
| 返回值 | 正确则返回子进程的PID，出错返回-1                |

wait调用示例：

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
    
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return 1;
    }

    if(pid > 0) {
        printf("Parent: %d fork child [%d]\n", getpid(), pid);
        int st = 0;
        /*
        	在子进程退出之前，父进程不会输出，wait会阻塞父进程
        */
        printf("child %d exit code: %d\n",wait(&st),st);
    } else {
        printf("Child: %d\n",getpid());
        sleep(2); //延迟2秒
    }

    return 0;
}
```

这种情况是子进程不退出，父进程只能阻塞，直到wait返回。类似的调用还有一个是waitpid：

| 头文件 | \#include <sys/types.h><br>#include <wait.h>                 |
| ------ | ------------------------------------------------------------ |
| 原型   | pid_t  waitpid(pid_t pid, int* wstatus, int options);        |
| 参数   | pid：进程的PID；wstatus：子进程退出状态保存在此；options：选项 |
| 返回值 | 成功返回退出进程的pid，如果设置了WNOHANG选项或没有子进程退出返回0， 错误返回-1。 |

waitpid的参数pid可以是下列值：

| 值     | 说明                                         |
| ------ | -------------------------------------------- |
| 小于-1 | 等待所有组ID是pid绝对值的进程退出            |
| -1     | 等待所有子进程退出                           |
| 0      | 等待所有组ID和调用进程的组ID相同的子进程退出 |
| 大于0  | 等待指定PID的进程退出                        |

waitpid的options的常用选项是WNOHANG，表示如果没有子进程退出，则马上返回，而不是阻塞。

调用wait相当于调用：waitpid(-1, &status,0);

**处理SIGCHLD信号**

子进程退出时，父进程会收到SIGCHLD信号，此时父进程通过waitpid可以知道哪个进程退出，通过选项WNOHANG以及两个宏定义函数：WIFEXITED、WIFSIGNALED。WIFEXITED(status)检测退出状态如果是正常退出则返回true；WIFSINALED(status)检测退出状态如果是被信号中断则返回true。

接下来要给出的示例仍然是等待子进程退出后，父进程再退出，但是在子进程退出之前，父进程会做其他的事，当然这里给出的示例程序是每秒输出一条信息。这种异步的操作方式是使用信号实现的，通过捕获SIGCHLD信号，当子进程退出时，父进程可以捕获这个信号。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

void handle_sigchld(int sig) {
    int st;
    pid_t pid;

    /*
    	设置WNOHANG选项，不会导致阻塞。使用循环获取所有退出的进程
    */
    while((pid=waitpid(0, &st, WNOHANG)) > 0) {
        if (WIFEXITED(st)) { //如果子进程正常退出
            printf("child %d exited\n", pid);
        } else if (WIFSIGNALED(st)) { //如果子进程被信号中断
            printf("child %d terminate by signal\n", pid);
        }
        exit(0);
    }
}

int main(int argc, char *argv[]) {
    
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return 1;
    }

    if(pid > 0) {
        printf("Parent: %d fork child [%d]\n", getpid(), pid);
        signal(SIGCHLD, handle_sigchld); //设置SIGCHLD信号处理函数
        /*
        	在子进程退出之前，父进程每秒输出一条信息
        */
        while (1) {
            printf("waiting child exit...\n");
            sleep(1);
        }
    } else {
        printf("Child: %d\n",getpid());
        sleep(5);
    }

    return 0;
}
```



### 创建指定数量的进程并使用信号控制

当调用fork的时候，会创建子进程，此时如果不做具体的控制，子进程和父进程都会继续执行后面的代码，所以要创建指定数量的子进程，就要根据返回值进行控制，父进程在一个循环里负责fork子进程，子进程跳出循环执行自己的代码。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <wait.h>

/*
	用于标记进程类型：父进程，子进程，未设置
*/
#define PCS_UNSET   0
#define PCS_PARENT  1
#define PCS_CHILD   2

//此标记的变量fork出的进程都会有，每个进程设置自己的标记
int process_flag = PCS_UNSET;

int main(int argc, char *argv[]) {
    
    process_flag = PCS_PARENT;

    int child = 3;
    pid_t pid;

    for(int i=0; i<child; i++) {
        pid = fork();

        if (pid<0) { //fork出错
            perror("fork");
            kill(0, SIGTERM); //向所有子进程发送SIGTERM信号中断子进程
            return -1;
        }

        if (pid>0) {
            continue;
        } else {
            //子进程设置标记后跳出循环
            process_flag = PCS_CHILD;
            break; 
        }
    }
    
    if (process_flag == PCS_PARENT) {
        printf("parent:%d\n", getpid());
        
        int status = 0;
        for(int i=0;i<child; i++) {
            printf("child %d, exit code: %d\n",wait(&status), status);
        }
        
    } else if (process_flag == PCS_CHILD) {
        pid_t myid = getpid();
        printf("child:%d\n",myid);
        sleep(myid % 5); //挂起时间：进程PID对5求余数作为秒数
    }

    return 0;
}
```

现在看一个稍加改动的示例，下面的程序在fork出3个子进程后，父进程输出一条信息后，调用sleep挂起5秒钟，之后使用kill(0, SIGTERM);对所有子进程发送SIGTERM信号，当然父进程自己也会收到信号；在子进程中，注册信号处理函数，之后每隔一秒输出一条信息，如果遇到SIGTERM信号则输出接收者的信息以及信号信息然后退出。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <signal.h>
#include <wait.h>

#define PCS_UNSET   0
#define PCS_PARENT  1
#define PCS_CHILD   2

int process_flag = PCS_UNSET;

void handle_sig(int sig) {
    printf("%d get signal: %u  ", getpid(), sig);
    switch (sig) {
        case SIGINT:
            printf("SIGINT -> exit...\n");
            exit(0);
        case SIGTERM:
            printf("SIGTERM -> exit...\n");
            exit(0);
        default:;
    }
}

int main(int argc, char *argv[]) {
    
    process_flag = PCS_PARENT;

    int child = 3;
    pid_t pid;

    for(int i=0; i<child; i++) {
        pid = fork();

        if (pid<0) {
            perror("fork");
            kill(0, SIGTERM);
            return -1;
        }

        if (pid>0) {
            continue;
        } else {
            process_flag = PCS_CHILD;
            break;
        }
    }
    if (process_flag == PCS_PARENT) {
        printf("parent %d will kill all child\n ", getpid());
        sleep(5);
        kill(0, SIGTERM); //向所有子进程发送SIGTERM信号
    } else if (process_flag == PCS_CHILD) {
        pid_t myid = getpid();
        signal(SIGTERM, handle_sig); //注册信号处理函数
        signal(SIGINT, handle_sig);
        while(1) {
            printf("child %d say : hello liux.\n", myid);
            sleep(1);
        }
    }

    return 0;
}
```

**kill(-1, SIGTERM)要慎用，这个操作会向所有具备权限的进程发送SIGTERM信号。**比如此进程所属用于ID是128，则所有用户ID是128的进程都会收到，运行完之后你会发现很多其他进程都退出，可能你还要重新登录。



### 如何实现守护进程

普通的进程在shell退出后，就随之退出，因为在shell中运行命令时，shell作为父进程，登陆后会开启新的会话，shell作为会话的session leader（会话中的第一个进程），shell退出后，所有属于这个会话的进程都会退出。

守护进程和普通的进程不同，守护进程的父进程一般是1，并且在系统执行期间不会退出，或者完成指定的工作退出。在shell中运行后，也不会随着shell退出而退出。

编写服务类软件要在运行后作为守护进程执行，这要求脱离shell开启的会话控制。

**父进程在fork子进程之后退出，此时子进程没有退出，这时候子进程会被init（PID为1）进程接管，init成为其父进程。**这个设计是实现守护进程的基础。之后调用setsid就可以开启一个新的会话，并且自己就是session leader。

实现守护进程的主要工作就是这些，往往还需要切换工作目录，重定向IO等操作。

守护进程示例：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <time.h>
#include <string.h>

int main(int argc, char *argv[]) {
    
    int pid = fork();
    if (pid < 0) {
        perror("fork");
        return -1;
    }

    if (pid>0) {
        //父进程退出，让子进程被init接管
        return 0;
    }

    chdir("/"); //工作目录切换到/
    setsid(); //创建新的会话

    /*
    	把0，1，2都重定向到/dev/null
    	不会输出任何信息，也不接受输入
    */
    int fd = open("/dev/null", O_RDWR);
    dup2(fd, 0);
    dup2(fd, 1);
    dup2(fd, 2);
    close(fd);
    

    //创建文件做记录
    fd = open("/tmp/dserv", O_CREAT|O_APPEND|O_RDWR, 0644);
    if (fd<0) {
        return -1;
    }
    
    char *tm = NULL;
    time_t t;
    int count = 0;
    while(1) {
        time(&t);
        tm = ctime(&t);
        write(fd, tm, strlen(tm));
        count++;
        sleep(2+count/3);
    }

    return 0;
}
```

程序编译后的名字是daeserv。

程序运行后，会发现，终端不会等待进程退出，而是立即返回，因为程序fork出子进程之后，立即退出，子进程已经被init接管。运行：

ps -e -o user,pid,ppid,tty,comm,args | grep  daeserv  |  grep -v grep

发现进程的父进程ID时1，终端列显示?表示不属于控制终端。

cat  /tmp/dserv会显示守护进程记录的信息。



**练习**

1. 如何让父进程控制子进程的运行时间？
2. 如何维持指定数量的子进程，在子进程被kill以后，父进程能够检测到并创建新的子进程补充？