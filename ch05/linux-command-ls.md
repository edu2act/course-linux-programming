## Linux命令ls的实现

ls是Linux上最常用的命令之一。可以显示目录/文件的名称，路径，大小，权限，所属用户等基本信息。如果你觉得Linux上的命令不能完全满足你的需要。可以自己编写一个新的命令。ls命令基本上可以完成大部分工作，但是情况并不总尽如人意。比如，ls -s 默认使用的是1024字节块为单位，可是对于显示文件来说可读性差，需要再次计算；无法进行详细的统计...

看起来ls的工作似乎只是显示文件、目录的基本信息这么简单。仔细想想，这些事情做起来却不是很容易：如何判断参数是不是一个存在的文件/目录、如何读取目录内容、如何判断是目录还是普通文件还是符号链接或是其他类型的文件、递归读取目录信息、输出信息排列、不同类型输出不同颜色、分类统计...；

所以ls命令的实现，至少需要以下要实现的操作或系统调用：

1. 判断文件/目录是否存在
2. 打开，读取，关闭目录
3. 获取文件/目录的详细信息
4. 获取文件/目录的所属用户和组
5. 对输出进行排序
6. 递归读取目录
7. 对不同类型用不同颜色标记
8. 支持使用尾部字符标记文件类型
9. 支持统计功能，统计不同类型的文件数量
10. 如果是软链接，获取目标文件的路径

所以，要完成这些操作，就要对任务进行分解，先完成最基本的操作，之后逐步添加功能。

### 如何判断文件或目录存不存在

access接口可以完成此功能，通过man 2 acccess查看文档部分内容如下：

```C
ACCESS(2)                           Linux Programmer's Manual                           ACCESS(2)

NAME
       access, faccessat - check user's permissions for a file

SYNOPSIS
       #include <unistd.h>

       int access(const char *pathname, int mode);

		...

DESCRIPTION
    access()  checks whether the calling process can access the file pathname.  If pathname is a symbolic link, it is dereferenced.
            
    The mode specifies the accessibility check(s) to be performed, and is  either  the  value F_OK, or a mask consisting of the bitwise OR of one or more of R_OK, W_OK, and X_OK.  F_OK tests for the existence of the file.  R_OK, W_OK, and X_OK test whether  the file exists and grants read, write, and execute permissions, respectively.

    The  check is done using the calling process's real UID and GID, rather than the effective IDs as is done when actually attempting 
		...
RETURN VALUE
    On success (all requested permissions granted, or mode is F_OK and the file exists),  zero is returned.  On error (at least one bit in mode asked for a permission that is denied, or mode is F_OK and the file does not exist, or some other error occurred), -1  is returned,and errno is set appropriately.
...
```

access接口接受两个参数，第一个是字符串指针，指向文件/目录的路径字符串，第二个是整数类型，整数的值使用宏定义表示，R_OK, W_OK, X_OK分别表示可读，可写，可执行。F_OK表示文件/目录存在。access成功返回0，错误返回-1。以下代码是access使用示例，用于检测文件是否存在。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char* argv[])
{

    if (argc < 2) {
        dprintf(2, "Error: less arguments -> file/dir name\n");
        return 1;
    }

    for (int i=1; i < argc; i++) {
        if (access(argv[i], F_OK) < 0)
            dprintf(2, "Error: %s not exists\n", argv[i]);
        else
            printf("%s exists\n", argv[i]);
    }

    return 0;
}

```



### 如何获取文件详细信息

Linux提供了stat，lstat，fstat系统调用用于获取文件详细信息。几个系统调用存在些许区别，stat和lstat通过字符串路径传递参数，并把获取的信息放在一个struct stat结构中，lstat和stat的区别是如果路径是一个符号链接，lstat获取的是符号链接文件自己的信息，而stat获取的是符号链接指向的目标文件的信息。通过man 2 stat查看联机文档：

```c
STAT(2)                             Linux Programmer's Manual                             STAT(2)

NAME
       stat, fstat, lstat, fstatat - get file status

SYNOPSIS
       #include <sys/types.h>
       #include <sys/stat.h>
       #include <unistd.h>

       int stat(const char *pathname, struct stat *statbuf);
       int fstat(int fd, struct stat *statbuf);
       int lstat(const char *pathname, struct stat *statbuf);
		......
		......
DESCRIPTION
    These  functions return information about a file, in the buffer 
pointed to by statbuf.  No permissions are required on the file 
itself, but—in the case  of  stat(),  fstatat(),  and 
lstat()—execute (search) permission is required on all of the
directories in pathname that lead to the file.

		......
   lstat()  is  identical  to  stat(),  except  that  if pathname is a symbolic link, then it returns information about the link itself,not the file that it refers to.

    fstat() is identical to stat(), except that the file about which information  is  to  be retrieved is specified by the file 
descriptor fd.

   The stat structure All of these system calls return a stat structure, which contains the following fields:

    struct stat {
        dev_t     st_dev;      /* ID of device containing file */
        ino_t     st_ino;      /* Inode number */
        mode_t    st_mode;     /* File type and mode */
        nlink_t   st_nlink;    /* Number of hard links */
        uid_t     st_uid;      /* User ID of owner */
        gid_t     st_gid;      /* Group ID of owner */
        dev_t     st_rdev;     /* Device ID (if special file) */
        off_t     st_size;     /* Total size, in bytes */
        blksize_t st_blksize;   /* Block size for filesystem I/O */
        blkcnt_t  st_blocks;  /* Number of 512B blocks allocated */

        /* Since Linux 2.6, the kernel supports nanosecond
           precision for the following timestamp fields.
           For the details before Linux 2.6, see NOTES. */

        struct timespec st_atim;  /* Time of last access */
        struct timespec st_mtim;  /* Time of last modification */
        struct timespec st_ctim;  /* Time of last status change */

        /* Backward compatibility */
        #define st_atime st_atim.tv_sec
        
        #define st_mtime st_mtim.tv_sec
        #define st_ctime st_ctim.tv_sec
    };

       ......
       ......
       ......
RETURN VALUE
    On success, zero is returned.  On error, -1 is returned, and errno is set appropriately.
