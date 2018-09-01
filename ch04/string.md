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
    The  strtod(), strtof(), and strtold() functions convert the initial portion of the string pointed to by nptr to double,float, and long double representation, respectively.

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

       long long int strtoll(const char *nptr, char **endptr, int 
       base);

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
        printf("%f %d\n",strtod(argv[i],NULL),strtol(argv[i],NULL, 10));
    }
    printf("\n");
    return 0;
}
```

这段程序把参数转换为数字，并输出结果。你可以输入不同的参数，会发现如果是"a123.4"类型的字符串转换为0，只有数字开头的字符串才会转换为数字，截止到非数字字符或者是字符串结尾。比如，"23w34.5"会转换为23。如果你想把"a123.4"这样的字符串转换为123.4，可以把指针位置移动到数字开头，对于"ab-123.24"这样的形式移动到 '-'处即可。

库函数atoi，atoll，atof同样可以完成这些操作。查看库函数手册得知，调用atoi函数相当于调用：

&emsp;&emsp;strtol(str, NULL, 10);

&emsp;&emsp;strtol(str, NULL, 10);

调用aotf相当于调用：

&emsp;&emsp;strtod(str, NULL);

调用atoll相当于调用：

&emsp;&emsp;strtoll(str, NULL, 10);



### 字符串分割

现在考虑另一个在实际开发中要进行的操作：如何把一个字符串用另一个字符串分割成数组。比如，"axdrtyucrtkp"使用"rt"分割之后变成"axd","yuc","kp"。在本课程之前你应该已经熟悉一些Python的基础使用，在Python环境中，字符串'abcdeabcdef'可以直接使用： &emsp;&emsp;'abcdefgabcdefghi'.split('cd')

分割成 ['ab', 'eab', 'ef']。

在PHP语言中，使用:

&emsp;&emsp;explode(':','/usr/bin:/usr/sbin:/bin:/sbin');

可以把字符串分割成数组 ['/usr/bin','/usr/sbin','/bin','/sbin']。

在C语言中没有这样高级的操作，库函数strtok提供了类似的操作。但是strtok函数不是把字符串分割为数组。而是把要按照分割子串，把源字符串对应的部分设置为'\0'，每次返回下一个字符串首地址，直到源字符串末尾或是没有分割字串。通过man 3 strtok查看库函数手册：

```c
STRTOK(3)                           Linux Programmer's Manual                           STRTOK(3)

NAME
       strtok, strtok_r - extract tokens from strings

SYNOPSIS
       #include <string.h>

       char *strtok(char *str, const char *delim);

       char *strtok_r(char *str, const char *delim, char **saveptr);
	......
    ......
DESCRIPTION
    The strtok() function breaks a string into a sequence of zero or more nonempty tokens.  On the first call to strtok(), the string to be parsed should be specified in str.   In  each subsequent call that should parse the same string, str must be NULL.

    The delim argument specifies a set of bytes that delimit the tokens in the parsed string. The caller may specify different strings in delim in successive calls that parse the  same string.

    Each  call  to  strtok() returns a pointer to a null-terminated string containing the next token.  This string does not include the delimiting byte.  If no more  tokens  are found,strtok() returns NULL.

    A  sequence  of calls to strtok() that operate on the same string maintains a pointer that determines the point from which to start searching for the next token.  The first call  to strtok() sets this  pointer  to point to the first byte of the string.  The start of the next token is determined by scanning forward for the next nondelimiter byte  in  str.   If such a byte is found, it is taken as the start of the next token.  If no such byte is found, then there are no more tokens, and strtok() returns NULL.  (A string that is  empty or  that  contains  only  delimiters  will thus cause strtok() to return NULL on the first call.)

    The end of each token is found by scanning forward until either the next delimiter byte is found  or  until  the terminating null byte ('\0') is encountered.  If a delimiter byte is found, it is 
