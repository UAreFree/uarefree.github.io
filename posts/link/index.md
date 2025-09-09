# 链接


链接：将代码和数据收集并组合成一个可执行文件，加载到内存中执行。

学习链接可以解决的问题：
1. 构造大型程序，遇到由于缺少库文件或者库文件版本不兼容而导致的链接错误。
2. 避免一些难以发现的编程错误。
3. 理解编程语言中的作用域规则是如何实现的。
4. 理解重要的系统概念。
5. 更好的利用共享库。

<!--more--> 

## 编译器驱动程序 GCC
GNU 编译器集合 GCC 会调用语言预处理器、编译器、汇编器和链接器。`gcc -Og -o prog main.c sum.c`调用gcc编译，-Og启动适中优化、-o生成可执行文件。./prog调用操作系统中的加载器函数，将可执行文件prog中的代码和数据复制到内存，然后将控制转移到这个程序的开头。

GNU是一个自由软件项目，旨在开发完全自由的操作系统。GCC 是 GNU 项目下的一组编译工具的集合。
| 命令                              | 作用                           |
|-----------------------------------|--------------------------------|
| `gcc -Og -E prog main.c sum.c`    | 预处理生成 `.i`               |
| `gcc -Og -S prog main.c sum.c`    | 编译成汇编文件 `.s`           |
| `gcc -Og -c prog main.c sum.c`    | 汇编成可重定位目标文件 `.o`   |
| `gcc -Og -v prog main.c sum.c`    | 链接成可执行目标文件 `.out`   |

## 目标文件
注意不只是.o文件，目标文件有以下三种：
1. 可重定位目标文件：包含二进制代码和数据，.o文件可与其他.o文件链接
2. 可执行目标文件：包含二进制代码和数据，.out文件可直接复制到内存并执行
3. 共享目标文件：特殊的可重定位目标文件，可以在加载或者运行时被动态地加载进内存并链接
目标文件是按照特定的格式来组织的，各个系统的目标文件格式都不相同。现代 x86-64 Linux 和 Unix 系统使用可执行可链接格式（Executable and Linkable Format，ELF）。

## 可重定位目标文件
ELF 格式的可重定位目标文件由 ELF header、Sections、Section header table组成。
{{< image src="/link/ELF文件格式.png" caption="ELF文件格式" width="45%" height="45%">}}

### ELF header
{{< image src="/link/ELF头.png" caption="ELF头" width="45%" height="45%">}}
ELF header 的长度是64个字节。
### Sections
.text：已编译程序的机器代码。

.data：已初始化 的全局和静态变量。

.bss：未初始化的全局和静态变量，以及所有被 初始化为0 的全局或静态变量。在目标文件中这个节不占据实际的空间，它仅仅是一个占位符（better save space）。目标文件格式区分已初始化和未初始化变量是为了空间效率：在目标文件中，未初始化变量不需要占据任何实际的磁盘空间。运行 时，在内存中分配这些变量，初始值为 0。COMMON存放未初始化的全局变量。

.rodata:存放 只读 数据。如printf中的格式串和switch语句中的跳转表。

.symtab：符号表，存放在程序中定义和引用的函数和全局变量的信息。

### Section header table
存放 sections 中每个 section 的起始位置、大小、类型等信息。

## 符号和符号表
链接的本质是把多个不同的目标文件粘合在一起，而符号是链接中的粘合剂。符号表就是 GOT (Global Offset Table)。

使用`readelf -S file.o`查看段头信息（包含每个段的名称、大小、类型等），使用`readelf -s file.o`查看符号表（包括符号的名称、类型、值等），使用`readelf -r file.o`查看重定位信息（指示代码中哪些地方需要在链接时调整）。
```Bash
yaf@yaf-VMware-Virtual-Platform:~/projects/csapp$ readelf -s main.o

Symbol table '.symtab' contains 6 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS main.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 .text
     3: 0000000000000000    30 FUNC    GLOBAL DEFAULT    1 main
     4: 0000000000000000     8 OBJECT  GLOBAL DEFAULT    3 array
     5: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND sum
```