......
```

此文档清晰地说明了函数调用方式，struct stat结构体中每个变量的含义。接下来我们写一个简单的程序测试lstat函数：

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>

void out_st_info(char * path, struct stat * st) {
    printf("%s -> size: %lu bytes\n", path, st->st_size);
    printf("    i-node: %-9lu  hard link: %-2lu ", st->st_ino, st->st_nlink);

    char * ptype = "";
    switch (st->st_mode & S_IFMT) {
        case S_IFDIR:
            ptype = "dir";
            break;
        case S_IFIFO:
            ptype = "fifo";
            break;
        case S_IFBLK:
            ptype = "block device";
            break;
        case S_IFCHR:
            ptype = "char device";
            break;
        case S_IFSOCK:
            ptype = "sock";
            break;
        case S_IFREG:
            ptype = "regular";
            if (access(path, X_OK)==0) { //如果可执行，输出可执行标记
                ptype = "regular*";
            }
            break;
        case S_IFLNK:
            ptype = "link";
            break;
    };

    printf("%s\n",ptype);
}

int main(int argc, char *argv[])
{
    struct stat st;

    for (int i=1; i<argc; i++) {
        //检测文件/目录是否存在
        if (access(argv[i], F_OK)) {
            dprintf(2, "Error: %s is not exists\n", argv[i]);
            continue;
        }
		//获取状态信息，成功则输出，失败输出错误信息
        if (lstat(argv[i], &st)==0) {
            out_st_info(argv[i], &st);
        } else {
            perror("lstat");
        }
    }

    return 0;
}
```

这个程序对argv获取的每个值作为路径传递给lstat获取文件信息，并使用out_st_info函数输出名称、大小、I-node号、硬链接数、类型等信息。



### 如何操作目录

基于一切皆是文件的设计，目录也是文件，这意味着open函数可以打开目录，确实如此，但是打开后读取的数据却不正常，早期的类Unix系统会显示很多目录、文件名称并有很多其他字符，因为是结构化的目录数据。而在目前的Linux上测试发现并不会输出任何结果。目录存储的是文件名称，文件类型，文件大小等信息，需要一个数据存储结构。这需要一套对应的接口进行操作。

Linux提供了目录操作相关的系统调用：opendir，readdir，closedir。通过名称就可以知道相应的操作。通过man 3 [API NAME]可以查看对应的说明手册。

opendir接受一个字符串作为目录路径，成功打开目录返回一个DIR结构的指针，错误返回NULL。

readdir从opendir返回的DIR*读取目录内容，每次返回一个struct dirent结构体指针，我们最关心的是其中的d_name存储文件的名称。

closedir关闭opendir打开的目录，传递参数是DIR*，就是opendir的返回值。

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <dirent.h>

int main(int argc, char *argv[])
{
    if (argc<2) {
        dprintf(2,"Error:less DIR_NAME\n");
        return -1;
    }

    DIR * dr = opendir(argv[1]);
    if (dr==NULL) {
        perror("opendir");
        return -1;
    }
    
    struct dirent * r = NULL;
    while((r=readdir(dr))!=NULL) {
        printf("%s\n", r->d_name);
    }
    
    return 0;
}
```



### ls命令的实现：最简版本

第一个版本是一个对目录相关系统调用的示例，仅仅打开目录并列出目录内容。实现起来比较容易。这个程序通过读取参数作为目录或文件的名称，并显示相关信息。如果没有参数则默认读取 . （当前目录）。

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <dirent.h>
#include <fcntl.h>

int list_dir(char * path);

int main(int argc, char *argv[])
{
    if (argc<2) {
        list_dir(".");
    }

    for (int i=1; i<argc; i++) {
        if (access(argv[i], F_OK)==0) {
            list_dir(argv[i]);
        } else {
            dprintf(2, "Error: %s is not exists\n", argv[i]);
        }
    }

    return 0;
}

int list_dir(char * path) {
    struct stat st;
    char flag = '\0';
    DIR * d = NULL;
    struct dirent * rd = NULL;
    if (lstat(path, &st)<0) {
        perror("lstat");
        return -1;
    }
    if (S_ISDIR(st.st_mode)) {
        if ((d=opendir(path))==NULL) {
            perror("opendir");
            return -1;
        }
        printf("%s/:\n",path);
        while((rd=readdir(d))!=NULL) {
            printf("%s\n", rd->d_name);
        }
        closedir(d);
    } else {
        printf("%s\n", path);
    }
    return 0;
}

```

这个程序功能十分简单，而且这仅仅是一个测试程序。为了能够区分我们自己实现的命令和系统默认的ls，我们把自己实现的版本命名为li。并且在后续课程中也使用li表示我们自己实现的ls命令。

### li命令功能增强实现

