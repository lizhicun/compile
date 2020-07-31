# 编译优化（上）

### 内容摘要
* 预处理
* 编译和链接过程
* 静态链接和动态链接
* CMake和Make
* 编译耗时分析
* 编译优化手段
* 涉及到的概念一览

### 预处理
可以粗略的认为只做了一件事情，那就是“宏展开”，也就是对那些#。。。的命令的一种展开，看例子。
```
#include <stdio.h>
#define MAX 10

int main() {
	int a = MAX;
#ifdef HAHA
	a++;
#endif
	printf("a = %d \n");
	return 0;
}
```
执行 ```gcc test3.c -E -o test3.i``` 查看test3.i

### 编译和链接过程
#### 单个cpp文件编译过程
  以下面的文件为例。
```
// test1.c

int g_a = 1;            //定义有初始值全局变量
int g_b;                //定义无初始值全局变量
static int g_c;         //定义全局static变量
extern int g_x;         //声明全局变量
extern int sub();   //函数声明

int sum(int m, int n) {     //函数定义
    return m+n;
}
int main(int argc, char* argv[]) {
    static int s_a = 0;     //局部static变量
    int l_a = 0;                //局部非static变量
    sum(g_a,g_b);
    return 0;
}
```
  其中，static，非static变量与全局，非全局的四种组合的效果如下
* 非 static 全局变量表示这个变量存在于程序执行的整个生命周期，同时可以被本源码文件之外的其他文件访问到。
* static 全局变量表示这个变量存在于程序执行的整个生命周期，但是只能被本源码文件的函数访问到。
* 非 static 局部变量表示，这个变量只在本变量所在的函数的执行上下文中存在（实际上这种变量是在函数执行栈的函数栈帧中）
* static 局部变量其实属于全部变量的范畴，它存在于程序执行的整个生命周期，但是作用域被局限在定义这个变量的代码块中（大括号包含的范围）
 
执行```gcc -c test1.c -o test1.o && nm test1.o```编译成目标文件,得到下面的结果 
 ```
0000000000000000 D g_a
0000000000000004 C g_b
0000000000000000 b g_c
                 U g_x // fake
0000000000000014 T main
0000000000000004 b s_a.1843
0000000000000000 T sum
  ```
  * 第一列是变量在所在段的相对地址，我们看到 g_a 和 g_c 的相对地址是相同的，这并不冲突，因为他们处于不同的段中（ D 和 b 表示它们在目标文件中处于不同的段中).  
  * 第二列表示变量所处的段的类型，比如 D 段就是数据段，专门存放有初始值的全局变量，T 段表示代码段，所有的代码编译后的指令都放到这个段中。同一个段中的变量相对地址是不能重复的。  
  *　第三列表示变量的名字，这里我们看到局部的静态变量名字被编译器修改为 s_a.1597。s_a 是一个局部静态变量，作用域限制在定义它的代码块中，所以我们可以在不同的作用域中声明相同名字的局部静态变量，比如我们可以在sum函数中声明另外一个 s_a。但是我们上面提过，局部静态变量属于全局变量的范畴，它是存在于程序运行的整个生命周期的，所以为了支持这个功能，编译器对这种局部的静态变量名字加了一个后缀以便标识不同的局部静态变量。  
  * g_x 没有地址是因为变量和函数的声明本质上是给编译器一个承诺，告诉编译器虽然在本文件中没有这个变量或者函数定义，但是在其他文件中一定有，所以当编译器发现程序需要读取这个变量对应的数据，但是在源文件中找不到的时候，就会把这个变量放在一个特殊的段（段类型为 U）里面，表示后续链接的时候需要在后面的目标文件或者链接库中找到这个变量，然后链接成为可执行二进制文件。 
  
  
#### 多个cpp文件编译链接过程

下面是test2.c文件, 执行```gcc test.o test2.o -o test; nm test```

