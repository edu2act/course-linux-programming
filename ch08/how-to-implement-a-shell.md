## shell的实现原理

> 现在开始考虑一个问题，如何在程序中运行其他程序？

这个问题有一个实际对应的场景：shell执行命令的过程就是要把用户输入的字符串，用空格分割成子串，并根据名称查找对应程序文件，找到后加载执行程序。这个是shell运行命令的基本过程。shell运行命令就是在程序中运行一个指定的其他程序。

### exec系列函数

execv，execvp等函数可以用来执行外部程序。通过man  3 execv查看手册：

```c
EXEC(3)                             Linux Programmer's Manual                             EXEC(3)

NAME
    execl, execlp, execle, execv, execvp, execvpe - execute a file

SYNOPSIS
       #include <unistd.h>

       extern char **environ;

       int execl(const char *path, const char *arg, ...
                       /* (char  *) NULL */);
       int execlp(const char *file, const char *arg, ...
                       /* (char  *) NULL */);
       int execle(const char *path, const char *arg, ...
                       /*, (char *) NULL, char * const envp[] */);
       int execv(const char *path, char *const argv[]);
       int execvp(const char *file, char *const argv[]);
       int execvpe(const char *file, char *const argv[],
                       char *const envp[]);

   Feature Test Macro Requirements for glibc (see feature_test_macros(7)):

       execvpe(): _GNU_SOURCE

DESCRIPTION
    The  exec()  family  of  functions  replaces  the current process image with a new process image.  The functions described in this manual page are front-ends  for  execve(2).   (See the  manual page for  execve(2) for further details about the replacement of the current process image.)

    The initial argument for these functions is the name of a file that is to be executed.

    The const char *arg and subsequent ellipses in the execl(),execlp(), and  execle()  functions can be thought of as arg0, arg1, ...,argn.Together they describe a list of one or more pointers to null-terminated strings that represent the argument list available to the executed program.  The first argument, by convention, should point to the filename associated with the file being executed.  The list of arguments must be  terminated  by  a null pointer, and,since these are variadic functions, this pointer must be cast (char *) NULL.

    The execv(), execvp(), and execvpe() functions provide an array of pointers to null-terminated strings that represent the argument list available to the new  program.   The  first argument,  by 
convention, should point to the filename associated with the file 
being executed.  The array of pointers must be terminated by a null 
pointer.
       ......
       ......
RETURN VALUE
    The exec() functions return only if an error has occurred.  The 
return value  is  -1,  and errno is set to indicate the error.
......
```

注意手册描述，传递的参数，execl,execlp等传递可变参数的形式，最终要以一个NULL指针结束。而execv这种第二个参数是指针数组的形式最终要以一个NULL结尾表示没有更多参数。以execv为例：

| 原型   | int execv(const char *path, char *const argv[]); |
| ------ | ------------------------------------------------ |
| path   | 要运行的命令字符串，要传递绝对路径               |
| argv   | 运行命令传递的参数，注意argv[0]是程序的名字      |
| 返回值 | 只有当程序出错的时候返回-1                       |

execv等函数会使用新的进程镜像替换当前的进程镜像，进程ID不变，但是当前进程注册的信号处理机制会失效。

### execv函数示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char * argv[])
{
    char * cmd_path = "/bin/uname";
    char * cmd_argv[] = {"/bin/uname", "-a", NULL};
    
    if (execv(cmd_path, cmd_argv)<0) {
        perror("execv");
        return 1;
    }

    return 0;
}

