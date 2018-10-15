## 递归

----

递归在软件编程中被广泛使用，有时候采用递归会使程序编写简化并能完美的体现设计思想或处理流程。需要注意的是，某些场景虽然可以使用递归解决，但并不合适。

>**如何解释递归：要理解递归你要先理解递归。**

递归是函数调用自身的情况，在某些场景下，递归可以轻松解决复杂的问题。编写递归函数一定要有一个或多个终止条件。

>递归调用：f(n) -> f(n-1) .......-> f(0)
>
>函数返回：f(0) -> f(1) ...... -> f(n)



### 滥用递归的情况

通常的示例总会去求解斐波那契数列、1~n的和、阶乘...，尽管可以解决问题，但是这种示例其实并不好，属于对递归的滥用。这几个示例的解法简单，不再以代码演示。

一个简单的原则是：如果递归能清晰描述运行原理，并且系统资源占用在合理范围内，则可以使用递归。如何把握需要根据业务场景，硬件资源等因素确定。

### 递归经典示例：快速排序

这里给出的示例是实现整数快速排序的递归实现，并且采用C语言实现。你在接触这部分内容时应该已经学过快速排序算法。我们会在这一部分实现快速排序并在后面实现ls命令的时候使用。快速排序在数据量很大的时候算法性能非常好，平均情况来看，其函数递归深度仍然能够控制在比较低的范围内。

快速排序的思想简洁优雅，但实现起来，还是有一些需要仔细考虑的地方。一个是枢纽元（另一种翻译为主元）的选取，另一个根据枢纽元对数据分区的方式。以下代码是一个快速排序的实现，实现方式在选区枢纽元的时候采用了中间位置取值，并把此值和start位置数据交换。之后从start+1开始和d[start]比较，并使用i记录分组位置。代码在交换数据的时候没有使用函数，而是使用宏定义展开。

```c
#define SWAP(a,b)   tmp=a;a=b;b=tmp;

void qsorti(int *d, int start, int end) {
    if (start >= end) {
        return ;
    }
    int med = (end+start)/2; //选择枢纽元位置
    int i = start;
    int j = end;
    int tmp = 0;
	
    SWAP(d[med],d[start]); //把枢纽元和第一个元素交换位置
    
    for(j=start+1;j<=end;j++) {
        /*
        	每个元素和枢纽元比较，如果小于枢纽元则先进行i++，然后
        	交换d[j]和d[i]。
        	i的作用很关键，记录位置，从start开始，通过交换位置
        	把数据按大小分类。
        	数据分类的巧妙之处在于，j从i+1的位置开始，i和j都会向后移动，
        	j肯定是>=i，如果d[j]<d[start]，d[i]一定是 >= d[start]，
        	因为如果不是，在此之前，j已经走过i的位置，已经发生了数据交换。
        	在执行SWAP(d[i], d[j])之后，d[i]以及之前的数据都是小于d[start]。
        */
        if (d[j]<d[start]) { 
            i++;
            SWAP(d[i],d[j]);
        }
    }
    //for循环结束，d[i]一定是 <= d[start]，所以交换它们的位置
	SWAP(d[i],d[start]);
    
    qsorti(d, start, i-1);
    qsorti(d, i+1,end);
}
```

### 字符串快速排序

对于数字进行快速排序，大小直接进行比较即可，对于字符串的比较，需要使用字符串比较函数，这样之前的程序要做以下修改：

* 使用字符串比较函数进行比较
* 交换两个变量的值改为指针类型

实现代码如下：

```c
#define SWAP(a,b)   tmp=a;a=b;b=tmp;

void qsortstr(char *d[], int start, int end) {
    if (start >= end) {
        return ;
    }
    int med = (end+start)/2;
    int i = start;
    int j = end;
    char *tmp = NULL;
    
    SWAP(d[med],d[start]);

    for(j=start+1;j<=end;j++) {
        if (strcmp(d[j],d[start]) < 0) {
            i++;
            SWAP(d[i],d[j]);
        }
    }
    
    SWAP(d[i],d[start]);
    qsortstr(d, start, i-1);
    qsortstr(d, i+1,end);
}
```



### 输出树形结构

现在来使用递归输出以下树形结构：

![](bintree.jpg)



```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void out_tree(int deep, int move) {
    if (deep <= 0)return;

    char fmt_str[16] = {'\0'};

    sprintf(fmt_str, "%%-%ds", move);
    printf(fmt_str,"");
    printf("|---T\n");
    
    out_tree(deep-1, move+4);
    out_tree(deep-1, move+4);
}

int main(int argc, char *argv[]) {

    out_tree(4, 0);

    return 0;
}
```

使用递归的方式可以输出目录树结构。



### mkdir自动创建父级目录

