## Linux基础命令：cd，pwd，mkdir

----

这几个命令在Linux中经常会用到，但是，如何实现命令是本节课的主题。通过研究命令如何工作加深对Linux系统的理解。

### cd命令如何切换工作目录

cd切换工作目录使用的是chdir系统调用，通过查阅系统调用手册，发现chdir的信息如下：

```c
SYNOPSIS
       #include <unistd.h>

       int chdir(const char *path);
       int fchdir(int fd);

   Feature Test Macro Requirements for glibc (see feature_test_macros(7)):
       fchdir():
           _BSD_SOURCE || _XOPEN_SOURCE >= 500 || _XOPEN_SOURCE && 	
               _XOPEN_SOURCE_EXTENDED
           || /* Since glibc 2.12: */ _POSIX_C_SOURCE >= 200809L
DESCRIPTION
    chdir()  changes  the current working directory of the calling process to the directory specified in path. fchdir() is identical to chdir(); the only difference is that the directory is given as an open file descriptor.

RETURN VALUE
    On  success, zero is returned.  On error, -1 is returned,and errno is set appropriately.
.......
.......
```

此文档说明调用此函数需要引入的头文件是unistd.h，chdir传递一个char*类型的参数指向要切换到目录字符串，返回值是int，0表示没有错误，-1表示出错。

似乎这一个函数不能解决所有的问题，想想cd命令的功能，当直接运行cd，会切换到当前用户主目录。所以要支持这个功能，需要获取当前用户的信息或者是直接获取主目录信息。解决这个问题的方法不止一种，但有一个比较快的方式是使用getenv函数，不过这不是一个Linux系统调用，而是C语言标准库函数。需要引入<stdlib.h>然后调用此函数：

char* getenv (const char *name);

getenv会从系统环境变量里去搜索name指定的变量，并返回指向此变量的指针，没有找到则返回NULL。这个函数的实现是依赖于平台的，并且环境变量的name在每个平台上都会有不同。在Linux/Unix上，使用getenv("HOME")获取用户主目录。

目前来说进展不错，现在的工作就是整合这些信息进行编码：

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

int main (int argc, char *argv[])
{
    if (argc<2) {
        if(chdir( getenv("HOME") ) < 0){
            perror("chdir");
            return -1;
        }
    } else if (chdir(argv[1]) < 0) {
        perror("chdir");
        return -1;
    }

    return 0;
}
```

代码可以正常编译执行，可是结果却不是预期的那样切换工作目录。为什么会这样？想想shell运行命令的流程：

**解析命令参数  -> 查找命令所在目录 -> fork子进程调用exec*相关函数运行命令**

答案很明显：cd操作发生在fork出的子进程中，对shell无效。所以shell把cd、pwd等一些操作实现为内建命令，由shell调用这些系统接口去实现。



### pwd如何工作

尽管pwd也是shell实现的内建命令，但是独立编写的命令也是可以显示当前工作目录的，再次考虑shell运行命令的过程，fork出的子进程会继承父进程的全部信息，工作目录和父进程是相同的，此时就可以获取工作目录并输出。

获取工作目录的系统调用：

```c
NAME
       getcwd, getwd, get_current_dir_name - get current working directory

SYNOPSIS
       #include <unistd.h>

       char *getcwd(char *buf, size_t size);

       char *getwd(char *buf);

       char *get_current_dir_name(void);

DESCRIPTION
       These functions return a null-terminated string containing 
       an absolute pathname that is  the  current working directory 
       of the calling process.  The pathname is returned as the 
       function result and via the argument buf, if present. 
	  	......
       getwd()  does  not malloc(3) any memory.  The buf argument 
       should be a pointer to an array at least PATH_MAX bytes 
       long.  If the length of the absolute pathname  of  the 
       current  working directory,  including  the terminating null 
       byte, exceeds PATH_MAX bytes, NULL is returned, and errno is 
       set to ENAMETOOLONG.

RETURN VALUE
       On  success, these functions return a pointer to a string 
       containing the pathname of the current working directory.  
       In the case getcwd() and getwd() this  is  the  same value 
       as buf.

       On  failure,  these  functions  return NULL, and errno is 
       set to indicate the error. The contents of the array pointed 
       to by buf are undefined on error.

......
```

通过文档描述发现，可使用的函数不止一个，这里使用getcwd获取当前工作目录，getcwd接受两个参数，第一个参数表明获取的目录信息要存储的位置，第二个是最大存储长度（字节数），这里引入了<linux/limits.h>，其中的宏定义PATH_MAX作为最大长度。成功获取当前工作目录则返回指向目录字符串的指针，否则返回NULL。

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <linux/limits.h>

int main (int argc, char *argv[])
{
    char buf[PATH_MAX] = {'\0'};
    if (getcwd(buf,PATH_MAX)==NULL) {
        perror("getcwd");
        return -1;
    }
    printf("%s\n",buf);
    return 0;
}
```

这个程序是可以正常执行的，会输出当前工作目录。



### 开始研究mkdir命令

