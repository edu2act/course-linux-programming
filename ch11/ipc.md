## IPC

IPC（Inter Process Communication）是进程间通信的英文简称。IPC是进程之间传递信息的方法。多个进程之间的协作总要涉及到进程间通信，通过协作实现共享数据、同步数据、避免冲突、模块化分离等。进程间通信开始接触会觉得编写困难，程序比较复杂。随着理解深入，使用熟练，会发现这些功能是为了简化程序设计，提高可用性，实现解藕，保证数据安全避免冲突而出现的。



###  匿名管道

通常所说的管道经常指的就是匿名管道。匿名管道在两个进程间创建了一个通信机制。父子进程可以使用管道进行数据传输。匿名管道使用pipe调用：

```c
#include <unistd.h>

int pipe(int fd[2]);
```

原型说明了要引入的头文件以及调用方式，传递一个int数组，fd[0]用于读取，fd[1]用于写入。父子进程间通信，父进程通过fd[1]写入数据，子进程可以通过fd[0]获取到。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

int main(int argc, char *argv[]) {

    int fd[2];

    if (pipe(fd) < 0) {
        perror("pipe");
        return -1;
    }

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        close(fd[0]);
        close(fd[1]);
    }

    char buffer[1000]   = {'\0'};
    //parent
    if (pid > 0) {
        printf("Parent: writing data to child\n");
        sleep(1);
        char *data = "Hello, child. I am parent.\n";
        write(fd[1], data, strlen(data));
        close(fd[0]);
        close(fd[1]);
    } else {
        //child
        int count = 0;
        int err = 0;
        printf("Child: waiting data...\n");
        count = read(fd[0], buffer, 100);
        if (count < 0) {
            perror("read");
            err = -1;
        } else {
            printf("GET MESAGE : %s", buffer);
        }

        close(fd[0]);
        close(fd[1]);

        return err;
    }

	return 0;
}

```



### 命名管道

pipe只能用于组内进程的通信，两个互不相关的进程通信可以使用命名管道。命名管道使用mkfifo：

```c
#include <sys/types.h>
#include <sys/stat.h>

int mkfifo(char*pathname, mode_t mode);

```

参数分别是要创建的路径名称，权限。mode权限和open函数相同。创建之后，可以使用open打开管道，并使用read和write进行读写。和操作普通文件一样。

创建管道并写入的代码示例：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>


int main(int argc, char *argv[]) {

    char *fifoname = "/tmp/fifo-test.fo";

    int fifo = mkfifo(fifoname, 0644);

    if (fifo < 0) {
        perror("mkfifo");
        return -1;
    }

    int fd = open(fifoname, O_WRONLY);
    if (fd < 0) {
        perror("open");
        return -1;
    }

    char *data = "时间具有相对性";
    write(fd, data, strlen(data)); //这种打开模式，写操作会阻塞直到数据被读取

    close(fd);

	return 0;
}


```

读取管道示例代码：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>


int main(int argc, char *argv[]) {

    char *fifoname = "/tmp/fifo-test.fo";

    int fd = open(fifoname, O_RDONLY);
    if (fd < 0) {
        perror("open");
        return -1;
    }

    char buffer[1000];
    int count = 0;

    count = read(fd, buffer, 1000);

    printf("%s\n", buffer);

    close(fd);

	return 0;
}