在上一章是实现了mkdir命令。Linux自带的mkdir命令还有一个功能：如果父级目录不存在则自动创建。通过man mkdir查看命令手册发现-p选项开启此功能。如果在$HOME/tmp目录下创建linux/ubuntu目录。但是linux/和ubuntu/都不存在。自动创建linux目录，并且在linux/目录创建ubuntu。这个功能该如何实现。

考虑更复杂的情况：a/b/c/d/e。这种层级如果每个目录都不存在程序如何自动创建？

考虑到这章的主题，肯定是要使用递归实现的。现在考虑如何实现：

**如果从某一级目录开始不存在，则其子目录肯定都不存在，所以要从此一级目录开始创建，并一级一级的创建子目录。而要确定哪一层级的目录不存在，可以从路径字符串尾部开始向前递减，直到/字符截止，然后查看目录是否存在，直到到达字符串开头或一个存在的路径。**

字符串"a/b/c/d/e"，从最后开始向前递减，如果发现/a/b路径是存在的，则从此a/b/c开始创建。

**递归处理的流程是：**从字符串尾部开始，递减判断字符是否是'/'，找到'/'然后把它设置成'\0'，之后检测如果目录不存在则进行递归调用，递归调用结束，恢复之前'\0'为'/'，继续当前的目录创建过程，本例程使用了access检测目录是否存在：access(path, F_OK)，成功返回0，否则返回-1。

以下是完整的mkdir实现：

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>

#define ARGS_MKPARENT   0

#define ARGS_END        8

#define MK_ERR_NONE     0
#define MK_ERR_MALM     1
#define MK_ERR_FAIL     2
#define MK_ERR_ARGS     3

char _args[ARGS_END] = {'\0'};

int *_dir_ind = NULL;

