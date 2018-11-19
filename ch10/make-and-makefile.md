## 编译多个文件

通常一个小规模程序放在一个.c文件中即可。对于规模稍大的软件，这肯定是行不通的。这节课的主题主要是研究如何编译多个文件。

习惯了Windows平台IDE的开发环境，对本节课的内容要转变一些想法，可视化的IDE在创建项目的时候多个文件会自动编译成一个可执行文件，隐藏掉了细节，如果要在编译时加入扩展选项，需要在单独的设置窗口设置，但是对于其他项目就要去掉，反复操作会很麻烦，或者好的IDE可以对每个项目有独立的配置文件。但是总归会丧失掉一些灵活性。

我们先来看一个简单的示例，这个示例是把之前的控制终端的一些操作封装成了函数作为单独的一部分，并有两个文件：ctty.h和ctty.c。

ctty.h文件内容如下：

```c
#ifndef MY_CTTY_H
#define MY_CTTY_H

//获取终端的行列数
int * winsize(int *ws);

//重置终端属性
void tty_reset();

void clear();

void move(int row, int column);

//移动光标到指定位置输出字符串
void mvoutstr(int row, int column, char *s);

//移动光标到指定位置输出字符
void mvoutc(int row, int column, char c);

//允许滚屏
void enable_scroll();

//允许部分区域滚屏
void scroll_zone(int start, int end);

/*
    滚屏操作
*/
void scroll_down(int lines);

void scroll_up(int lines);

/*
 *  光标移动操作
*/

void cursor_left(int count);

void cursor_right(int count);

void cursor_up(int count);

void cursor_down(int count);


#endif

```

ctty.c文件内容如下：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <termios.h>
#include "ctty.h"

//获取终端的行列数
int * winsize(int *ws) {
    ws[0] = ws[1] = 0;

    struct winsize _wz;
    ioctl (1, TIOCGWINSZ, &_wz);

    if (_wz.ws_col > 0)
        ws[1] = _wz.ws_col;
    if (_wz.ws_row > 0)
        ws[0] = _wz.ws_row;

    return ws;
}

//重置终端属性
void tty_reset(){
    printf("\033c");
}

void clear() {
    printf("\x1b[2J\x1b[;H");
}

void move(int row, int column) {
    printf("\x1b[%d;%dH", row, column);
}

//移动光标到指定位置输出字符串
void mvoutstr(int row, int column, char *s) {
    move(row, column);
    printf("%s", s);
    fflush(stdout);
}

//移动光标到指定位置输出字符
void mvoutc(int row, int column, char c) {
    move(row, column);
    printf("%c", c);
    fflush(stdout);
}

//允许滚屏
void enable_scroll() {
    printf("\x1b[r");
    fflush(stdout);
}

//允许部分区域滚屏
void scroll_zone(int start, int end) {
    printf("\033[%d;%dr", start, end);
    fflush(stdout);
}

/*
    滚屏操作
*/
void scroll_down(int lines) {
    for(int i=0; i<lines; i++) {
        printf("\033D");
    }
    fflush(stdout);
}

void scroll_up(int lines) {
    for(int i=0; i<lines; i++) {
        printf("\x1bM");
    }
    fflush(stdout);
}

/*
 *  光标移动操作
*/

void cursor_left(int count) {
    printf("\e[%dD", count);
    fflush(stdout);
}

void cursor_right(int count) {
    printf("\e[%dC", count);
    fflush(stdout);
}

void cursor_up(int count) {
    printf("\e[%dA", count);
    fflush(stdout);
}

void cursor_down(int count) {
    printf("\e[%dB", count);
    fflush(stdout);
}

```

main.c文件：

```c
#include <stdio.h>
#include <stdlib.h>
#include "ctty.h"

int main(int argc, char *argv[]) {

    clear();
    mvoutstr(5, 12, "I like Linux\n");

	return 0;
}

```

 编译方式：

gcc -o main ctty.c main.c

还可以展开成详细的过程，先编译成.o文件，然后再把.o文件链接成完整的程序：

gcc -c  ctty.c

gcc -c  main.c

gcc -o  main  ctty.o  main.o

如果你仔细考虑，问题已经出现了，如果文件数量过多，这种手动编译的方式非常麻烦，而且容易出错，每一步都需要你考虑依赖关系。为了解决这个问题，出现了make工具。

make是GNU发布的程序自动编译的软件，make是把我们手动编译的命令自动逐条执行。

#### make和makefile

make在执行时，会自动在当前目录查找makefile或Makefile文件，并执行此文件。如果文件不是名命为makefile，可以通过-f参数指定。在makefile文件中，指定了每一步编译的命令，依赖关系，最终会链接成一个程序。如果在此过程中出错，make会报错并退出。

接下来我们就看看如何编写makefile文件，makefile文件主要表明依赖规则，提供编译命令，make在执行makefile时会自动运行。如果把之前的编译写成makefile：

```makefile
main: main.o ctty.o
	gcc -o main main.o ctty.o
ctty.o: ctty.c ctty.h
	gcc -c ctty.c
main.o: main.c ctty.h
	gcc -c main.c
	
```

首先要说明的是，在makefile中，缩进一定要使用TAB键，不要转换成空格。当我们输入make命令的时候，make会找到makefile文件，首先看到目标文件main，依赖于main.o和ctty.o。

如果目标文件不存在或者依赖的文件创建时间比目标文件新，则会执行后面的命令，命令一定是要缩进的。

继续后面的过程，make会检查依赖的.o文件是否存在，这样逐层检查，直到所有的.o文件被建立。最后生成main程序。

在makefile中可以加入clean标记，这样输入make  clean会执行clean后面的命令用于删除已编译的文件。

```makefile
main: main.o ctty.o
	gcc -o main main.o ctty.o
ctty.o: ctty.c ctty.h
	gcc -c ctty.c
main.o: main.c ctty.h
	gcc -c main.c

clean:
	rm main.o ctty.o main
```

这种方式简化了每次编译的操作，但是makefile编写还是比较麻烦，而且，如果需要加入一个.o文件，在每个依赖的地方都要加入，make提供了变量的功能来方便操作：

```makefile
objs = main.o ctty.o
main: $(objs)
	gcc -o main $(objs)
main.o: main.c ctty.h
	gcc -c main.c
ctty.o: ctty.c ctty.h
	gcc -c ctty.c
	
clean:
	rm $(objs) main
```

make是一个很强大的工具，具备自动推导的能力。make发现.o文件会自动把.c文件纳入依赖规则。所以还可以简化makefile的编写：

```makefile
objs = main.o  ctty.o

main: $(objs)
	gcc -o main $(objs)

clean:
	rm $(objs) main
```

更进一步，可以省略目标文件的编译命令：

```makefile
objs = main.o  ctty.o

main: $(objs)

clean:
	rm $(objs) main
```

在makefile中只有行注释，使用#开头。

**引入其他文件**

如果过于复杂，在makefile中可以通过include  引入其他文件： include  a.mk