```



### \*消息队列

消息队列是一个队列，放入其中的数据符合先进先出的规则，Linux内核提供的系统调用保证了数据被顺序读取，不会出现混乱。

消息队列主要用于业务数据量大，或者是处理复杂，但是可以延迟处理的场景，比如用户注册，需要发送验证邮件，而发送验证邮件并是必须要及时响应的，用户注册后，通知用户查看邮箱的验证邮件即可。此时并没有真正的发送邮件，而是交给一个专门处理此任务的进程。如果用户注册数量比较多，就好比去银行办理业务，要排队一样，这些请求都被放入消息队列，监听此消息队列的进程顺序读取并发送邮件。

类似的还有其他需要发送邮件的场景，所以消息队列会提供更多的功能，比如支持消息分类，监听进程可以只选择获取某些类型的消息，这样多个进程可以监听消息队列中不同类型的消息，各自完成不同的任务，互不影响。

其他操作比如转账，也会把请求放入队列按顺序处理。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>
#include <signal.h>
#include <sys/wait.h>
#include <time.h>

#define MSG_INUM    1
#define MSG_STR     2
#define MSG_DNUM    3
#define MSG_MAX     4

#define MAX_NAME_LEN    2048

#define MSG_KEY     1248
//消息的具体类型
struct msg_content {
    int n;
    double d;
    char name[MAX_NAME_LEN];
};

//消息队列信息结构
struct msg_queue {
    long msg_type;
    struct msg_content msg;
};


#define MAX_RANDSTR_LEN    42

char * rand_str(int len) {
    
    char *total_str = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMPOPQRSTUVWXYZ0123456789";
    int tlen = strlen(total_str);

    static char rstr[MAX_RANDSTR_LEN+1] = {'\0',};

    for(int i=0; i<MAX_RANDSTR_LEN; i++)
        rstr[i] = '\0';

    if (len > MAX_RANDSTR_LEN)
        len = MAX_RANDSTR_LEN;
    else if (len <= 0)
        len = 5;

    time_t tm = time(NULL);
    srand(tm);
    for(int i=0; i<len; i++) {
        rstr[i] = total_str[rand() % tlen];
        srand(i+1 + tm);
    }

    return rstr;
}

int push_msg(int msg_type) {

    //获取消息队列，没有则创建
    int mid = msgget(MSG_KEY, IPC_CREAT|0664);

    if (mid < 0) {
        perror("msgget");
        return -1;
    }

    struct msg_queue msq;

    time_t tm = time(NULL);

    //对不同类型使用msgsnd发送不同的消息
    if (msg_type == MSG_INUM) {
        srand(msg_type);
        for (int i=0; i<100; i++) {
            msq.msg_type = MSG_INUM;
            msq.msg.n = rand() % 8192;
            srand(i + tm);
            if (msgsnd(mid, &msq, sizeof(msq.msg), 0) < 0) {
                perror("msgsnd");
            }
        }
    } else if (msg_type == MSG_DNUM) {
        srand(tm - 1);
        for(int i=0; i<100; i++) {
            msq.msg_type = MSG_DNUM;
            msq.msg.d = ((double)i) * 3.14 + (double)(rand() % 1024);
            if (msgsnd(mid, &msq, sizeof(msq.msg), 0) < 0) {
                perror("msgsnd");
            }
            srand(tm + i);
        }
    } else if (msg_type == MSG_STR) {

        for(int i=0; i<100; i++) {
            msq.msg_type = MSG_STR;
            strcpy(msq.msg.name, rand_str(((i*i)%41) + 3));
            if (msgsnd(mid, &msq, sizeof(msq.msg), 0) < 0) {
                perror("msgsnd");
            }
        }
    } else {
    
    }

    return 0;
}

int msq_recv(int msg_type) {

    int mid = msgget(MSG_KEY, 0664|IPC_CREAT);

    if (mid < 0) {
        return -1;
    }

    struct msg_queue msq;

    int msg_count = 1;

    while(1) {
        //接收消息队列的消息
        if (msgrcv(mid, &msq, sizeof(msq.msg), msg_type, 0) < 0) {
            break;
        }

        printf("%d--%-3d ", getpid(), msg_count++);
        if (msq.msg_type == MSG_INUM)
            printf("get msg : %d\n", msq.msg.n);
        else if (msq.msg_type == MSG_DNUM)
            printf("get msg : %.2lf\n", msq.msg.d);
        else if (msq.msg_type == MSG_STR)
            printf("get msg : %s\n", msq.msg.name);
    }

    return 0;
}


/*
    args :
        send [int|double|str|all]
        recv [int|double|str|all]
        msqrm
*/

#define HELP_INFO   "\
usage msq [INSTRUCTION] [MSG TYPE]\n\
    instruction : send recv msqrm\n\
    msg type [int|double|str|all]\n\
    example:\n\
        msq send all\n\
        msq send str\n\
        \n\
        msq recv str\n\
        msq msqrm\n\
"

void help() {
    printf("%s\n", HELP_INFO);
}

int main(int argc, char *argv[]) {

    if (argc < 2) {
        help();
        return 0;
    }

    if (argc < 3) {
        if (strcmp(argv[1], "msqrm") == 0) {
            int mid = msgget(MSG_KEY, 0664); 
            if (mid > 0 && msgctl(mid, IPC_RMID, NULL) < 0) {
                perror("msgctl");
                return -1;
            }
            return 0;
        }

        return -1;
    }

    int msg_type = 0;

    if (strcmp(argv[2], "all") == 0) {
        msg_type = 0;
    } else if (strcmp(argv[2], "int") == 0) {
        msg_type = MSG_INUM;
    } else if(strcmp(argv[2], "double") == 0) {
        msg_type = MSG_DNUM;
    } else if (strcmp(argv[2], "str") == 0) {
        msg_type = MSG_STR;
    } else {
        dprintf(2, "Error: unknow msg type\n");
        return -1;
    }

    if (strcmp(argv[1], "send")==0) {
        if (msg_type == 0) {
            push_msg(MSG_INUM);
            push_msg(MSG_DNUM);
            push_msg(MSG_STR);
        }
        else
            push_msg(msg_type);
    } else if (strcmp(argv[1], "recv") == 0) {
        msq_recv(msg_type);
    } else {
        dprintf(2 , "Error : unknow command\n");
        return -1;
    }

	return 0;
}


```