接下来我们可以实现更复杂的功能，并且我们会重新设计程序。下面的程序实现了显示文件大小，链接的目标文件，文件格式，文件所有者和组（UID，GID），硬链接数，等信息。并且提供了参数用于控制显示哪些信息。配合程序的注释你可以看懂这个程序。
```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <dirent.h>
#include <fcntl.h>


#define TYPE_DIR        '/'
#define TYPE_LNK        '@'
#define TYPE_FIFO       '|'
#define TYPE_SOCK       '='
#define TYPE_EXEC       '*'
#define TYPE_CHR        '%'
#define TYPE_BLK        '#'


#define ARGS_PATH       0
#define ARGS_SIZE       1
#define ARGS_LINK       2
#define ARGS_TYPE       3    //char for file type
#define ARGS_MODE       4
#define ARGS_LSALL      5
#define ARGS_INO        6
#define ARGS_LONGINFO   7
#define ARGS_HELP       8

#define ARGS_END        32

#define MAX_NAME_LEN    2048

char _args[ARGS_END] = {0};


int list_fildir(char * pathname);

void help(void);

void out_info(char * pathname, char * name, struct stat * st);

int main(int argc, char *argv[])
{
    char _path_buffer[MAX_NAME_LEN] = {'\0'};
    int len_buf = 0;
    int args_dir_count = 0;

    int * fil_ind = malloc(sizeof(int)*argc);
    if (fil_ind ==NULL) {
        perror("malloc");
        return 1;
    }
    bzero((void*)fil_ind, sizeof(int)*argc);

    for(int i=1;i<argc;i++) {
        if (strcmp(argv[i],"--lnk")==0) {
            _args[ARGS_LINK] = 1;
        } else if (access(argv[i], F_OK)==0) {
            len_buf = strlen(argv[i]);
            if (len_buf >= MAX_NAME_LEN) {
                dprintf(2,"Error: the length of path too long -> %s\n",argv[i]);
                continue;
            }
            fil_ind[i] = 1;
            args_dir_count += 1;
        } else if (argv[i][0]=='-') {
            len_buf = strlen(argv[i]);
            for(int a=1; a<len_buf; a++){
                if (argv[i][a]=='a')
                    _args[ARGS_LSALL] = 1;
                else if (argv[i][a] == 'i')
                    _args[ARGS_INO] = 1;
                else if (argv[i][a] == 'm')
                    _args[ARGS_MODE] = 1;
                else if (argv[i][a] == 'p')
                    _args[ARGS_PATH] = 1;
                else if (argv[i][a] == 's')
                    _args[ARGS_SIZE] = 1;
                else if (argv[i][a] == 'f')
                    _args[ARGS_TYPE] = 1;
                else if (argv[i][a] == 'l')
                    _args[ARGS_LONGINFO] = 1;
                else if (argv[i][a] == 'h') {
                    _args[ARGS_HELP] = 1;
                }
      
            }
        }
        else
            dprintf(2, "Error:unknow arguments -> %s\n", argv[i]);
    }
    
    if (_args[ARGS_HELP]) {
        help();
    } else if (args_dir_count == 0) {
        list_fildir(".");
    } else {
        for(int i=1; i<argc; i++) {
            if (fil_ind[i]==1) {
                list_fildir(argv[i]);
            }
        }
    }

    free(fil_ind);
    fil_ind = NULL;

    return 0;
}

int list_fildir(char* pathname) {
    struct stat st, stbuf;
    DIR* d = NULL;
    struct dirent * rd = NULL;

    char fbuf[MAX_NAME_LEN] = {'\0'};

    if (lstat(pathname, &st)<0) {
        perror("lstat");
        return -1;
    }

    int path_len = strlen(pathname);
    
    strcpy(fbuf, pathname);
    if (pathname[path_len] != '/'  
        && (path_len > 1 || pathname[0]!='/')
    ) {
        strcat(fbuf, "/");
        path_len += 1;
    }

    if (S_ISDIR(st.st_mode)) {
        d = opendir(pathname);
        if (d==NULL) {
            perror("opendir");
            return -1;
        }
        while((rd = readdir(d))!=NULL) {
            if (strlen(pathname) + strlen(rd->d_name) + 1 >= MAX_NAME_LEN) {
                dprintf(2, "Error: name too long\n");
                continue;
            }
            if (_args[ARGS_LSALL]==0 && rd->d_name[0]=='.')
                continue;

            fbuf[path_len] = '\0';
            strcat(fbuf, rd->d_name);

            if (lstat(fbuf, &stbuf)<0) {
                perror("lstat");
                continue;
            }
            out_info(fbuf, rd->d_name, &stbuf);
        }
        closedir(d);
    } else {
        out_info(pathname, pathname, &st);
    }
    
    return 0;
}

void out_info(char * pathname, char * name, struct stat * st) {
    if (_args[ARGS_INO])
        printf("%-8lu ", st->st_ino);
    
    if(_args[ARGS_MODE] || _args[ARGS_LONGINFO])
        printf("%o ", st->st_mode & 0777);

    if (_args[ARGS_LONGINFO]) {
        printf("%-2lu %d %d ", st->st_nlink, st->st_uid, st->st_gid);
    }

    if (_args[ARGS_PATH])
        printf("%s", pathname);
    else
        printf("%s", name);

    char flag = '\0';
    if (_args[ARGS_TYPE]) {
        if (S_ISDIR(st->st_mode))
            flag = TYPE_DIR;
        else if (S_ISLNK(st->st_mode))
            flag = TYPE_LNK;
        else if (S_ISFIFO(st->st_mode))
            flag = TYPE_FIFO;
        else if (S_ISSOCK(st->st_mode))
            flag = TYPE_SOCK;
        else if (S_ISCHR(st->st_mode))
            flag = TYPE_CHR;
        else if (S_ISBLK(st->st_mode))
            flag = TYPE_BLK;
        else if (S_ISREG(st->st_mode) && access(pathname,X_OK)==0)
            flag = TYPE_EXEC;
        if (flag > 0)
            printf("%c",flag);
    }

    if (_args[ARGS_SIZE] || _args[ARGS_LONGINFO]) {
        if (st->st_size <= 1024)
            printf(" %luB",st->st_size);
        else if (st->st_size > 1024 && st->st_size < 1048576)
            printf(" %.2lfK", (double)st->st_size/1024);
        else
            printf(" %.2lfM",(double)st->st_size/1048576);
    }

    if (S_ISLNK(st->st_mode) 
        && (_args[ARGS_LONGINFO] || _args[ARGS_LINK])
    ) {
        char _path_buffer[MAX_NAME_LEN];
        int link_len = readlink(pathname, _path_buffer, MAX_NAME_LEN-1);
        if (link_len > 0) {
            _path_buffer[link_len] = '\0';
            printf("-> %s ", _path_buffer);
        }
    }

    printf("\n");
}

void help(void)
{
    char *help_info[] = {
        "显示文件/目录信息,类似于ls命令。\n",
        "参数/选项：\n",
        "-l ：显示详细信息，包括权限、用户、用户组、大小等信息；\n-m ：权限\n",
        "-a ：显示隐藏文件；\n-i ：i-node编号；\n-s ：大小；\n-f ：文件类型字符\n",
        "-p ：显示路径；\n--lnk ：链接目标文件\n",
        "示例：\n",
        "li -al /usr \nli -rfs /usr/share /usr/local\n",
        "\n",
        "\0"
    };
    int i=0;
    while (strcmp(help_info[i],"\0")!=0) {
        printf("%s",help_info[i++]);
    }
}
```

如果按照这样的方式设计下去，加入递归显示目录，分类统计，支持颜色输出还是可以的，但是会让程序变得比较乱，不容易维护。









### *li命令的完整实现

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <dirent.h>
#include <fcntl.h>
#include <math.h>
#include <pwd.h>
#include <grp.h>
#include <regex.h>
#include <time.h>
#include <sys/ioctl.h>
#include <termios.h>

#define PROGRAM_NAME    "li"

#define TYPE_DIR        '/'
#define TYPE_LNK        '@'
#define TYPE_FIFO       '|'
#define TYPE_SOCK       '='
#define TYPE_EXEC       '*'
#define TYPE_CHR        '%'
#define TYPE_BLK        '#'

#define TYPE_INFO   "\
    / : dir\n\
    @ : symble link\n\
    | : FIFO(PIPE)\n\
    = : socket\n\
    * : exec\n\
    %% : character device\n\
    # : block device\n"