符号main是位于.text节（Ndx = 1）偏移量为 0 （value = 0）处的 30 字节函数；符号array是位于.data节（Ndx = 3）偏移量为 0 （value = 0）处的 8 字节对象；符号sum是对外部符号的引用（Ndx = UND）
```C
#include <stdio.h>

int count = 10; // 初始化的全局变量 .data
int value; // 未初始化的全局变量  .bss

void func(int sum){ // 全局函数
    printf("sum = %d\n", sum);
}

int main() {
    static int a = 1; // 初始化的静态变量 .data
    static int b = 0; // 初始化的静态变量 .bss
    int x = 1; // 局部变量
    func(a + b + x);
    return 0;
}
```
```Bash
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS main.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 .text
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 .data
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 .bss
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 .rodata
     6: 0000000000000004     4 OBJECT  LOCAL  DEFAULT    3 a.1
     7: 0000000000000004     4 OBJECT  LOCAL  DEFAULT    4 b.0
     8: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    3 count
     9: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    4 value
    10: 0000000000000000    43 FUNC    GLOBAL DEFAULT    1 func
    11: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND printf
    12: 000000000000002b    52 FUNC    GLOBAL DEFAULT    1 main
```
a.1和b.0是名称修饰，为了防止静态变量的名字冲突。局部变量符号表里是没有的，由于在栈上，符号表并不关心。

符号类型：
1. 全局符号，由该模块定义，同时能被其他模块引用。
2. 外部符号，被其他模块定义，同时被该模块引用。
3. 局部符号，只能被该模块定义和引用。
   - 区别局部符号和全局符号：static属性。

## 符号解析
将程序中的符号（如变量名、函数名等）与它们在内存或程序中的实际地址进行关联的过程。

### 符号同名冲突
1. 多个同名的强符号
   - 找死，出 error
2. 一个强符号和多个同名的弱符号
   - 没事，隐藏 bug
3. 多个同名的弱符号
   - warning，不易察觉的运行时 error
   - 编译时添加-fno -common的编译选项，多个重名触发错误；或-Werror把所有警告变成错误

强符号：函数和已初始化的全局变量。

弱符号：未初始化的全局变量。

### 静态链接
printf、atoi等函数存放在libc.a中，静态库以一种称为archive的特殊文件格式存放在磁盘上。使用`ar rcs libvector.a addvec.o multvec.o`构造静态库，使用`gcc -static -o prog main.o ./libvector.a`链接静态库。根据main.o中的符号，将所用到的符号对应的.o文件复制进可执行文件。
{{< image src="/link/静态链接过程.png" caption="静态链接过程" width="45%" height="45%">}}
目标文件是字节块的集合，有代码块、数据块、引导链接器和加载器的数据结构。

链接器必须完成的两个主要任务：
1. 符号解析：
  1. 按命令行从左到右扫描可重定位文件和静态库文件
     - 可重定位文件放到集合 E 中，最后合并起来
     - 引用了当尚未定义的符号放到集合 U 中
     - 文件中已定义的符号放到集合 D 中
使用静态链接，命令行文件的输入顺序十分重要，一般将库放到命令行结尾。而且如果库是独立的，必须对它们进行排序
2. 重定位

### 重定位
链接器合并输入模块，并为每个符号分配运行时地址。

1. 重定位节和符号定义
   - 把所有相同类型的 section 合并为一个新的 section
   - 每条指令和全局变量都有了唯一的运行时内存地址
2. 重定位节中的符号引用 
   - 修改对符号的引用使其指向正确的运行时地址
重定位条目：告诉链接器在合成可执行文件时应该如何修改这个引用，.rel.text是代码的重定位条目，.rel.data是已初始化数据的重定位条目
```C
typedef struct{
    long offset; // 引用的节偏移量
    long type:32, // 重定位类型 相对地址重定位、绝对地址重定位
        symbol:32; // 修改哪个符号
    long addend;
}ELF64_Rela;
```
最开始编译出来的.o文件 call 了外部函数是不知道具体地址的，需要填后32位为“S+A-P”，使得满足约束断言：（转到调用函数的实际地址）
```C
assert(hello == main + 0xf + // call hello 的 next pc
                + (main + 0xb) // call 指令的 offset
);
```
## 可执行文件
{{< image src="/link/可执行文件段分布.png" caption="可执行文件段分布" width="45%" height="45%">}}
.init节定义了_init函数，程序初始化代码调用_init函数进行初始化。程序执行时代码段和数据段加载到内存执行，其他不被加载到内存。

程序头部表描述了代码段、数据段与内存的映射关系。.bss中的数据不占用可执行文件的空间，但在运行时需要被初始化为0，所以加载到内存需要增加空间。
{{< image src="/link/段描述.png" caption="段描述" width="45%" height="45%">}}
./prog运行可执行文件的过程（加载）：
1. 调用execve()调用加载器
2. 加载器将可执行文件中的代码和数据从磁盘复制到内存
3. 跳转到程序入口运行该程序
   - ctrl.o系统目标文件中定义了程序入口函数_start()
   - _start()调用libc.so中的_libc_start_main()初始化执行环境
   - 调用用户层中可执行程序prog中的main()，函数返回值返回给_libc_start_main()，且在需要时把控制权返回给操作系统Kernel