shell执行mkdir  [DIR NAME]可以创建目录，mkdir也支持--mode参数用于指定权限，一般使用最多的是默认权限，系统默认权限是通过umask设定的权限掩码去设定默认权限。通过man syscalls查找发现存在一个mkdir的系统调用，通过man 2 mkdir查看文档：

```c
NAME
       mkdir, mkdirat - create a directory
SYNOPSIS
       #include <sys/stat.h>
       #include <sys/types.h>

       int mkdir(const char *pathname, mode_t mode);
		......

DESCRIPTION
       mkdir() attempts to create a directory named pathname.
       The argument mode specifies the mode for the new directory 
       (see inode(7)).  It is modified by the process's umask in 
       the usual way: in the absence of a default ACL, the mode of 
       the created directory is (mode & ~umask & 0777).  Whether 
       other mode bits are honored for the created directory 
       depends on the operating system. 
       For Linux, see NOTES below.

       The newly created directory will be owned by the effective 
       user ID of the process.  If the directory  containing  the  
       file  has  the set group-ID  bit set, or if the filesystem 
       is mounted with BSD group semantics (mount -o bsdgroups or, 
       synonymously mount -o grpid), the new directory will inherit 
       the group ownership from its parent; otherwise it will be
       owned by the effective group ID of the process.
		......
RETURN VALUE
       mkdir()  and  mkdirat() return zero on success, or -1 if an error 
       occurred (in which case,errno is set appropriately).
```

 通过此文档可以发现，mkdir需要两个参数，第一个是要创建的目录路径名称，第二个是权限，用数字表示，比如0755，但是注意这个是8进制的表示，二进制形式：111101101。转换成10进制是493。注意文档中描述的权限计算方式：mode & ~umask & 0777。如果umask是0002（二进制：000000010）。则~umask是111 111 101。所以，如果mode默认是0777，则计算后的权限是0775。

现在先来实现一个最简单的创建目录的命令，使用默认的0755权限，并且只从argv[1]传递的参数作为要创建的目录名称：

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[])
{
    if (argc<2) {
        dprintf(2,"Error:less DIR_NAME\n");
        return -1;
    }
    int mode = 0755;
    if (mkdir(argv[1], mode)<0) {
        perror("mkdir");
    	return -1;
    }
    return 0;
}
```

这个程序能很好的工作，并且能满足大部分的需求。但我们仍然希望程序更完善，就像系统内置的mkdir命令。mkdir支持同时创建多个目录，支持--mode=754这样的参数，支持--help参数输出帮助信息。所以我们编写的命令也支持这样的操作。要支持这样的操作，需要做一些修改：

* 编写一个help函数输出帮助信息
* main函数要通过argc和argv获取参数，并在循环里和预定义选项/参数比较，如果是--help就输出帮助信息退出。如果前7个字符是--mode=，则开始检测后面是不是3个字符长度，是不是全部是0-7之间的数字，注意此时其实是0-7的字符，要把三个字符转换成八进制的数字大小。
* 循环argv时，除了支持的控制选项以外，其他全部作为要创建的目录。

完整的代码如下：

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>

void help(void)
{
    char *help_info[] = {
        "创建目录，支持参数--help，--mode\n",
        "--help：输出帮助信息\n",
        "--mode=[MODE]：设定创建目录的权限，MODE应该是一个三位的数字，",
        "否则程序会报错，数字是0-7的范围。比如，754表示rwxr-xr--。\n",
        "示例：\n",
        "    mkdir --mode=755 a/ b/ c/\n",
        "    mkdir study/\n",
        "    mkdir --help",
        "\n",
        "\0"
    };
    int i=0;
    while (strcmp(help_info[i],"\0")!=0) {
        printf("%s",help_info[i++]);
    }
}

int main(int argc, char *argv[])
{
    if (argc<2) {
        dprintf(2,"Error:less DIR_NAME\n");
        return -1;
    }
    int mode_flag = 0;
    int mode = 0755;
    int i;
    int mode_buf = 0;
    char tmp;
    for(i=1;i<argc;i++) {
        if (strcmp(argv[i],"--help")==0) {
            help();
            return 0;
        } else if (strncmp(argv[i],"--mode=",7)==0) {
            if (strlen(argv[i]+7)!=3) { //仅支持3位数八进制数字
                dprintf(2, "Error: mode is wrong\n");
                return -1;
            }
            for (int k=0;k<3;k++) {
                tmp = argv[i][7+k];
                if (tmp < '0' || tmp > '7') { //如果数字字符超出范围则报错
                    dprintf(2, "Error: mode number must in [0,7]\n");
                    return -1;
                }
                mode_buf += (tmp-48)*(1<<(3*(2-k))); //转换成8进制
            }
            mode_flag = i;
            break;
        }
    }

    if (mode_flag>0 && argc==2) {
        dprintf(2,"Error: less DIR_NAME\n");
        return -1;
    }

    if (mode_flag>0 && mode_buf > 0)
        mode = mode_buf;

    for (i=1;i<argc;i++) {
        if (i==mode_flag)continue;
        if (mkdir(argv[i],mode) < 0) {
            perror("mkdir");
            return -1;
        }
    }

    return 0;
}
```



**练习**

- 实现rmdir命令。