void list_type_info() {
    printf(TYPE_INFO);
}

#define SORT_BYNAME     0
#define SORT_BYTM       1
#define SORT_BYCHTM     2
#define SORT_BYSIZE     3

#define SORT_TY_CELL    'd'
#define SORT_TY_ALL     'a'


#define ARGS_PATH       0
#define ARGS_SIZE       1
#define ARGS_LINK       2
#define ARGS_TYPE       3    //char for file type
#define ARGS_MODE       4
#define ARGS_USRGRP     5
#define ARGS_CREATM     6  //create time
#define ARGS_CHANGTM    7  //last change time
#define ARGS_ACCESTM    8    //last access time
#define ARGS_REGEX      9 
#define ARGS_RECUR      10
#define ARGS_STATIS     11    //recursion
#define ARGS_DIRSELF    12    //statistic
#define ARGS_REGNODIR   13
#define ARGS_REGNOFIL   14
#define ARGS_SORT       15
#define ARGS_BLOCK      16
#define ARGS_NODIR      17    //not list dir, just file
#define ARGS_SORTYP     18    //sort type : sort in cell of dir or sort all fil
#define ARGS_ONELINE    19    //show info in one line
#define ARGS_LSALL      20
#define ARGS_INO        21
#define ARGS_LONGINFO   22
#define ARGS_SHOWSTATS  23
#define ARGS_COLOR      24
#define ARGS_OUTMORE    25

#define ARGS_END        32

#define STDOUT_SCRN     1
#define STDOUT_FIFO     2
#define STDOUT_FILE     3

#define MAX_NAME_LEN    2048
#define PATH_CELL_END   16



/*
    save path info 
*/
struct path_cell {
    char path[MAX_NAME_LEN];
    char is_root;
    int  plen;
    int  height;
};

/*
    path list for recur dir 
*/
struct path_list {
    struct path_cell pce[PATH_CELL_END];
    int end_ind;

    struct path_list * next;
    struct path_list * prev;
    struct path_list * plast;
};

struct path_list *
init_path_list(struct path_list* pl) {
   pl->end_ind = 0;
   pl->next = NULL;
   pl->prev = NULL;
   pl->plast = pl;
   return pl;
}

void destroy_path_list(struct path_list * pl) {
    struct path_list * ptmp;
    ptmp = pl;

    while(pl!=NULL) {
        ptmp = pl->next;
        free(pl);
        pl = ptmp;
    }
}

struct path_list *
add_path_list(struct path_list * pl, char * path, int height);

struct path_list *
del_path_list(struct path_list * pl, struct path_list * pnode);

struct path_list *
get_path_list_last(struct path_list * pl);


struct path_list *
add_path_list(struct path_list * pl, char * path, int height) {
    struct path_list * plast =  pl->plast; //get_path_list_last(pl);
    struct path_cell * pcell = NULL;
    struct path_list * ptmp;

    //printf("add path: %s\n", path);
    if (plast->end_ind < PATH_CELL_END) {
        pcell = plast->pce+plast->end_ind;
        plast->end_ind += 1;
    } else {
        ptmp = (struct path_list*)malloc(sizeof(struct path_list));
        if (ptmp==NULL) {
            perror("malloc");
            return NULL;
        }
        plast->next = ptmp;
        ptmp->next = NULL;
        ptmp->prev = plast;
        ptmp->end_ind = 0;
        pcell = ptmp->pce;
        pl->plast = ptmp;
    }

    strcpy(pcell->path, path);
    pcell->is_root = 0;
    int path_len = strlen(path);
    if (path_len==1 && path[0]=='/') {
        pcell->is_root = 1;
    }

    pcell->plen = path_len;
    if (height > 0) {
        pcell->height = height;
    }

    return ptmp;
}

struct path_list *
del_path_list(struct path_list * pl, struct path_list * pnode) {
    
}

struct path_list *
get_path_list_last(struct path_list * pl) {
    struct path_list * pcur = pl;
    while (pcur!=NULL && pcur->next!=NULL) {
        pcur = pcur->next;
    }
    return pcur;
}

/*
    global vars
*/
struct infoargs {

    //main arguments
    char args[ARGS_END];

    //for regex
    regex_t    regcom[1];
    regmatch_t regmatch[1];
    
    //standard out type
    int stdout_type;
};

int set_stdout_type(int * stdo) {
    struct stat dst;
    if (fstat(1, &dst) < 0) {
        perror("fstat");
        return -1;
    }

    if (S_ISFIFO(dst.st_mode))
        *stdo = STDOUT_FIFO;
    else if (S_ISCHR(dst.st_mode))
        *stdo = STDOUT_SCRN;
    else
        *stdo = STDOUT_FILE;

    return 0;
}

void init_infoargs(struct infoargs * ia) {
    bzero(ia->args, sizeof(char)*ARGS_END);
    ia->stdout_type = STDOUT_SCRN;
    set_stdout_type(&ia->stdout_type);
}

//define global var
struct infoargs _iargs;



/*
   定义文件缓存的结构，用于存储获取到的文件名、
   状态信息、所属用户和组等信息。
   然后定义文件缓存的链表结构，定义文件列表缓存结构。
   文件列表缓存结构包含文件缓存结构体数组，以及文件缓存链表
   当数组使用完毕开始使用链表，目的在于平衡存储与性能。
*/
struct file_buf {
    char name[256];
    char path[MAX_NAME_LEN];
    char uname[40];
    char group[40];
    int height;
    int dir_is_hide; //如果是正则表达式搜索模式，用于标记是否显示，用于目录
    struct stat st;
};

#define BUFF_LEN    4096

struct file_buf_list {
    struct file_buf fbuf;
    struct file_buf_list * next;
};

struct file_list_cache {
    int end_ind;
    struct file_buf fcache[BUFF_LEN+1];
    int list_count;
    struct file_buf_list fbhead;
    struct file_buf_list * flast;

    //for sort
    struct file_buf **fba;
};

struct format_info {
    int count;
    int name_max_len;
    int uname_max_len;
    int group_max_len;
    int win_row;
    int win_col;
    int out_row;
};

