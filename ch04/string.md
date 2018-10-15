## 文本处理

### 字符串基本处理函数



**字符串常用函数原型和说明：**

| 函数                                                         |
| ------------------------------------------------------------ |
| size_t strlen(const char *s);<br>计算字符串长度。            |
| char* strchr(const char *s, char c);<br>返回字符c在s中首次出现的位置，如果没有发现返回NULL。 |
| char* strcpy(char *dest, const char *src);<br>复制字符串src到dest，返回指针指向dest。 |
| char* strncpy(char \*dest, const char \*src, size_t n);<br>从src复制n个字节到dest。 |
| char* strcat(char \*dest, const char \*src)<br>把src放到dest末尾。 |
| char* strncat(char \*dest, const char \*src, size_t n)<br>把src的前n个字节放到dest末尾。 |
| char * strcmp(const char \* s1, const char \*s2);<br>比较两个字符串。 |
| char * strncmp(const char \*s1, const char \*s2, size_t n);<br>比较前n个字节。 |
| char * strstr(const char *haystack,  const char *needle);<br>查找字符串needle在haystack中首次出现的位置，找到返回haystack中needle字串的首地址，否则返回NULL。 |
|                                                              |

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

如果第二个参数不为空，则需要传递一个二级指针\*\*ep，\*ep指向的是上一次转换的字符串。

### 字符串分割

现在考虑另一个在实际开发中要进行的操作：如何把一个字符串用另一个字符串分割成数组。比如，"axdrtyucrtkp"使用"rt"分割之后变成"axd","yuc","kp"。在本课程之前你应该已经熟悉一些Python的基础使用，在Python环境中，字符串'abcdeabcdef'可以直接使用：

&emsp;&emsp;'abcdefgabcdefghi'.split('cd')

分割成 ['ab', 'eab', 'ef']。

在PHP语言中，使用:

&emsp;&emsp;explode(':','/usr/bin:/usr/sbin:/bin:/sbin');

可以把字符串分割成数组 ['/usr/bin','/usr/sbin','/bin','/sbin']。

在C语言中没有这样高级的操作，库函数strtok提供了类似的操作。但是strtok函数不是把字符串分割为数组。而是按照分割子串，把源字符串对应的部分设置为'\0'，每次返回下一个字符串首地址，直到源字符串末尾或是没有分割字串。通过man 3 strtok查看库函数手册：

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

源代码的实现方式验证了我们的想法。

另一个需要注意的问题是，分割字符串是一个字符集，比如使用";,"分割字符串 "Linux;Unix,Windows"，那么';'和','都会分割成'\0'，";,"也会分割成"\0\0"。这点和Python的实现不同。



### 字符串移位与包含

给定两个字符串s1和s2，检测s2是否可以通过s1的循环移位得到字符串s2或包含s2。比如AACDB和CDBA。AACDB可以移位成ACDBA包含CDBA。而AACDB和CDAA，AACDB无法通过移位包含CDAA。

编程实现函数：int  strrotstr(char *s1, char *s2);如果s1循环移位可以包含s2则返回1,否则返回0,出错返回-1。

最直接的方式就是对s1循环移位所有的结果，并调用strstr函数确定s2是否在s1中。

但是如果我们不采用常规的循环移位方式，那需要大量的交换数据的操作，而是把移位的字符串直接放在后面，可以发现：

AACDB -> AACDBA -> AACDBAA -> AACDBAAC ->AACDBAACD -> AACDBAACDB

注意到AA CDBA ACDB已经包含了CDBA。所以通过牺牲一些存储空间来提升性能。实际操作方式很简单：

申请2 × strlen(s1) + 1的内存，+1 是防止strlen(s1) 为0。然后把s1s1放到申请的内存中，一次调用strstr即可。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>


int strrotstr(char *s1, char *s2) {
    char *buf = (char*)malloc(sizeof(char)*strlen(s1)*2 + 1);
    if (buf == NULL) {
        return -1;
    }

    int is_in = 0;

    strcpy(buf, s1);
    strcat(buf, s1);

    if (strstr(buf, s2))
        is_in = 1;

    free(buf);

    return is_in;
}


