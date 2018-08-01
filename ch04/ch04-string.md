## 文本处理

### 字符串基本处理函数

strlen：计算字符串长度

strcmp：比较字符串

strcat：把后一个字符串的值，放在前一个字符串的末尾

strcpy：把后一个字符串的值复制到目标字符串

同时这些函数都有另一个版本，strn...，比如strncmp用于比较前n个字符，这些函数都比较简单，通过man 3 str...可以查看对应的函数联机文档。这部分内容不作为本节课重点内容，如果你不熟悉可以再去编程实际操作看看结果。

### 字符串转换成数字

一个经常用到的功能是如何把字符串转换成数字，比如"ac123d"转换成数字是123，"13.4ae"转换成数字是13.4。

库函数strtod，strtold，strtof，strtol可以完成这些操作。通过查看手册发现第一个参数是要转换的字符串，第二个是要保存的值，但是可以传递NULL参数，返回值是转换的数字，strtol会转换整数，还有第三个参数是要转换的进制。

```c
STRTOD(3)                           Linux Programmer's Manual                           STRTOD(3)

NAME
       strtod, strtof, strtold - convert ASCII string to floating-point number

SYNOPSIS
       #include <stdlib.h>

       double strtod(const char *nptr, char **endptr);
       float strtof(const char *nptr, char **endptr);
       long double strtold(const char *nptr, char **endptr);

   Feature Test Macro Requirements for glibc (see feature_test_macros(7)):

       strtof(), strtold():
           _ISOC99_SOURCE || _POSIX_C_SOURCE >= 200112L

DESCRIPTION
       The  strtod(), strtof(), and strtold() functions convert the initial 
       portion of the string pointed to by nptr to double, float, and long 
       double representation, respectively.

......
......
```

```c
STRTOL(3)                           Linux Programmer's Manual                           STRTOL(3)

NAME
       strtol, strtoll, strtoq - convert a string to a long integer

SYNOPSIS
       #include <stdlib.h>

       long int strtol(const char *nptr, char **endptr, int base);

       long long int strtoll(const char *nptr, char **endptr, int base);

   Feature Test Macro Requirements for glibc (see feature_test_macros(7)):
......
......
```

使用示例：

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char* argv[]) {
    for (int i=1; i<argc; i++) {
        printf("%f %d\n",strtod(argv[i], NULL),strtol(argv[i], NULL, 10));
    }
    printf("\n");
    return 0;
}
```

这段程序把参数转换为数字，并输出结果。你可以输入不同的参数，会发现如果是"a123.4"类型的字符串转换为0，只有数字开头的字符串才会转换为数字，截止到非数字字符或者是字符串结尾。比如，"23w34.5"会转换为23。如果你想把"a123.4"这样的字符串转换为123.4，可以把指针位置移动到数字开头，对于"ab-123.24"这样的形式移动到 '-'处即可。







### 字符串分割

现在考虑另一个在实际开发中要进行的操作：如何把一个字符串用另一个字符串分割成数组。比如，"axdefrtyucovrtkp"使用"rt"分割之后变成"axdef","yucov","kp"。



### 字符串匹配：BM算法

学过数据结构和算法的课程，应该对KMP算法有所了解，但是KMP只具有理论上的性能，实际 测试并不是很理想，由于Donald Ervin Knuth是此算法发明者之一，所以名气很大。但是字符串匹配最快的算法是BM算法。BM无论是理论性能还是实际测试都是最好的。





### 正则表达式