#define SWAP(a,b)   tmp=a;a=b;b=tmp;
void qsortfbuf(struct file_buf *d[], int start, int end) {
    if (start >= end) {
        return ;
    }

    int med = (end+start)/2;
    int i = start;
    int j = end;
    struct file_buf *tmp = NULL;
    
    SWAP(d[med],d[start]);

    for(j=start+1;j<=end;j++) {
        if (strcmp(d[j]->name,d[start]->name) < 0) {
            i++;
            if (i==j)continue;
            SWAP(d[i],d[j]);
        }
    }

    SWAP(d[i],d[start]);

    qsortfbuf(d, start, i-1);
    qsortfbuf(d, i+1,end);
}

void init_flist_cache (struct file_list_cache * flcache) {
    flcache->end_ind = 0;
    flcache->list_count = 0;
    flcache->fbhead.next = NULL;
    flcache->flast = &flcache->fbhead;
    flcache->fba = NULL;
}

int set_st_fbuf(struct file_buf *fbuf, 
        struct stat *st, char * name, char *path, int height)
{
    memcpy(&fbuf->st, st, sizeof(struct stat));
    if (name) {
        strcpy(fbuf->name, name);
    }
    if (path)
        strcpy(fbuf->path, path);

    if (height > 0)
        fbuf->height = height;

    fbuf->uname[0] = '\0';
    fbuf->group[0] = '\0';

    struct passwd * pd; 
    struct group * grp;
    pd = getpwuid(fbuf->st.st_uid);
    grp = getgrgid(fbuf->st.st_gid);
    if (pd && grp) {
        strcpy(fbuf->uname, pd->pw_name);
        strcpy(fbuf->group, grp->gr_name);
    }
    fbuf->dir_is_hide = 0;
    
    return 0;
}

void set_fbuf_hide(struct file_buf *fbuf, int hide) {
    fbuf->dir_is_hide = hide;
}

int add_to_flcache(struct file_list_cache * flcache, struct file_buf * fbuf) {
    struct file_buf * fbtmp = NULL;
    struct file_buf_list *fl = NULL;
    if (flcache->end_ind < BUFF_LEN) {
        fbtmp = flcache->fcache+flcache->end_ind;
        flcache->end_ind += 1;

    } else {
        fl = (struct file_buf_list *)malloc(sizeof(struct file_buf_list));
        if (fl == NULL) {
            perror("malloc");
            return -1;
        }
        fl->next = NULL;
        flcache->flast->next = fl;
        flcache->flast = fl;
        flcache->list_count += 1;
        fbtmp = &fl->fbuf;
    }
    memcpy(fbtmp, fbuf, sizeof(struct file_buf));

    return 0;
}

void destroy_flcache(struct file_list_cache *flcache) {
    flcache->end_ind = 0;
    struct file_buf_list * fbtmp = flcache->fbhead.next;
    struct file_buf_list * fbtmp2 = NULL;
    
    while(fbtmp) {
        fbtmp2 = fbtmp->next;
        free(fbtmp);
        fbtmp = fbtmp2;
    }
    flcache->list_count = 0;
    flcache->fbhead.next = NULL;
    flcache->flast = &flcache->fbhead;
    free(flcache->fba);
    flcache->fba = NULL;
}


int fbuf_sort(struct file_list_cache * flcache, int sort_flag) {
    int total_count = flcache->end_ind + flcache->list_count;
    flcache->fba = (struct file_buf**)malloc(sizeof(struct file_buf*)*total_count);
    if (flcache->fba==NULL) {
        return -1;
    }

    int i=0;
    for (; i<flcache->end_ind; i++)
        flcache->fba[i] = flcache->fcache+i;

    struct file_buf_list *fl = flcache->fbhead.next;
    while(fl) {
        flcache->fba[i] = &fl->fbuf;
        i++;
        fl = fl->next;
    }
    
    qsortfbuf(flcache->fba, 0, total_count-1);

    return 0;
}


int
add_fbuf_dirs_to_plist(struct file_list_cache* flcache, 
        struct path_list* plist)
{

    struct file_buf *fbtmp = NULL;
    struct file_buf_list * fl;
    int i=0;
    int total = flcache->list_count + flcache->end_ind;
    for(i=0; i<total; i++) {
        if (S_ISDIR(flcache->fba[i]->st.st_mode))
            add_path_list(plist, flcache->fba[i]->path, flcache->fba[i]->height);
    }
    /*
    for (i=0; i<flcache->end_ind; i++) {
        fbtmp = &flcache->fcache[i];
        if (S_ISDIR(fbtmp->st.st_mode)) {
            add_path_list(plist, fbtmp->path, fbtmp->height);
        }
    }
    
    if (flcache->list_count > 0) {
        fl = flcache->fbhead.next;
        while(fl) {
            fbtmp = &fl->fbuf;
            if (S_ISDIR(fbtmp->st.st_mode)) {
                add_path_list(plist, fbtmp->path, fbtmp->height);
            }
            fl = fl->next;
        }
    }*/
    return 0;
}


struct statis {
    unsigned long long dir_count;
    unsigned long long file_count;
    unsigned long long fifo_count;
    unsigned long long link_count;
    unsigned long long sock_count;
    unsigned long long chr_count;
    unsigned long long blk_count;
    unsigned long long total_count;
    unsigned long long total_size;
    unsigned long long file_total_size;
};


struct allinfocell {
    struct statis stats;
    struct format_info fi;
    struct file_list_cache flcache;
};

struct allinfocell _aic;
struct path_list _pathlist;

#define MAX_OUTLINE_LEN     4096

void out_color(struct file_buf * fb, char *pname) {
    if (S_ISDIR(fb->st.st_mode))
        printf("\e[1;34m%s",pname);
    else if(S_ISLNK(fb->st.st_mode))
        printf("\e[1;35m%s",pname);
    else if (S_ISREG(fb->st.st_mode) && access(fb->path,X_OK)==0)
        printf("\e[2;36m%s",pname);
    else if (S_ISFIFO(fb->st.st_mode))
        printf("\e[2;33m%s", pname);
    else if (S_ISCHR(fb->st.st_mode) || S_ISBLK(fb->st.st_mode))
        printf("\e[2;31m%s", pname);
    else if (S_ISSOCK(fb->st.st_mode))
        printf("\e[2;35m%s", pname);
    else
        printf("%s", pname);
    printf("\e[0;m");
}

void format_size(unsigned long long size, char * fstr) {
    if (size <= 1024)
        sprintf(fstr, "%lluB",size);
    else if (size > 1024 && size < 1048576)
        sprintf(fstr, "%.2lfK", (double)size/1024);
    else
        sprintf(fstr, "%.2lfM", (double)size/1048576);
}