每一个 Linux 程序都有一个运行时内存镜像，如下图所示。
{{< image src="/link/运行时程序内存分布.png" caption="运行时程序内存分布" width="45%" height="45%">}}
1. 数据段有地址对齐的要求，所以数据段和代码段之间是有间隙的
2. 为防止程序受到攻击，在分配栈、共享库以及堆的运行时地址时，链接器会使用地址空间随机化的策略，所以每次运行时地址都会变，但相对位置不变

## 动态链接
静态链接库的缺点：
1. 需要定期维护和更新
2. 几乎每个 C 程序都需要使用标准的 I/O 函数，每个进程都需要把其所用到的静态库复制到内存中，对于运行成百上千个进程的系统，这是对内存资源的极大浪费

共享库是一种特殊的可重定位目标文件，在 Linux 系统中用.so表示，windows 系统中用 .dll表示。共享库在运行或加载时可以被加载到任意的内存地址，还能和一个在内存中的程序链接起来，这个过程称为动态链接，具体是由动态链接器来执行的。

使用`gcc -shared -fpic -o libvector.so addvec.c multvec.c`生成动态链接库，-shared创建共享的目标文件，-fpic生成位置无关的代码。

链接器并没有把libvector.so中的代码和数据复制到可执行文件中，而是只复制了符号表和一些重定位信息。在可执行程序运行时，加载器会发现可执行程序中存在一个名为.interp的section，这个section中包含了动态链接器的路径名，加载器将动态链接器加载到内存中运行，由其执行重定位代码和数据的工作。

动态链接器：本质上也是一个共享目标文件（ld-linux.so）

### 动态链接的过程
1. 将libc.so的代码和数据重定位到某个内存段
2. 重定位libvector.so的代码和数据到另一个内存段
3. 重定位 proc2 中由libc.so和libvector.so定义的符号引用
   - 应用程序a.out里会有一个表PLT存放符号的地址，从而指向符号代码
   - 懒加载，用到了的符号才回去查表间接跳转

以上是编译、加载的过程，应用程序还可能在运行时要求动态链接器加载和链接某个共享库。

实现位置无关代码 PIC ：
PLT（过程链接表）：每个外部函数的调用都通过 PLT 表进行跳转，PLT 条目中存储了跳转的目标地址（初始时指向 GOT 表），并通过延迟绑定在运行时解析。

GOT（全局偏移量表）：GOT 存储了指向实际函数或数据的地址，每个函数调用之前，GOT 会被动态链接器填充为正确的地址。

在运行时，动态链接器负责将程序中的外部符号链接到实际的共享库函数地址。位置无关代码通过延迟绑定（lazy binding）机制优化了性能。只有当程序第一次调用某个共享库函数时，动态链接器才会解析该函数的地址，并更新 GOT 表。之后的调用不需要再进行重定位，直接通过 PLT 表进行跳转。

### 应用
1. 分发软件
   - Windows 应用开发者利用共享库进行软件版本的更新
2. 构建高性能 web 服务器
   - 生成个性化的 web 页面及账户余额
   - 将每个生成动态内容的函数打包在共享库中，服务器根据请求动态加载和链接适当的函数，然后直接调用，函数会一直缓存在服务器的地址空间中
   - 在更新函数时并不需要停止正在运行的服务器
  
### 实现运行时加载和链接共享库
Linux 提供了dlopen()接口允许应用程序在运行时加载和链接共享库。
```C
void* dlopen(const char* filename, int flag);
handle = dlopen("./libvector.so", RTLD_LAZY); // 推迟到共享库中代码执行时再进行符号解析
void* dlsym(void* handle, char* symbol);
addvec = dlsym(handle, "addvec"); // handle句柄 符号addvec 返回符号地址
addvec(x, y, z, 2);
int dlclose(void* handle);
dlclose(handle); // 卸载共享库
```
参考：
1. {{<link href="https://www.bilibili.com/video/BV1yq4y1t7is/?spm_id_from=333.1387.collection.video_card.click&vd_source=050bf08a3643c244550421a1ccc94f6b" content="CSAPP讲解课程——九曲阑干">}}
2. {{<link href="https://www.bilibili.com/video/BV15F411G7Ti?spm_id_from=333.788.videopod.sections&vd_source=050bf08a3643c244550421a1ccc94f6b" content="南大操作系统课程——jyy">}}


---

> 作者: [UAreFree](https://github.com/UAreFree)  
> URL: http://localhost:1313/posts/link/  

