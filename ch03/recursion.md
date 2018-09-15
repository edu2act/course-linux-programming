## 递归

----

递归在软件编程中被广泛使用，有时候采用递归会使程序编写简化并能完美的体现设计思想或处理流程。需要注意的是，某些场景虽然可以使用递归解决，但并不合适。

### 滥用递归的情况

通常的示例总会去求解斐波那契数列、1~n的和、阶乘...，尽管可以解决问题，但是这种示例其实并不好，属于对递归的滥用。这几个示例的解法简单，不再以代码演示。

一个简单的原则是：如果递归能清晰描述运行原理，并且系统资源占用在合理范围内，则使用递归。‘合理范围’是一个不明确的说法，不能严格定义，随着开发水平提高，开发经验积累，你将能够根据系统需求，业务场景，硬件特性等因素确定解决方案。

### 递归经典示例：快速排序

这里给出的示例是实现整数快速排序的递归实现，并且采用C语言实现。你在接触这部分内容时应该已经学过快速排序算法。我们会在这一部分实现快速排序并在后面实现ls命令的时候使用。快速排序的思想简洁优雅，但实现起来，还是有一些需要仔细考虑的地方。一个是基准元的选取，另一个是交换大小数据的方式。以下代码是一个快速排序的实现，实现方式在选区基准元的时候采用了中间位置取值，并把此值和start位置数据交换。之后从start+1开始和d[start]比较，并使用i记录分组位置。代码在交换数据的时候没有使用函数，而是使用宏定义展开。

```c
#define SWAP(a,b)   tmp=a;a=b;b=tmp;

void qsorti(int *d, int start, int end) {
    if (start >= end) {
        return ;
    }
    int med = (end+start)/2;
    int i = start;
    int j = end;
    int tmp = 0;
	
    SWAP(d[med],d[start]);
    for(j=start+1;j<=end;j++) {
        if (d[j]<d[start]) {
            i++;
            if (i==j)continue;
            SWAP(d[i],d[j]);
        }
    }
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
            if (i==j)continue;
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





### 如何消除递归

函数递归调用实际上是系统在内存的栈空间不断调用函数，所以使用栈结构保存相应的数据就可以不使用递归。当然消除递归并非只能用栈结构，实现递归读取目录采用的是链表结构，我们在后面的开发过程中会进行实际的操作。





>泛型快速排序需要比较好的C编程功底，有精力有兴趣的同学可以看看。

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
            if (k==j)continue;
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

比如对于字符串的排序，实际交换的是字符串的指针地址。字符串比较的回调函数：

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