```



### 最小shell

现在，让我们来设计一个最小shell，并由此更深刻理解shell的工作方式。使用一个全局变量\_path保存默认的路径，并且使用全局变量\_home_bin存储$HOME/bin目录信息，并检测如果存在的话才会作为默认的命令搜索路径。

执行命令的基本过程：

1. 获取用户的输入
2. 使用strtok按空格分割字符串，第一个字符串作为命令名称，其他的作为参数
3. 从默认的路径中按照输入的名称搜索文件
4. 找到的话，把绝对路径以及其他参数传递给execv函数执行命令
5. 否则提示命令未找到。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <string.h>
#include <dirent.h>

char *_path[] = {
    NULL,
    "/bin",
    "/sbin",
    "/usr/bin",
    "/usr/sbin",
    "/usr/local/bin",
    "/usr/local/sbin"
};

char _home_bin[256] = {'\0'};

#define ARGS_END    1024

#define MAX_NAME_LEN    2048

int _args_ind;

char * _args_p[ARGS_END] = {NULL,};

char _cmd_path[MAX_NAME_LEN];

int find_command(char * dir_list[], int n, char* name);

int main(int argc, char * argv[])
{
    _args_ind = 0;
    char * path = getenv("HOME");
    //这里并没有对长度做检测，实际上Linux会对用户名有最大长度限制
    if (path) {
        strcpy(_home_bin, path);
        strcat(_home_bin, "/bin");
    }
    
    struct stat st;

    if (access(_home_bin,F_OK|R_OK|X_OK)==0
        && lstat(_home_bin, &st)==0
        && S_ISDIR(st.st_mode)
    ) {
        _path[0] = _home_bin;
    }

    int pid = 0;
    char cmd_buf[8192] = {'\0'};
    int count = 0;
    char **cmd_argv = NULL;
    int i;
    while (1) {
        write(1, "|==>", 4); //提示符信息
        count = read(0,cmd_buf,8191); 读取用户输入
        if (count<0) {
            perror("read");
            continue;
        } else {
            /*
            	初始化一些信息，并开始解析参数，获取到的用户输入是一个字符串，
            	要把字符串根据空格分割成数组（C语言中就是多个子串），然后把
            	第一个作为命令名称，其他的作为参数
            */
            cmd_buf[count-1] = '\0';
            _args_ind = 0;
            _args_p[_args_ind] = strtok(cmd_buf, " ");
            if (_args_p[_args_ind]!=NULL) {
                while (_args_ind < ARGS_END) {
                    _args_ind++;
                    //记录每个字串的位置
                    _args_p[_args_ind] = strtok(NULL, " ");
                    if (_args_p[_args_ind]==NULL)break;
                }
            } else {
                _args_ind = 0;
            }
            
            if (_args_p[0]==NULL || strlen(_args_p[0])==0)
                continue;
        }
        
        cmd_argv = (char**)malloc(sizeof(char*)*(_args_ind+1));
        if (cmd_argv == NULL) {
            perror("malloc");
            continue;
        }
        for(i=0;i<_args_ind;i++)
            cmd_argv[i] = _args_p[i];

        //execv要求第二个参数的格式：必须是NULL结尾
        cmd_argv[_args_ind] = NULL; 

        int p_size = sizeof(_path);
        int chp_size = sizeof(char*);
        //查找命令
        if (find_command(_path,p_size/chp_size,cmd_argv[0])) 
        {
            printf("Error,command not found:%s\n",cmd_argv[0]);
            continue;
        }

        pid = fork();
        if (pid < 0) {
            perror("fork");
            continue;
        }

        if (pid > 0) {
            int status = 0;
            wait(&status);//父进程等待子进程退出
            free(cmd_argv);
            cmd_argv = NULL;
        }

        if (pid == 0) {
            //子进程执行命令
            if (execv(_cmd_path, cmd_argv)<0) {
                perror("execv");
                return -1;
            }
        }
    }

    return 0;
}

int find_command(char * dir_list[], int n, char * name) {

    DIR * d = NULL;
    struct dirent * rd;

    for (int i=0; i<n; i++) {
        if (dir_list[i]==NULL)continue;

        d = opendir(dir_list[i]);
        if (d==NULL) {
            perror("opendir");
            continue;
        }
        while((rd = readdir(d))!=NULL) {
            if (strcmp(rd->d_name, "..")==0
               || strcmp(rd->d_name, ".")==0)
                continue;
            if (strcmp(rd->d_name, name)==0) {
                strcpy(_cmd_path, dir_list[i]);
                strcat(_cmd_path, "/");
                strcat(_cmd_path, rd->d_name);
                closedir(d);
                return 0;
            }
        }
        closedir(d);
    }
    return -1;
}
```

这个程序需要强制退出，我们并没有提供exit命令控制程序退出，也没有提供cd，pwd等命令，这些命令都要实现为内建命令。并且现在实现的shell没有处理SIGINT信号。



### 加入内建命令以及信号处理

这里使用一个build_in函数执行内建命令，我们实现的shell在执行命令的时候先检测是不是内建命令，是的话执行，否则从默认路径搜索加载执行。

注册SIGINT信号的处理函数，在用户输入Ctrl+C的时候不会退出。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <string.h>
#include <dirent.h>

char *_path[] = {
    NULL,
    "/bin",
    "/sbin",
    "/usr/bin",
    "/usr/sbin",
    "/usr/local/bin",
    "/usr/local/sbin"
};

char _home_bin[256] = {'\0'};

#define ARGS_END    1024

#define MAX_NAME_LEN    2048