void out_info(struct file_buf *fbuf, char *ppath, 
        struct format_info *fi, char *outline)
{
    char fmt_str[256] = {'\0'};
    int posi = 0, count=0;


    if (_iargs.args[ARGS_INO]) {
        count = sprintf(outline + posi, "%-9lu ", fbuf->st.st_ino);
        posi += count;
    }
    
    if (_iargs.args[ARGS_MODE] || _iargs.args[ARGS_LONGINFO]) {
        count = sprintf(outline + posi, "%o ", fbuf->st.st_mode & 0777);
        posi += count;
    }

    if (_iargs.args[ARGS_LONGINFO]) {
        count = sprintf(outline + posi, "%-2lu ", fbuf->st.st_nlink);
        posi += count;
    }
    
    if (_iargs.args[ARGS_USRGRP] || _iargs.args[ARGS_LONGINFO]) {
        sprintf(fmt_str, "%%-%ds %%-%ds ", fi->uname_max_len, fi->group_max_len);
        count = sprintf(outline + posi, fmt_str, fbuf->uname, fbuf->group);
        posi += count;
    }

    if (_iargs.args[ARGS_CREATM]) {
        time_t t = time(NULL);
        struct tm *ct = localtime(&t);
        count = sprintf(outline+posi, "%d.%2d.%2d %2d:%2d ", ct->tm_year+1900, 
                ct->tm_mon+1, ct->tm_mday, ct->tm_hour, ct->tm_min);
        posi += count;
    }

    
    int fmt_name_len = 0;
    if ((_iargs.args[ARGS_PATH] || _iargs.args[ARGS_REGEX]) 
        && ppath
    ) {

        count = sprintf(outline + posi, "%s/", ppath);
        posi += count;
    }

    fmt_name_len = fi->name_max_len;
    
    count = sprintf(outline+posi, "%s", fbuf->name);
    posi += count;

    char flag = '\0';
    if (_iargs.args[ARGS_TYPE]) {
        fmt_name_len += 1;
        if (S_ISDIR(fbuf->st.st_mode))
            flag = TYPE_DIR;
        else if (S_ISLNK(fbuf->st.st_mode))
            flag = TYPE_LNK;
        else if (S_ISFIFO(fbuf->st.st_mode))
            flag = TYPE_FIFO;
        else if (S_ISSOCK(fbuf->st.st_mode))
            flag = TYPE_SOCK;
        else if (S_ISCHR(fbuf->st.st_mode))
            flag = TYPE_CHR;
        else if (S_ISBLK(fbuf->st.st_mode))
            flag = TYPE_BLK;
        else if (S_ISREG(fbuf->st.st_mode) && access(fbuf->path,X_OK)==0)
            flag = TYPE_EXEC;
        if (flag =='\0'){
            flag = ' ';
        }
        count = sprintf(outline+posi, "%c", flag);
        posi += count;
    }
    
    if (_iargs.args[ARGS_OUTMORE]) {
        sprintf(fmt_str, "%%-%ldc", fmt_name_len-strlen(fbuf->name));
        count = sprintf(outline+posi, fmt_str, ' ');
        posi += count;
    }
    
    count = sprintf(outline+posi, " ");
    posi += count;

    if (_iargs.args[ARGS_SIZE] || _iargs.args[ARGS_LONGINFO]) {
        format_size(fbuf->st.st_size, outline+posi);
        posi = strlen(outline);
    }

    if (_iargs.args[ARGS_LINK] || _iargs.args[ARGS_LONGINFO]) {
        
        char _path_buffer[MAX_NAME_LEN];
        int link_len = readlink(fbuf->path, _path_buffer, MAX_NAME_LEN-1);
        if (link_len > 0) {
            _path_buffer[link_len] = '\0';
            count = sprintf(outline+posi, "-> %s", _path_buffer);
            posi += count;
        }
    }

}

int out_flcache(struct file_list_cache *flcache, 
        char *path, struct format_info *fi)
{

    char outline[MAX_OUTLINE_LEN+128];
    char outcolor[MAX_OUTLINE_LEN+128];
    int max_out_len = strlen(path)
                    + fi->uname_max_len
                    + fi->group_max_len
                    + fi->name_max_len;
    if (max_out_len > MAX_OUTLINE_LEN) {
        return -1;
    }

    if (fbuf_sort(flcache, SORT_BYNAME)<0) {
        return -1;
    }

    if (!_iargs.args[ARGS_PATH]
        && !_iargs.args[ARGS_REGEX]
        && !_iargs.args[ARGS_SHOWSTATS]
    ) {
        printf("%s/:\n", path);
    }

    int max_len = fi->name_max_len + 2;
    
    if (_iargs.args[ARGS_TYPE])
        max_len += 1;

    fi->out_row = fi->win_col/max_len;

    int fmt_width = fi->win_col/fi->out_row;
    
    char fmt_str[128] = {'\0'};
    int next_line = 0;
    int total = flcache->end_ind + flcache->list_count;
    
    for(int i=0; i<total; i++) {
        if (S_ISDIR(flcache->fba[i]->st.st_mode)
            && flcache->fba[i]->dir_is_hide
        ) {
            continue;
        } 

        out_info(flcache->fba[i], path, fi, outline);
        
        if (_iargs.args[ARGS_OUTMORE]) {
            if (_iargs.args[ARGS_COLOR] 
                && _iargs.stdout_type == STDOUT_SCRN
            ) {
                out_color(flcache->fba[i], outline);
                printf("\n");
            }
            else
                printf("%s\n", outline);
        } else {
            sprintf(fmt_str, "%%-%ds", max_len);
            if (_iargs.args[ARGS_COLOR]
                && _iargs.stdout_type == STDOUT_SCRN
            ) {
                sprintf(outcolor, fmt_str, outline);
                out_color(flcache->fba[i], outcolor);
            } else {
                printf(fmt_str, outline);
            }
            next_line += 1;
            if (next_line >= fi->out_row) {
                next_line = 0;
                printf("\n");
            }
        }
    }
    if (next_line!=0) {
        printf("\n");
    }

    return 0;
}