### \*信号量

信号量主要用于需要对资源进行互斥访问的场景，比如多个进程写入文件，需要控制按顺序写入，这需要对文件加锁，并写入，然后释放锁，只有获取锁定的进程才可以写入文件。信号量本质上是一个计数器。而且不能小于0。

使用信号量就是要保证访问的资源只能被某些进程访问，多数情况是二元信号量，只有0，1。释放信号量会是数字+1,获取会导致-1。在某一时刻，进程尝试获取信号量，如果成功，其他进程则无法获取。这个操作是原子操作，系统内核保证了这种操作的原子性。

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>
#include <signal.h>
#include <unistd.h>
#include <errno.h>
#include <sys/wait.h>

#define PCS_CHILD   1
#define PCS_PARENT  2

union semun {
    int val;
    struct semid_ds *buf;
    unsigned short *array;
    struct seminfo *__buf;
};

key_t _save_semid = 0;

void handle_sig(int sig) {
    if (sig == SIGINT || sig == SIGTERM) {
        printf("remove semid\n");
        semctl(_save_semid, 0, IPC_RMID);
        exit(0);
    }
}

int main(int argc, char *argv[]) {
    
    int process_flag = PCS_PARENT;

    key_t semk = 1024;
    int semid;
    semid = semget(semk, 1, IPC_CREAT|0666);
    if ( semid < 0 ) {
        perror("semget");
        return 1;
    }
    _save_semid = semid;

    union semun semarg;

    struct sembuf mf;
    mf.sem_num = 0;
    mf.sem_op = 1;
    mf.sem_flg = SEM_UNDO;

    semarg.val = 1;
    semarg.buf = NULL;
    semarg.array = NULL;
    semarg.__buf = NULL;

    if (semctl(semid, 0, SETVAL, semarg)<0) {
        semctl(semid, 0, IPC_RMID);
        perror("semctl");
        return 2;
    }

    semop(semid, &mf, 1);

    pid_t pid;

    for(int i=0; i<2; i++) {
        pid = fork();
        if(pid<0) {
            perror("fork");
            kill(-1, SIGTERM);
        }
        if (pid > 0) {
            continue;
        } else {
            process_flag = PCS_CHILD;
            break;
        }
    }

    if (process_flag == PCS_PARENT) {
        int st=0;
        signal(SIGINT, handle_sig);
        for(int i=0; i<3; i++) {
            wait(&st);
        }
    } else {
        /*
            检测信号量是否为0，不为0则减1并输出信息，
            否则等待信号量大于0。
        */
        int ret;

        while (1) {
            //printf("%d \n", semctl(semid, 0, GETVAL));
            //printf("%u waiting\n", getpid());

            mf.sem_op = -1;
            mf.sem_flg |= IPC_NOWAIT | SEM_UNDO;
            
            ret = semop(semid, &mf, 1);
            if (ret < 0) {
                //printf("errno : %d\n", errno);
                //perror("semop");
            } else if (ret == 0) {
                printf("%u : say hey!\n", getpid());
                usleep(500000);
                printf("%u : done.\n", getpid());
                
                mf.sem_op = +1;
                mf.sem_flg = SEM_UNDO;
                if (semop(semid, &mf, 1)<0) {
                    perror("semop");
                }
            }
            sleep(1);
        }
    }


	return 0;
}

```