```
// test2.c

int g_x = 100;
int sub() {}
```
执行后链接test1.o和test2.o，即```gcc test1.o test2.o -o test; nm test```得到下面的内容
```
0000000000601038 B __bss_start
0000000000601038 b completed.7594
0000000000601020 D __data_start
0000000000601020 W data_start
0000000000400410 t deregister_tm_clones
0000000000400490 t __do_global_dtors_aux
0000000000600e18 t __do_global_dtors_aux_fini_array_entry
0000000000601028 D __dso_handle
0000000000600e28 d _DYNAMIC
0000000000601038 D _edata
0000000000601048 B _end
00000000004005a4 T _fini
00000000004004b0 t frame_dummy
0000000000600e10 t __frame_dummy_init_array_entry
0000000000400728 r __FRAME_END__
0000000000601030 D g_a
0000000000601044 B g_b
000000000060103c b g_c
0000000000601000 d _GLOBAL_OFFSET_TABLE_
                 w __gmon_start__
00000000004005b4 r __GNU_EH_FRAME_HDR
0000000000601034 D g_x
0000000000400390 T _init
0000000000600e18 t __init_array_end
0000000000600e10 t __init_array_start
00000000004005b0 R _IO_stdin_used
                 w _ITM_deregisterTMCloneTable
                 w _ITM_registerTMCloneTable
0000000000600e20 d __JCR_END__
0000000000600e20 d __JCR_LIST__
                 w _Jv_RegisterClasses
00000000004005a0 T __libc_csu_fini
0000000000400530 T __libc_csu_init
                 U __libc_start_main@@GLIBC_2.2.5
00000000004004ea T main
0000000000400450 t register_tm_clones
0000000000601040 b s_a.1843
00000000004003e0 T _start
000000000040051c T sub
00000000004004d6 T sum
0000000000601038 D __TMC_END__
```
  之前在第一个源文件中声明的变量 g_x 和声明的函数 sub 最终在第二个目标文件中找到了定义；其次，在不同目标文件中定义的变量，比如 g_a, g_x 都会放在了数据段中(段类型为 D)；还有，之前在目标文件中变量的相对地址全部变成了绝对地址。  
  
  编译是以一个个独立的文件作为单元的，一个文件就会编译出一个目标文件。因此编译只负责本单元的那些事，而对外部的事情一概不理会，在这一步里，我们可以调用一个函数而不必给出这个函数的定义，但是要在调用前得到这个函数的声明。  
  
  链接过程分为两步，第一步是合并所有目标文件的段，并调整段偏移和段长度，合并符号表，分配内存地址；第二步是链接的核心，进行符号的重定位。
  
  * 合并段  
       所有相同属性的段进行合并，组织在一个页面上，这样更节省空间。如.text段的权限是可读可执行，.rodata段也是可读可执行，所以将两者合并组织在一个页面上；同理合并.data段和.bss段。  
  * 合并符号表  
       链接阶段只处理所有obj文件的global符号，local符号不作任何处理。     
  * 符号解析  
       符号解析指的是所有引用符号的地方都要找到符号定义的地方。     
  * 分配内存地址  
       在编译过程中不分配地址（给的是零地址和偏移），直到符号解析完成以后才分配地址。  
  * 符号重定位  
       因为在编译过程中不分配地址，所以在目标文件所以数据出现的地方都给的是零地址，所有函数调用的地方给的是相对于下一条指令的地址的偏移量。在符号重定位时，要把分配的地址回填到数据和函数调用出现   的地方，而且对于数据而言填的是绝对地址，而对函数调用而言填的是偏移量。  
       