void start_statis(struct stat *sttmp, struct statis * stats) {
    stats->total_count++;
    stats->total_size += sttmp->st_size;
    
    if (!S_ISDIR(sttmp->st_mode))
        stats->file_total_size += sttmp->st_size;

    if (S_ISDIR(sttmp->st_mode))
        stats->dir_count++;
    else if (S_ISREG(sttmp->st_mode))
        stats->file_count++;
    else if (S_ISLNK(sttmp->st_mode))
        stats->link_count++;
    else if (S_ISFIFO(sttmp->st_mode))
        stats->fifo_count++;
    else if (S_ISSOCK(sttmp->st_mode))
        stats->sock_count++;
    else if (S_ISCHR(sttmp->st_mode))
        stats->chr_count++;
    else if (S_ISBLK(sttmp->st_mode))
        stats->blk_count++;
}

void out_statis(struct statis * stats) {
    printf("-- statis --\n");
    char * count_name[] = {
        "regular file",
        "fifo file",
        "link file",
        "sock file",
        "directory",
        "char device",
        "block device",
        "\0"
    };

    unsigned long long count_ind[] = {
        stats->file_count, stats->fifo_count, stats->link_count, 
        stats->sock_count, stats->dir_count, stats->chr_count,
        stats->blk_count
    };
    int i=0;
    while (strcmp(count_name[i],"\0")!=0) {
        if (count_ind[i]>0)
            printf("%13s : %llu\n", count_name[i], count_ind[i]);
        i++;
    }

    
    char size_str[64] = {'\0',};
    format_size(stats->total_size, size_str);
    size_str[31] = '\0';
    format_size(stats->file_total_size, size_str+32);
    size_str[63] = '\0';

    if (!_iargs.args[ARGS_REGEX]) {
        printf("%13s : %llu\n", "total count",stats->total_count);
        printf("%13s : %s\n", "total size",size_str);
    }
    printf("%13s : %s\n","total file size",size_str+32);
}


int recur_dir(int deep) {
    struct path_list *pl = &_pathlist;
    struct path_cell *pcell = NULL;
    struct stat stmp;
    struct file_buf fbuf;

    DIR * d = NULL;
    struct dirent *rd = NULL;

    int i,k;
    int cur_height = 1;

    char pathbuf[MAX_NAME_LEN+1] = {'\0'};
    int len_buf = 0;
    int regex_count = 0;
    int stats_flag = 0;
    while (pl) {
 
        for(i=0; i<pl->end_ind; i++) {
            pcell = &pl->pce[i];
            cur_height = pcell->height;
            if (deep > 0 && cur_height > deep)goto end_recur;

            d = opendir(pcell->path);
            if (!d) {
                perror("opendir");
                continue;
            }
            
            _aic.fi.name_max_len = 0;
            _aic.fi.uname_max_len = 0;
            _aic.fi.group_max_len = 0;
            _aic.fi.out_row = 0;

            regex_count = 0;
            while((rd=readdir(d))!=NULL) {
                stats_flag = 1;
                if (!_iargs.args[ARGS_LSALL]
                    && rd->d_name[0] == '.'
                ) {
                    continue;
                }
                strcpy(pathbuf, pcell->path);
                if (!pcell->is_root)
                    strcat(pathbuf, "/");
                
                strcat(pathbuf, rd->d_name);

                if (lstat(pathbuf, &stmp)<0) {
                    perror("lstat");
                    continue;
                }
                if (_iargs.args[ARGS_REGEX]) {

                    if (regexec(_iargs.regcom,rd->d_name,1,_iargs.regmatch,0)!=0)
                    {
                        stats_flag = 0;
                        if (S_ISDIR(stmp.st_mode)) {
                            set_st_fbuf(&fbuf, &stmp, rd->d_name, pathbuf, cur_height+1);
                            set_fbuf_hide(&fbuf, 1);
                            add_to_flcache(&_aic.flcache, &fbuf);
                        } 
                    } else {
                        set_st_fbuf(&fbuf, &stmp, rd->d_name, pathbuf, cur_height+1);
                        add_to_flcache(&_aic.flcache, &fbuf);
                        regex_count += 1;
                    }

                } else {
                    set_st_fbuf(&fbuf, &stmp, rd->d_name, pathbuf, cur_height+1);
                    add_to_flcache(&_aic.flcache, &fbuf);
                }


                if ((_iargs.args[ARGS_STATIS]
                    || _iargs.args[ARGS_SHOWSTATS])
                    && stats_flag
                ) {
                    //printf("stats\n");
                    start_statis(&stmp, &_aic.stats);
                }

                len_buf = strlen(rd->d_name);
                if (_aic.fi.name_max_len < len_buf)
                    _aic.fi.name_max_len = len_buf;

                len_buf = strlen(fbuf.uname);
                if (_aic.fi.uname_max_len < len_buf)
                    _aic.fi.uname_max_len = len_buf;

                len_buf = strlen(fbuf.group);
                if (_aic.fi.group_max_len < len_buf)
                    _aic.fi.group_max_len = len_buf;

            }//end readdir
            closedir(d);
            out_flcache(&_aic.flcache, pcell->path, &_aic.fi);
            add_fbuf_dirs_to_plist(&_aic.flcache, &_pathlist);
            destroy_flcache(&_aic.flcache);
            if (!_iargs.args[ARGS_REGEX])
                printf("\n");
        }//end for
        
        pl = pl->next;
    }

  end_recur:;
    if (_iargs.args[ARGS_STATIS] || _iargs.args[ARGS_SHOWSTATS]) {
        out_statis(&_aic.stats);
    }

    return 0;
}

void help(void);