void help(void)
{
    char *help_info[] = {
        "创建目录，支持参数--help，--mode\n",
        "--help：输出帮助信息\n",
        "--mode=[MODE]：设定创建目录的权限，MODE应该是一个三位的数字，",
        "否则程序会报错，数字是0-7的范围。比如，754表示rwxr-xr--。\n",
        "-p 如果父级目录不存在则创建。\n",
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

int try_make_parent(char *path, int mode);

int recur_make_parent(char *path, int mode);

void clean_exit(int err) {
    if (_dir_ind) {
        free(_dir_ind);
    }

    exit(err);
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
    int dir_count = 0;

    _dir_ind = (int*)malloc(sizeof(int)*(argc-1));
    if(_dir_ind == NULL) {
        perror("malloc");
        return MK_ERR_MALM;
    }

    for(int i=0; i<argc-1; i++)
        _dir_ind[i] = 0;

    for(i=1;i<argc;i++) {
        if (strcmp(argv[i],"--help")==0) {
            help();
            clean_exit(0);
        } else if (strncmp(argv[i],"--mode=",7)==0) {
            if (mode_flag > 0) {
                dprintf(2,"Error: too many --mode\n");
                clean_exit(MK_ERR_ARGS);
            }

            if (strlen(argv[i]+7)!=3) {
                dprintf(2, "Error: mode is wrong\n");
                clean_exit(MK_ERR_ARGS);
            }

            for (int k=0;k<3;k++) {
                tmp = argv[i][7+k];
                if (tmp < '0' || tmp > '7') {
                    dprintf(2, "Error: mode number must in [0,7]\n");
                    clean_exit(MK_ERR_ARGS);
                }
                mode_buf += (tmp-48)*(1<<(3*(2-k)));
            }

            mode_flag = i;
        } else if (strcmp(argv[i], "-p") == 0) {
            _args[ARGS_MKPARENT] = 1;
        } else {
            _dir_ind[i-1] = 1;
            dir_count ++;
        }
    }

    if (dir_count == 0) {
        dprintf(2,"Error: less DIR_NAME\n");
        clean_exit(MK_ERR_ARGS);
    }

    if (mode_flag>0 && mode_buf > 0)
        mode = mode_buf;

    for (i=1;i<argc;i++) {
        
        if (_dir_ind[i-1]==0)continue;

        if (mkdir(argv[i],mode) < 0) {
            if (_args[ARGS_MKPARENT]) {
                if (try_make_parent(argv[i], mode) < 0) {
                    clean_exit(MK_ERR_FAIL);
                }
            } else {
                perror("mkdir");
                return -1;
            }
        }
    }

    free(_dir_ind);
    _dir_ind = NULL;

    return 0;
}

/*
	创建父级目录，实际是调用recur_make_parent函数，此函数先把路径进行预处理，
	去掉目录最后的'/'，否则最后的'/'会影响目录的创建。
*/
int try_make_parent(char *path, int mode) {
    int plen = strlen(path);
    if (plen > 0 && path[plen-1]=='/')
        path[plen-1] = '\0';

    return recur_make_parent(path, mode);
}

int recur_make_parent(char *path, int mode) {
    
    int plen = strlen(path);
    if(plen<=0 || access(path, F_OK)==0)
        return 0;

    int i = plen-1;
    while(i>0 && path[i]!='/')i--;
    if (i == 0)
        goto start_mkdir; //如果到达字符串开头，则跳转到创建目录的操作

    //只有i大于0，说明是path[i] == '/'，把此处作为字符串结尾
    path[i] = '\0';

    if (access(path, F_OK) < 0)
        if(recur_make_parent(path, mode) < 0) //递归调用
            return -1;

    path[i] = '/'; //递归完成操作以后恢复数值

  start_mkdir:;
    if (mkdir(path, mode) < 0) {
        dprintf(2, "%s:\n", path);
        perror("mkdir");
        return -1;
    }

    return 0;
}
```



### 如何消除递归

函数递归调用实际上是系统在内存的栈空间不断调用函数，所以使用栈结构保存相应的数据就可以不使用递归。当然消除递归并非只能用栈结构，实现递归读取目录采用的是链表结构，我们在后面的开发过程中会进行实际的操作。



>泛型快速排序需要比较好的C编程功底，实现泛排序，可以对任何类型进行快速排序，但是排序驱动函数编写比较麻烦。

### *泛型快速排序

现在我们来考虑如何让快速排序支持任何数据类型？支持任何数据类型就是要支持泛型。C语言提供了void类型，void是无类型或者说空类型。这个关键字很少被人关注，但是要实现泛型操作就要用到void\*，void是无类型，那它就可以是任何类型。void*指向一块内存地址，怎么解释这块地址的数据，根据具体场景而定。

于是我们使用void\*指向任何类型的数据，在实际操作时再进行强制类型转换。这样在比较时，就要提供一个用于比较的回调函数，传递两个参数用a，b表示。返回值有三种：大于0、等于0、小于0，分别表示： a>b，a=b，a<b。



```c
#define SWAP(a,b)   tmp=a;a=b;b=tmp;

void qsort_core(void * base, int start, int end,
    unsigned int size, int (*comp)(const void *, const void *)
) {
    if (start >= end) {
        return ;
    }
    
    int med = (end+start)/2;
    int k = start;
    int j;
    char tmp;
    char * b = base;

    for (int i=0;i<size;i++) {
        SWAP(b[med*size+i],b[start*size+i]);
    }

    for(j=start+1;j<=end;j++) {
        if (comp(b+j*size,b+start*size) < 0) {
            k += 1;
            for(int i=0;i<size;i++) {
                SWAP(b[k*size+i],b[j*size+i]);
            }
        }
    }

    for (int i=0; i<size; i++) {
        SWAP(b[k*size+i],b[start*size+i]);
    }

    qsort_core(base, start, k-1, size, comp);
    qsort_core(base, k+1, end, size, comp);
}

void vqsort(void* base, unsigned int nmemb, unsigned int size, 
    int(*comp)( const void *, const void *)
) {
    qsort_core(base, 0, nmemb/size - 1, size, comp);
}

```



通过man 3 qsort查看文档发现：

```c
QSORT(3)                               Linux Programmer's Manual                               QSORT(3)

NAME
       qsort, qsort_r - sort an array

SYNOPSIS
       #include <stdlib.h>

       void qsort(void *base, size_t nmemb, size_t size,
                  int (*compar)(const void *, const void *));

       ......

   Feature Test Macro Requirements for glibc (see feature_test_macros(7)):

       qsort_r(): _GNU_SOURCE
       ......
```

系统提供的快速排序库函数和我们设计的非常相似，因为基本的逻辑是一致的。

这里有一个关键的问题是，用于回调的排序驱动函数如何编写。如果编写有误，会导致排序出错，尤其对指针的操作很容易出现段错误。

比如对于字符串的排序，实际交换的是字符串的指针地址。所以传递的是一个二级指针，要进行强制类型转换，对于链表等其他更复杂的结构，也是如此。

字符串比较的回调函数：

```c
int str_comp(void *a, void *b) {
    return strcmp(*(char**)a, *(char**)b);
}
```



---

> 以下内容是PHP方向相关的扩展内容，有兴趣的同学可以看看。

***

### #新浪面试题：PHP数组转换成JSON数据

此问题是2015级PHP方向的学生在新浪面试中被问到的问题。当时要求学生不使用PHP的JSON扩展，仅使用PHP的类型处理，文本处理等语言基础功能实现。实现此功能是一个递归的经典案例。

你可能看完PHP语法的基础介绍以及JSON的说明仍然无法理解，可以配合PHP参考手册了解更多语法，如果你对JSON以及使用有足够的了解，就能够很容易看明白。

在讲解此问题之前，简单提一下PHP语法：

* PHP的变量要使用$，定义变量：\$a = 123; \$b = 'abc';分别是数字和字符串。

* 数组：['a','b','c']或者['name'=>'bravewang','age'=>28]的形式，第一个通过arr[0]访问a，而第二个通过arr['name']的形式获取，第二种形式类似于Python的字典。PHP使用一种数组类型实现了所有复杂的结构。数组有更复杂的形式：

  	[ 123, 345, 'asd', 'a'=>'abc',  'b'=>[ 'qwe', 'we', 'ty' ] ]

* PHP中以  . 连接字符串，例：\$a . \$b，此时数字会转换成字符串。

* 字符串可使用' '或" ", 双引号中的变量和转义字符会解析，单引号不解析，就是原生的字符串。


JSON格式是一个独立的数据格式化标准，JSON格式经常用于API开发的数据格式化，配置文件格式等。JSON 使用 Javascript语法来描述数据对象，但是 JSON 仍然独立于语言和平台。JSON 解析器和 JSON 库支持许多不同的编程语言。JSON 语法是 JavaScript 语法的子集：

* 数据在名称-数值对中
* 名称要放在""中，字符串值也要使用""
* 数据由逗号分隔
* 大括号保存对象
* 中括号保存数组

JSON示例：

```json
{
    "name":"xy",
    "id":1023,
    "detail":{
        "a":"wert",
        "b":"yure"
    },
    "list":["Linux","Unix","Windows"]
}
```

JSON的数据值可以是JSON，数组，字符串，数字。

所以一个PHP数组:

```php
[
    'name' => 'BraveWang',
    'age' => 28,
    'job' => 'teacher & programmer',
    'skill' => ['Linux C', 'LNMP', 'MySQL', 'PHP', 'Python'],
    'detail' => [
        'address' => '河北石家庄',
        'work_address' => '河北师范大学'
    ]
]
```

转换成JSON格式就是以下形式：

```json
{
    "name":"BraveWang",
    "age":28,
    "job":"teacher & programmer",
    "skill":["Linux C", "LNMP", "MySQL", "PHP", "Python"],
    "detail":{
        "address":"河北石家庄",
        "work_address":"河北师范大学"
    }
}
```

由于JSON的数据还是可以是JSON，所以层层嵌套的形式，使用递归解决很容易。

以下代码是PHP实现方式，并使用json_encode函数输出进行对比，注意is_ind函数用于判断一个数组类型中是否存在索引值，举例来说，检测数组是[1,2,3]的形式还是['name'=>'ay', 'x','w']的形式。要做检测是因为第一种形式转换成数组类型的值，而第二种要转换成JSON类型的值。

```php
<?php
function is_ind($a) {
    if (is_array($a)) {
        $keys = array_keys($a);
        return ($keys!==array_keys($keys));
    }
    return false;
}

function json_num_str($v)
{ 
    /*
      转换" -> \"  \ -> \\ ，
      但是 ' 不能是 \' 要变成 ',object直接转换成{}
    */
    return (is_object($v)?'{},':
            (is_numeric($v)?($v . ','):
             ('"' . str_replace("\\'","'",addslashes($v))  .'",')
            )
           );
}

function arr_to_json($arr)
{
    $ii = is_ind($arr);
    $json = ($ii?'{':'[');
    if ($ii) {
        foreach($arr as $k=>$v) {
            if(is_array($v)) {
                $json .= '"'.$k.'":' . arr_to_json($v) . ',';
            } else {
                $json .= '"' . $k . '":' . json_num_str($v);
            }
        }
    }
    else{
        foreach($arr as $v) {
            $json .= (is_array($v)
                      ?(arr_to_json($v) . ',')
                      :json_num_str($v));
        }
    }
    return rtrim($json,',') . ($ii?'}':']');
}
//以上是实现Array -> JSON的代码。
/*
   以下是数据测试，输出结果和json_encode结果对比
*/
$a = [
    'a' => 'abc',
    'b' => 123,
    'c' => [
        1,2,3
    ]
];

$b = [1,2,3,4];

$c = [
    'name'=>'BraveWang',
    'age' => 28,
    'skill' => [
        'Linux','C','PHP','Python','Shell Script','MySQL','Nginx'
    ]
];

$d = [
    'sdf' => [
        '"sdf"sdf"','\'sdfewer\''
    ],
    'dch' => [
        ":sdf:\"",':,\\'
    ]
];

$aset = [$a,$b,$c,$d];

foreach ($aset as $ar) {
    echo arr_to_json($ar),"\n";
    echo json_encode($ar),"\n\n";
}
```

这个程序可以很好的工作，甚至在大多数API开发场景完全可以满足JSON转换的需求。

你可以尝试把Python的数据类型转换成JSON格式，仅仅使用Python最基本的文本处理，类型处理等操作。注意is_ind函数在Python中是不需要的，因为Python区分list和dict两种类型。所以对Python来说进行类型判断即可。