#### 头文件
  头文件的作用就是被其他的.cpp包含进去的。它们本身并不参与编译，但它们的内容却在多个.cpp文件中得到了编译。如果在头文件中放了定义，头文件被多个cpp文件包含后，那么也就相当于在多个文件中出现了对于一个符号（变量或函数）的定义，纵然这些定义都是相同的，但对于编译器来说，这样做不合法。但是有以下几个例外。  
  ```
  multiple definition 见 test4
  ```
  * 头文件中可以写const对象的定义。因为全局的const对象默认是没有extern的声明的，所以它只在当前文件中有效。把这样的对象写进头文件中，即使它被包含到其他多个.cpp文件中，这个对象也都只在包含它的 那个文件中有效，对其他文件来说是不可见的，所以便不会导致多重定义。同理，static对象的定义也可以放进头文件。  
  * 头文件中可以写内联函数（inline）的定义。因为inline函数是需要编译器在遇到它的地方把它展开，而并非是普通函数那样可以先声明再链接的（内联函数不会链接），所以编译器就需要在编译时看到内联函数的完整定义才行。如果内联函数像普通函数一样只能定义一次的话，这事儿就难办了。因为在 一个文件中还好，我可以把内联函数的定义写在最开始，这样可以保证后面使用的时候都可以见到定义；但是，如果我在其他的文件中还使用到了这个函数那怎么办 呢？这几乎没什么太好的解决办法，因此C++规定，内联函数可以在程序中定义多次，只要内联函数在一个.cpp文件中只出现一次，并且在所有的.cpp文 件中，这个内联函数的定义是一样的，就能通过编译。那么显然，把内联函数的定义放进一个头文件中是非常明智的做法。  
  * 头文件中可以写类（class）的定义。因为在程序中创建一个类的对象时，编译器只有在这个类的定义完全可见的情况下，才能知道这个类的对象应该如何布局。一般的做法是，把类的定义放在头文件中，而把函数成员的实现代码放在一个.cpp文件中。还有另一种办法就是直接把函数成员的实现代码也写进类定义里面。在C++的类中，如果函数成员在类的定义体中被定义，那么编译器会视这个函数为内联的。因此，把函数成员的定义写进类定义体，一起放进头文件中，是合法的。注意一下，如果把函数成员的定义写在类定义的头文件中，而没有写进类定义中， 这是不合法的，因为这个函数成员此时就不是内联的了。一旦头文件被两个或两个以上的.cpp文件包含，这个函数成员就被重定义了。  
  

### 静态链接和动态链接