overwritten with a null byte to terminate the  current  token, and 
strtok() saves  a  pointer  to  the following byte; that pointer 
will be used as the starting point when searching for the next token.  In this case, strtok() returns a pointer to the  start of the found token.

    From the above description, it follows that a sequence of two or more contiguous delimiter bytes in the parsed string is considered to be a  single  delimiter,  and  that  delimiter bytes at the start or end of the string are ignored.  Put another way: the tokens returned by strtok() are always nonempty strings.  Thus, for example, given the string "aaa;;bbb,", successive  calls  to  
strtok()  that  specify  the delimiter string ";," would return the
strings "aaa" and "bbb", and then a null pointer.
       ......
       ......
RETURN VALUE
   The strtok() and strtok_r() functions return a pointer to the next token, or NULL if there are no more tokens.

```

从这个函数的说明手册发现，strtok和其他调用有些不同。比如，要使用","分割字符串 "Linux,Unix,Windows"，初次调用是这样的形式：

```c
char src[1024] = "Linux,Unix,Windows";
char * sub = NULL;
sub = strtok(src, ",");
```

但是再次调用，由于还是对src进行分割，就是这样的形式：

```C
sub = strtok(NULL, ",");
```

如果strtok返回NULL则分割结束。到这里，你可能会有疑惑，我们先来看一个完整的示例：

```c
#include <string.h>
#include <stdlib.h>
#include <stdio.h>

int main(int argc, char *agrv[]) {
    char src[1024] = "Linux,Unix,Windowx,FreeBSD";
    char * dem = ",";
    char * sub = NULL;

    sub = strtok(src, dem);
    while (sub!=NULL) {
        printf("%s\n", sub);
        sub = strtok(NULL, dem);
    }

    return 0;
}
```

这个程序会输出：

```c
Linux
Unix
Windows
FreeBSD
```

现在我们来探索更多的细节，首先要知道的是，strtok会改变源字符串的值，如果在一开始使用strlen计算src的长度并保存，在strtok调用之后，根据src最开始的长度循环输出每个字符的数值，会发现','已经变成了'\0'，strtok是在src上操作，每次调用返回对应的指针位置。strtok调用完之后，src会变成这样："Linux\0Unix\0Windows\0FreeBSD"。

另一个比较疑惑的是，strtok除第一次调用以外，第一个参数要传递NULL表示使用第一次调用传递的字符串。这意味着strtok知道首次调用的源字符串地址，我们能想到的方式就是使用一个static变量保存首次调用传递的源字符串地址，以下代码是glibc的实现：

```c
//file : strtok.c
char *
strtok (char *s, const char *delim)
{
  static char *olds;
  return __strtok_r (s, delim, &olds);
}

// file : strtok_r.c
......
#ifndef _LIBC
/* Get specification.  */
# include "strtok_r.h"
# define __strtok_r strtok_r
#endif

/* Parse S into tokens separated by characters in DELIM.
   If S is NULL, the saved pointer in SAVE_PTR is used as
   the next starting point.  For example:
	char s[] = "-abc-=-def";
	char *sp;
	x = strtok_r(s, "-", &sp);	// x = "abc", sp = "=-def"
	x = strtok_r(NULL, "-=", &sp);	// x = "def", sp = NULL
	x = strtok_r(NULL, "=", &sp);	// x = NULL
		// s = "abc\0-def\0"
*/
char *
__strtok_r (char *s, const char *delim, char **save_ptr)
{
  char *end;

  if (s == NULL)
    s = *save_ptr;

  if (*s == '\0')
    {
      *save_ptr = s;
      return NULL;
    }

  /* Scan leading delimiters.  */
  s += strspn (s, delim);
  if (*s == '\0')
    {
      *save_ptr = s;
      return NULL;
    }

  /* Find the end of the token.  */
  end = s + strcspn (s, delim);
  if (*end == '\0')
    {
      *save_ptr = end;
      return s;
    }

  /* Terminate the token and make *SAVE_PTR point past it.  */
  *end = '\0';
  *save_ptr = end + 1;
  return s;
}

```

注意strtok函数以及__strtok_r函数的最开始，就可以明白strtok的实现方式。

另一个需要注意的问题是，分割字符串是一个字符集，比如使用";,"分割字符串 "Linux;Unix,Windows"，那么';'和','都会分割成'\0'，";,"也会分割成"\0\0"。这点和Python的实现不同。