int main(int argc, char *argv[]) {

    if (argc < 3) {
        dprintf(2, "Error: string at least 2\n");
        return -1;
    }

    printf("%d\n", strrotstr(argv[1], argv[2]));

	return 0;
}
```



### \*最小正则表达式

可能你没有接触过正则表达式，但是你应该知道在Windows上使用的一些通配符：*  ?。shell中也支持这两个通配符。这种操作方式已经有正则表达式的思想。正则表达式提供的是一种匹配模式，如果仅仅是在一行文本中匹配一个字符串，功能很受限制。比如：你要在硬盘中搜索所有含有linux的扩展名为pdf的文件。正则表达式为这样的场景提供了很好的支持，通过一些元字符用于标记匹配的行为。比如使用.表示匹配任意一个字符，*表示前面的匹配出现0到多次，$表示尾部匹配...

正则表达式是一个强大的工具，在软件领域，正则表达式已经成为不可缺失的一部分。即使你不直接使用，各种工具都会用到正则表达式。

正则表达式是从Unix软件开始普及起来的。Ken Thompson（肯·汤姆逊，Unix之父）最早实现了正则表达式的工具，他开发的ed编辑器支持正则表达式匹配，后来演变出了grep命令，这个命令在Liux/Unix中使用频率非常高。

目前正则表达式支持的功能非常多，但不是所有的功能你都能用到，以下是一些正则表达式的示例：

| 模式和说明                                                   |
| ------------------------------------------------------------ |
| ^\[a-zA-Z\]\[a-zA-Z0-9\]{5,15}$<br>匹配字母开头只含有字母和数字并且长度在6~16的字符串，可用于用户名检测。 |
| ^1\[1-9\]?\[0-9\]{9}$<br>检测是否为合法的手机号格式。        |
| linux.*\.pdf$<br>匹配所有名称含有linux并且中间出现任意字符，扩展名为.pdf的文件。 |

这些看起来有些古怪的表达式对于程序来说也不过是一个普通的字符串，对于要进行正则匹配的程序来说，像 ^ . * [] {} $ 都是有特殊含义的字符，要区别对待。

现在很多种语言都提供了正则表达式的支持。C语言也有相关的库可以调用。我们在这里实现一个最小的正则表达式程序来展示匹配原理，另一方面你可以体会到递归的强大。

| 元字符 | 说明                                             |
| ------ | ------------------------------------------------ |
| ^      | 从开头开始匹配                                   |
| .      | 匹配任意一个字符                                 |
| *      | 前一个匹配出现任意次数                           |
| $      | 尾部匹配，文本不是刚好在末尾处匹配成功则表示失败 |
| c      | 普通字符                                         |

示例程序采用了三个函数实现正则匹配，对于编程来说，只需要调用match函数，matchreg和matchchar函数是匹配过程内部调用的。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define REGEX_CHAR  ".^*$"

int match(char *, char *);
int matchreg(char *, char *);
int matchchar(char, char *, char *);

int match(char *regex, char *text) {

    /*
    	如果是匹配行首则从regex下一个字符开始和text进行匹配，
    	否则如果regex[0]不属于元字符就让text++直到指向的字符
    	和regex[0]相等，之后使用matchreg进行匹配。
    */
    if (regex[0] == '^')
        return matchreg(regex+1, text);
    else if(strchr(REGEX_CHAR, regex[0]) == NULL) {
        while(*text != '\0' && *text != regex[0])
            text++;
    }
    return matchreg(regex, text);
}

int matchreg(char *regex, char *text) {

    //正则结束表示匹配成功
    if (regex[0] == '\0')
        return 1;

    //如果是尾部匹配，则判断*text是否使字符串末尾并返回比较的值。
    if (regex[0] == '$' && regex[1] == '\0')
        return *text == '\0';

    /*
    	*表示字符regex[0]可以出现0次或多次，这还要区分两种情况：
    	1. 如果是 . 则表示任意字符，这时候让循环匹配regex+2开始的
    	正则和text，并让text++，只要有一个能匹配成功就返回1，
    	因为之前的字符都属于 .*匹配的范围；
    	2. 如果regex[0]是普通字符，则只要有和regex[0]相同的字符都要匹配，
    	但是要先进行后面的匹配，如果后续的正则可以匹配，就检测和regex[0]相同的字符
    	是不是刚好和后续的正则能对接。比如：正则'a*ab' 要能够匹配'aaaab'、'aab'，
    	但是'acab'无法匹配。
    */
    if (regex[1] == '*') {
        if (regex[0]=='.') {
            while(*text!='\0') {
                if (matchreg(regex+2, text))
                    return 1;
                text++;
            }
        } else {
            char c = regex[0];
            char *tbuf = text;
            while(*tbuf != '\0') {
                if (matchreg(regex+2, tbuf)) {
                    while(*text!='\0' && text!=tbuf && *text==c) {
                        text++;
                    }

                    if (text==tbuf)
                        return 1;

                    return 0;
                }
                tbuf++;
            }
        }
        return matchreg(regex+2, text);

    }
	/*
		检测 . 要在 *之后，否则在 .*这种情况的时候会出现问题。
	*/
    if (regex[0] == '.' && *text!='\0') {
        text++;
        return matchreg(regex+1, text);
    }

    //之前的条件都不符合就是普通字符的匹配过程
    return matchchar(regex[0], regex, text);
}

int matchchar(char c, char *regex, char *text) {
	/*
		因为在循环中regex[0]也就是c已经等于*text，
		从regex+1开始和text++之后的字符串匹配。
		注意：这里涉及到的一种情况是如果匹配不成功，需要回溯到之前的数据匹配，
		否则会出现匹配不符合直观理解的结果，比如：
		  正则表达式 unix.pdf$ 和字符串 ununix.pdf，此时如果匹配到第
		  三个字符就会出现不匹配的情况，此时如果我们自己观察就会发现将
		  正则表达式和字符串第三个字符开始匹配就可以成功，
		  所以在else逻辑中调用match，注意此时text是已经进行++操作的，
		  match会根据regex把text移动到第一个字符相同的位置开始匹配。
	*/
    while(*text != '\0' && *text++ == c) {
        if (matchreg(regex+1, text))
            return 1;
        else
            return match(regex,text);
    }

    return 0;
}


int main(int argc, char *argv[]) {

    if (argc < 2) {
        dprintf(2, "Error: less regexp\n");
        return -1;
    }

    char buffer[2048] = {'\0'};
    int len;

    while(1) {
        fgets(buffer, 2000, stdin);
        len = strlen(buffer);
        if (buffer[len-1]=='\n')
            buffer[len-1] = '\0';

        if (match(argv[1], buffer)) {
            printf("match: %s\n", buffer);
        } else {
            printf("[%s] not match\n", buffer);
        }
    }

	return 0;
}
```