```
#include <stdio.h>
#include <string.h>

int main(int argc, char* argv[]) {
    char buf[32];
    strncpy(buf, "Hello, World\n", 32);
    printf("%s",buf);
}
```
```
gcc test5.c -o test5 && nm test5  
ldd test5 
``` 
  静态库是链接器通过静态链接将其和其它目标文件合并生成可执行文件的，如下图一所示，而静态库只不过是将多个目标文件进行了打包，在链接时只取静态库中所用到的目标文件，因此，你可以将静态链接想象成如下图2所示的过程。  
 ![Image text](https://github.com/lizhicun/compile/blob/master/src/pic1.png)
 
 ![Image text](https://github.com/lizhicun/compile/blob/master/src/pic2.png)
 
  静态链接下可执行文件的生成  
  
 ![Image text](https://github.com/lizhicun/compile/blob/master/src/pic3.png)
 
  使用静态库时，静态库的代码段和数据段都会直接打包copy到可执行文件当中，使用静态库无疑会增大可执行文件的大小，同时如果程序都需要某种类型的静态库，比如libc，使用静态链接的话，每个可执行文件当中都会有一份同样的libc代码和数据的拷贝，如图所示  
  
  ![Image text](https://github.com/lizhicun/compile/blob/master/src/pic4.png)
  
  动态库允许使用该库的可执行文件仅仅包含对动态库的引用而无需将该库拷贝到可执行文件当中。也就是说，同静态库进行整体拷贝的方式不同，对于动态库的使用仅仅需要可执行文件当中包含必要的信息 
  
  ![Image text](https://github.com/lizhicun/compile/blob/master/src/pic5.png)
  
  静态库其实是在编译期间(Compile time)链接使用的，动态链接可以在两种情况下被链接使用，分别是load-time dynamic linking(加载时动态链接) 以及 run-time dynamic linking(运行时动态链接) 
  
  1.load-time dynamic linking(加载时动态链接)
  这里的加载指的是程序的加载，而所谓程序的加载就是把可执行文件从磁盘搬到内存的过程，因为程序最终都是在内存中被执行的。当把可执行文件复制到内存后，且在程序开始运行之前，操作系统会查找可执行文件依赖的动态库信息(主要是动态库的名字以及存放路径)，找到该动态库后就将该动态库从磁盘搬到内存，并进行符号决议，如果这个过程没有问题，那么一切准备工作就绪，程序就可以开始执行  
  
  2.run-time dynamic linking(运行时动态链接)
  运行时动态链接这种方式对于“动态链接”阐释的更加淋漓尽致，因为可执行文件在启动运行之前都不知道需要依赖哪些动态库，只在运行时根据代码的需要再进行动态链接。同加载时动态链接相比，运行时动态链接将链接这个过程再次推迟往后推迟，推迟到了程序运行时。  

  动态链接下可执行文件的生成  
  在动态链接下，链接器并不是将动态库中的代码和数据拷贝到可执行文件中，而是将动态库的必要信息写入了可执行文件，这样当可执行文件在加载时就可以根据此信息进行动态链接了。如图所示，在动态链接下，可执行文件当中会新增两段，即dynamic段以及GOT（Global offset table）段，这两段内容就是是我们之前所说的必要信息。  
  
  ![Image text](https://github.com/lizhicun/compile/blob/master/src/pic6.png)

  动态库VS静态库  
  在计算机的历史当中，最开始程序只能静态链接，但是人们很快发现，静态链接生成的可执行文件存在磁盘空间浪费问题，因为对于每个程序都需要依赖的libc库，在静态链接下每个可执行文件当中都有一份libc代码和数据的拷贝，为解决该问题才提出动态库。  
  
 * 动态链接下可执行文件当中仅仅保留动态库的必要信息，因此解决了静态链接下磁盘浪费问题  
 * 修改了动态库的代码，我们只需要重新编译动态库就可以了而无需重新新编译我们自己的程序，因为可执行文件当中仅仅保留了动态库的必要信息，重新编译动态库后这些必要都信息是不会改变的（只要不修改动态库的名字和动态库导出的供可执行文件使用的函数）  
 * 动态链接可以出现在运行时（run-time dynamic link），动态链接的这种特性可以用于扩展程序能力。比如插件。我们知道使用运行时动态链接无需在编译链接期间告诉链接器所使用的动态库信息，可执行文件对此一无所知，只有当运行时才知道使用什么动态库，以及使用了动态库中哪些函数，但是在编译链接可执行文件时又怎么知道插件中定义了哪些函数呢，因此所有的插件实现函数必须都有一个统一的格式，程序在运行时需要加载所有插件（动态库），然后调用所有插件的入口函数（统一的格式），这样我们写的插件就可以被执行起来了。  
 * 使用动态链接的程序在性能上要稍弱于静态链接，这时因为对于加载时动态链接，这无疑会减慢程序都启动速度，而对于运行时链接，当首次调用到动态库的函数时，程序会被暂停，当链接过程结束后才可以继续进行。  
 * 静态链接都最大优点就是使用简单，编译好的可执行文件是完备的，即静态链接下的可执行文件不需要依赖任何其它的库，因为静态链接下，链接器将所有依赖的代码和数据都写入到了最终的可执行文件当中，这就消除了动态链接下的库依赖问题。  

### 编译耗时分析
编译器，是把一种语言（通常是高级语言）转换为另一种语言（通常是低级语言）的程序。  

大多数编译器由三部分组成：   
![Image text](https://github.com/lizhicun/compile/blob/master/src/pic7.png)

前端（Frontend）：负责解析源码，检查错误，生成抽象语法树（AST），并把 AST 转化成类汇编中间代码；  
优化器（Optimizer）：对中间代码进行架构无关的优化，提高运行效率，减少代码体积，例如删除 if (0) 无效分支；  
后端（Backend）：把中间代码转换成目标平台的机器码。  

Clang/LLVM 编译器是开源的，我们可以从官网下载其源码，根据上述编译过程，在每个编译阶段埋点输出耗时，生成定制化的编译器。clang 增加 -ftime-trace 选项，编译时生成 Chrome（chrome://tracing） JSON 格式的耗时报告，列出所有阶段的耗时。  

![Image text](https://github.com/lizhicun/compile/blob/master/src/pic8.png)

1）编译器前端处理（Frontend）耗时 7,659.2s，占整体 87%；  
2）而前端处理下头文件处理（Source）耗时 7,146.2s，占整体 71.9%！  
猜测：头文件嵌套严重，每个源文件都要引入几十个甚至几百个头文件，每个头文件源码要做预处理、词法分析、语法分析等等。实际上源文件不需要使用某些头文件里的定义（如 class、function），所以编译时间才那么长。  

下图统计所有头文件被引用次数、总处理时间、头文件分组（指一个耗时顶部的头文件所引用到的所有子头文件的集合）。  
![Image text](https://github.com/lizhicun/compile/blob/master/src/pic9.png)

Header1 处理时间 1187.7s，被引用 2,304 次；  
Header2 处理时间 1,124.9s，被引用 3,831 次；  
后面 Header3～10 都是被 Header1 引用。  

该案例最终的优化总结  
* 优化头文件搜索路径；  
* 关闭 Enable Index-While-Building Functionality；  
* 优化 PB/模版，减少冗余代码；  
* 使用 PCH 预编译；  
* 使用工具优化头文件引入；尽量避免头文件里包含 C++ 标准库。  

### 编译优化手段
#### 使用 PCH 预编译头文件  
* 解决什么问题？
　　C++ 编译器是单独，分别编译的，每个cpp文件，进行预编译（也就是对#include，define 等进行文本替换），生成编译单元。编译单元是一个自包含文件，C++编译器对编译单元进行编译。考虑，头文件A.h被多个cpp文件（比如A1.cpp，A2.cpp）包含，每个cpp文件都要进行单独编译，其中的A.h部分就会被多次重复第编译，影响效率。  
* 怎么解决？
   把A.h以及类似A.h这样的头文件，包含到stdafx.h中（当然也可以是其他文件），在stdafx.cpp中包含stdafx.h，设置stdafx.cpp文件的属性，预编译头设置为 创建。对于原先包含A.h的cpp文件，删除#include "A.h"，改成包含stdafx.h，同时设置这些cpp文件（A1.cpp，A2.cpp）的属性，预编译头设置为 使用。这样的话，下次编译A1.cpp，A2.cpp的时候，对于A.h头文件中的那部分，就不需要编译了，节省时间.
* 预编译头文件原理
　　工程对预先编译的代码进行编译，会生成一个pch文件（precompiled header），包含了编译的结果。注意，可以对任何代码生成到pch中，但是生成pch是个很耗时的操作，因此，只对那些稳定的代码创建预编译头文件。
* 缺点
1、程序间的耦合性增加，不需要的头文件也被引用了进来。这是由于预编译头文件通常集成了多个不同的头文件——不管你的模块用不用得到。  
2、增加了程序逻辑和编译特性的依赖关系。首先，头文件通常是模块的外部接口，对于头文件的包含，应该反映的是程序逻辑依赖关系，而不是编译关系。其次，由于VC下的预编译头文件命名方式通常是以“pch”结尾，进一步违反了模块设计的“单一职责”原则。这样的头文件反映的更多的是编译优化依赖关系，而不是程序逻辑依赖关系。

#### 前置声明
来不及了,直接看 https://www.jianshu.com/p/de396c0751cc  
效果应该是减少了头文件的引用  

#### PIMPL
```
//x.h
class X
{
public:
    void Fun();
private:
    int i; //add int i;
};

//c.h
#include <x.h>
class C
{
public:
    void Fun();
private:
    X x; //与X的强耦合
};

PIMPL做法：
//c.h
class X; //代替#include <x.h>
class C
{
public:
    void Fun();
private:
    X *pImpl; //pimpl
};
```
使用Pimpl的优点
* 降低模块的耦合。因为隐藏了类的实现，被隐藏的类相当于原类不可见，对隐藏的类进行修改，不需要重新编译原类。
* 降低编译依赖，提高编译速度。指针的大小为（32位）或8（64位），X发生变化，指针大小却不会改变，文件c.h也不需要重编译。
* 接口与实现分离，提高接口的稳定性。
```
//c.cpp
C::C()pImpl(new X())
{
}

C::~C()
{
     delete pImpl;
     pImpl = NULL;
}

void C::Fun()
{
    pImpl->Fun();
}

//main
#include <c.h>
int main()
{
    C c1;
    c1.Fun();
    return 0;
}
```
### 涉及到的概念一览
* 符号  
* 段
* 目标文件，可执行文件