int main(int argc, char *argv[])
{

    init_infoargs(&_iargs);

    bzero(&_aic.stats, sizeof(_aic.stats));
    bzero(&_aic.fi, sizeof(struct format_info));
    init_flist_cache(&_aic.flcache);
    
    init_path_list(&_pathlist);

    struct winsize _wz;
    ioctl (1, TIOCGWINSZ, &_wz);
    if (_wz.ws_col > 0)
        _aic.fi.win_col = _wz.ws_col;
    if (_wz.ws_row > 0)
        _aic.fi.win_row = _wz.ws_row;

    char path_buffer[MAX_NAME_LEN] = {'\0'};

    struct stat stmp;
    struct file_buf fbuf;
    
    int recur_deep = 1;
    int len_buf = 0;

    for(int i=1;i<argc;i++) {
        if (strncmp(argv[i],"--deep=",7)==0) {
            recur_deep = atoi(argv[i]+7);
        }
        else if (strcmp(argv[i],"--lnk")==0) {
            _iargs.args[ARGS_LINK] = 1;
            _iargs.args[ARGS_OUTMORE] = 1;
        }
        else if (strcmp(argv[i],"--path")==0) {
            _iargs.args[ARGS_PATH] = 1;
            _iargs.args[ARGS_OUTMORE] = 1;
        }
        else if (strcmp(argv[i], "--show-stats") == 0) {
            _iargs.args[ARGS_SHOWSTATS] = 1;
            _iargs.args[ARGS_STATIS] = 1;
        }
        else if (strcmp(argv[i],"--color")==0)
            _iargs.args[ARGS_COLOR] = 1;
        else if (strcmp(argv[i],"--sort")==0)
            _iargs.args[ARGS_SORT] = 1;
        else if(strcmp(argv[i],"--no-file")==0)
            _iargs.args[ARGS_REGNOFIL] = 1;
        else if(strcmp(argv[i],"--no-dir")==0)
            _iargs.args[ARGS_REGNODIR] = 1;
        else if (strncmp(argv[i],"--sort=",7)==0) {
            if (strlen(argv[i])==8) {
                if (argv[i][7] == SORT_BYTM
                    || argv[i][7] == SORT_BYCHTM
                    || argv[i][7] == SORT_BYSIZE
                    || argv[i][7] == SORT_BYNAME
                ){
                    _iargs.args[ARGS_SORT] = argv[i][7];
                } else {
                    _iargs.args[ARGS_SORT] = SORT_BYNAME;
                }
            } else {
                dprintf(2, "Error:unknow sort type\n");
                return 1;
            }
        }
        else if (strcmp(argv[i],"--regex")==0) {
            _iargs.args[ARGS_OUTMORE] = 1;
            i++;
            if (i >= argc) {
                dprintf(2,"Error:less argument -> --regex [REGEX]");
                return 2;
            }
            if (regcomp(_iargs.regcom, argv[i], REG_EXTENDED|REG_ICASE)!=0) {
                dprintf(2, "Error: compile regex -> %s\n",argv[i]);
                perror("regcomp");
                return 2;
            }
            _iargs.args[ARGS_REGEX] = 1;
        }
        else if (strcmp(argv[i], "-h")==0) {
            help();
            return 0;
        }
        else if (argv[i][0]=='-') {
            len_buf = strlen(argv[i]);
            for(int a=1; a<len_buf; a++){
                if (argv[i][a]=='a')
                    _iargs.args[ARGS_LSALL] = 1;
                else if (argv[i][a] == 'i') {
                    _iargs.args[ARGS_INO] = 1;
                    _iargs.args[ARGS_OUTMORE] = 1;
                }
                else if (argv[i][a] == 'm') {
                    _iargs.args[ARGS_MODE] = 1;
                    _iargs.args[ARGS_OUTMORE] = 1;
                }
                else if (argv[i][a] == 'p') {
                    _iargs.args[ARGS_PATH] = 1;
                    _iargs.args[ARGS_OUTMORE] = 1;
                }
                else if (argv[i][a] == 's') {
                    _iargs.args[ARGS_SIZE] = 1;
                    _iargs.args[ARGS_OUTMORE] = 1;
                }
                else if (argv[i][a] == 't')
                    _iargs.args[ARGS_STATIS] = 1;
                else if (argv[i][a] == 'f')
                    _iargs.args[ARGS_TYPE] = 1;
                else if (argv[i][a] == 'l') {
                    _iargs.args[ARGS_LONGINFO] = 1;
                    _iargs.args[ARGS_OUTMORE] = 1;
                }
                else if (argv[i][a] == 'r') {
                    _iargs.args[ARGS_RECUR] = 1;
                    recur_deep = 0;
                }
                else if (argv[i][a] == 'c') {
                    _iargs.args[ARGS_CREATM] = 1;
                    _iargs.args[ARGS_OUTMORE] = 1;
                }
                else { //不匹配则可能是目录/文件名称
                    
                }
            }
        }
        else if (access(argv[i], F_OK)==0) {
            if (lstat(argv[i],&stmp)<0)
                continue;

            len_buf = strlen(argv[i]);
            if (len_buf >= MAX_NAME_LEN) {
                dprintf(2,"Error: the length of path too long -> %s\n",argv[i]);
                continue;
            }
            if (S_ISDIR(stmp.st_mode)) {
                if (len_buf > 1 && argv[i][len_buf-1]=='/') {
                    strncpy(path_buffer, argv[i], len_buf-1);
                    path_buffer[len_buf-1] = '\0';
                } else {
                    strcpy(path_buffer, argv[i]);
                }
                add_path_list(&_pathlist, path_buffer, 1);
            } else {
                set_st_fbuf(&fbuf, &stmp, argv[i], NULL, 1);
               
                if (_aic.fi.name_max_len < len_buf)
                    _aic.fi.name_max_len = len_buf;
                
                len_buf = strlen(fbuf.uname);
                if (_aic.fi.uname_max_len < len_buf)
                    _aic.fi.uname_max_len = len_buf;

                len_buf = strlen(fbuf.group);
                if (_aic.fi.group_max_len < len_buf)
                    _aic.fi.group_max_len = len_buf;

                add_to_flcache(&_aic.flcache, &fbuf);
            }
        }
        else
            dprintf(2, "Error:unknow arguments -> %s\n", argv[i]);
    }

    if (_pathlist.end_ind == 0 && _aic.flcache.end_ind == 0)
        add_path_list(&_pathlist, ".", 1);
    
    if (_aic.flcache.end_ind > 0) {
        out_flcache(&_aic.flcache, "", &_aic.fi);
    }

    recur_dir(recur_deep);

    destroy_path_list(_pathlist.next);
    destroy_flcache(&_aic.flcache);

    return 0;
}

#define HELP_INFO   "\
    --color : 支持颜色输出\n\
    --lnk : 如果是软链接输出目标文件\n\
    --sort : 排序，默认会开启\n\
    --deep : 递归深度，--deep=[NUMBER]\n\
    \n\
    -h : 帮助文档\n\
    -l : 长格式输出{mode hard-link user group filename size [target]}\n\
    -c : 创建时间\n\
    -a : 显示隐藏文件\n\
    -r : 递归显示目录\n\
    -s : 文件大小\n\
    -p : 显示路径\n\
"

void help() {
   printf("%s manual\n", PROGRAM_NAME);
   printf("%s\n", HELP_INFO);
   printf("  type flag:\n");
   list_type_info();
}
```