#define BUILD_NOTFD     -1
#define BUILD_OK        0
#define BUILD_ERR       1

int _args_ind;

char * _args_p[ARGS_END] = {NULL,};

char _cmd_path[MAX_NAME_LEN];

int find_command(char * dir_list[], int n, char* name);

int build_in(char * cmd, char * cmd_argv[]);

void handle_sig(int sig) {
    printf("\n");
}

int main(int argc, char * argv[])
{
    signal(SIGINT, handle_sig);
    
    _args_ind = 0;
    char * path = getenv("HOME");
    if (path) {
        strcpy(_home_bin, path);
        strcat(_home_bin, "/bin");
    }
    
    struct stat st;

    if (access(_home_bin,F_OK|R_OK|X_OK)==0
        && lstat(_home_bin, &st)==0
        && S_ISDIR(st.st_mode)
    ) {
        _path[0] = _home_bin;
    }

    int pid = 0;
    char cmd_buf[8192] = {'\0'};
    int count = 0;
    char **cmd_argv = NULL;
    int i;
    while (1) {
        write(1, "|==>", 4);
        count = read(0,cmd_buf,8191);
        if (count<0) {
            perror("read");
            continue;
        } else {
            cmd_buf[count-1] = '\0';
            _args_ind = 0;
            _args_p[_args_ind] = strtok(cmd_buf, " ");
            if (_args_p[_args_ind]!=NULL) {
                while (_args_ind < ARGS_END) {
                    _args_ind++;
                    _args_p[_args_ind] = strtok(NULL, " ");
                    if (_args_p[_args_ind]==NULL)break;
                }
            } else {
                _args_ind = 0;
            }
            
            if (_args_p[0]==NULL || strlen(_args_p[0])==0)
                continue;
        }
        
        cmd_argv = (char**)malloc(sizeof(char*)*(_args_ind+1));
        if (cmd_argv == NULL) {
            perror("malloc");
            continue;
        }
        for(i=0;i<_args_ind;i++)
            cmd_argv[i] = _args_p[i];

        cmd_argv[_args_ind] = NULL;

        if (build_in(cmd_argv[0], cmd_argv+1) != BUILD_NOTFD) {
            continue;
        }

        int p_size = sizeof(_path);
        int chp_size = sizeof(char*);
        if (find_command(_path,p_size/chp_size,cmd_argv[0]))
        {
            printf("Error,command not found:%s\n", cmd_argv[0]);
            continue;
        }

        pid = fork();
        if (pid < 0) {
            perror("fork");
            continue;
        }

        if (pid > 0) {
            int status = 0;
            wait(&status);
            free(cmd_argv);
            cmd_argv = NULL;
        }

        if (pid == 0) {
            if (execv(_cmd_path, cmd_argv)<0) {
                perror("execv");
                return -1;
            }
        }
    }

    return 0;
}

int find_command(char * dir_list[], int n, char * name) {

    DIR * d = NULL;
    struct dirent * rd;

    for (int i=0; i<n; i++) {
        
        if (dir_list[i]==NULL)continue;

        d = opendir(dir_list[i]);
        if (d==NULL) {
            perror("opendir");
            continue;
        }
        while((rd = readdir(d))!=NULL) {
            if (strcmp(rd->d_name, "..")==0
               || strcmp(rd->d_name, ".")==0)
                continue;
            if (strcmp(rd->d_name, name)==0) {
                strcpy(_cmd_path, dir_list[i]);
                strcat(_cmd_path, "/");
                strcat(_cmd_path, rd->d_name);
                closedir(d);
                return 0;
            }
        }
        closedir(d);
    }
    return -1;
}

int build_in(char * cmd, char * cmd_argv[]) {
    
    if (strcmp(cmd, "cd")==0) {
        if (cmd_argv[0]==NULL)
            chdir(getenv("HOME"));
        else {
            if (chdir(cmd_argv[0])<0) {
                perror("chdir");
                return BUILD_ERR;
            }
        }
    } else if (strcmp(cmd, "pwd")==0) {
        char cwd[MAX_NAME_LEN] = {'\0'};
        if (getcwd(cwd, MAX_NAME_LEN-1)==NULL) {
            perror("getcwd");
            return BUILD_ERR;
        }
        printf("%s\n",cwd);
    } else if (strcmp(cmd, "help")==0) {
        printf("There is no help\n");
    } else if (strcmp(cmd, "exit")==0 || strcmp(cmd, "quit")==0) {
        _exit(0);
    } else {
        return BUILD_NOTFD;
    }

    return BUILD_OK;
}
```

