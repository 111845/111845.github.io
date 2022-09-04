# 一、Linux GCC常用命令

### 1 新建一个文件test，代码如下：

#include <stdio.h>
 int main(void) 
 { printf("Hello World!\n"); return 0; }

直接编译： gcc test.c -o test

​    实际上编译过程有**四个阶段**，即预处理(也称预编译，Preprocessing)、编译 (Compilation)、汇编 (Assembly)和连接(Linking)。

### 2 编译过程

#### 2.1 预处理

gcc -E test.c -o test.i 或 gcc -E test.c

gcc 的-E 选项，可以让编译器在预处理后停止，并输出预处理结果。

#### 2.2 编译为汇编代码(Compilation)

gcc -S test.i -o test.s

gcc 的-S 选项，表示在程序编译期间，在生成汇编代码后，停止，-o 输出汇编代码文件。

#### 2.3 汇编(Assembly)

gcc -c test.s -o test.o

gas 汇编器负责将test.s编译为目标文件。

#### 2.4 连接(Linking)

gcc test.o -o test

gcc 连接器将程序的目标文件与所需的所有附加的目标文件（静态连接库和动态连接库）连接起来，最终生成可执行文件。

#### 2.5 执行命令

./test

原文链接：https://blog.csdn.net/qq_44644740/article/details/109086520





# 二、Linux中的保护机制

- NX：-z execstack / -z noexecstack (关闭 / 开启)
- Canary：-fno-stack-protector /-fstack-protector / -fstack-protector-all (关闭 / 开启 / 全开启)
- PIE：-no-pie / -pie (关闭 / 开启)
- RELRO：-z norelro / -z lazy / -z now (关闭 / 部分开启 / 完全开启)

  		在编写漏洞利用代码的时候，需要特别注意目标进程是否开启了NX、PIE等机制，例如存在NX的话就不能直接执行栈上的数据，存在PIE 的话各个系统调用的地址就是随机化的。

## 一：canary（栈保护）

​		栈溢出保护是一种缓冲区溢出攻击缓解手段，当函数存在缓冲区溢出攻击漏洞时，攻击者可以覆盖栈上的返回地址来让shellcode能够得到执行。当启用栈保护后，函数开始执行的时候会先往栈里插入cookie信息，当函数真正返回的时候会验证cookie信息是否合法，如果不合法就停止程序运行。攻击者在覆盖返回地址的时候往往也会将cookie信息给覆盖掉，导致栈保护检查失败而阻止shellcode的执行。在Linux中我们将cookie信息称为canary。

gcc在4.2版本中添加了-fstack-protector和-fstack-protector-all编译参数以支持栈保护功能，

因此在编译时可以控制是否开启栈保护以及程度，例如：

1、gcc -o test test.c // 默认情况下，不开启Canary保护

2、gcc **-fno-stack-protector** -o test test.c //禁用栈保护

3、gcc **-fstack-protector** -o test test.c //启用堆栈保护，不过只为局部变量中含有 char 数组的函数插入保护代码

4、gcc **-fstack-protector-all** -o test test.c //启用堆栈保护，为所有函数插入保护代码

## 二：NX（no execute）

​		NX即No-eXecute（不可执行）的意思，NX（DEP）的基本原理是将数据所在内存页标识为不可执行，当程序溢出成功转入shellcode时，程序会尝试在数据页面上执行指令，此时CPU就会抛出异常，而不是去执行恶意指令。

​		gcc编译器默认开启了NX选项，如果需要关闭NX选项，可以给gcc编译器添加-z execstack参数。 例如：

1、gcc -o test test.c // 默认情况下，开启NX保护

2、gcc -z execstack -o test test.c // 禁用NX保护

3、gcc -z noexecstack -o test test.c // 开启NX保护

在Windows下，类似的概念为DEP（数据执行保护）

## **三：PIE（position-independent executables）**

　　**位置独立的可执行区域**。这样使得在利用缓冲溢出和移动操作系统中存在的其他内存崩溃缺陷时采用面向返回的编程（return-oriented programming）方法变得难得多。一般情况下NX（Windows平台上称其为DEP）和地址空间分布随机化（ASLR）会同时工作。内存地址随机化机制（address space layout randomization)，有以下三种情况：

0 - 表示关闭进程地址空间随机化。

1 - 表示将mmap的基址，stack和vdso页面随机化。

2 - 表示在1的基础上增加栈（heap）的随机化。

 

liunx下关闭PIE的命令如下：

sudo -s echo 0 > /proc/sys/kernel/randomize_va_space

gcc编译命令：

1、gcc -o test test.c // 默认情况下，不开启PIE

2、gcc -fpie -pie -o test test.c // 开启PIE，此时强度为1

3、gcc -fPIE -pie -o test test.c // 开启PIE，此时为最高强度2

4、gcc -fpic -o test test.c // 开启PIC，此时强度为1，不会开启PIE

5、gcc -fPIC -o test test.c // 开启PIC，此时为最高强度2，不会开启PIE

## **四：RELRO（ read only relocation）**

　　在Linux系统安全领域，数据可以写的存储区就会是攻击的目标，尤其是存储函数指针的区域。 所以在安全防护的角度来说尽量减少可写的存储区域对安全会有极大的好处。GCC, GNU linker以及Glibc-dynamic linker一起配合实现了一种叫做relro的技术:。大概实现就是由linker指定binary的一块经过dynamic linker处理过 relocation之后的区域为只读.设置符号重定向表格为只读或在程序启动时就解析并绑定所有动态符号，从而减少对GOT（Global Offset Table）攻击。RELRO为” Partial RELRO”，说明我们对GOT表具有写权限。

gcc编译：

gcc -o test test.c // 默认情况下，是Partial RELRO

gcc -z norelro -o test test.c // 关闭，即No RELRO

gcc -z lazy -o test test.c // 部分开启，即Partial RELRO

gcc -z now -o test test.c // 全部开启

原文链接：https://www.cnblogs.com/ncu-flyingfox/p/11223390.html

# 三、linux  （kylin） 下编写shellcode（未完成）！！！！！！！！！！

## 1.环境：

gcc 

ld 

 nasm 

(1) addr2line：帮助调试器在调试的过程中定位对应的源代码位置。
(2) as：用于汇编；
(3) ld：用于链接；
(4) ar：用于创建静态库。

```
第一步：先判断系统是否已经安装了nasm--------------->打开终端，执行whereis nasm ；如果显示nasm: /usr/bin/nasm ，则已经安装；如果只显示nasm： ，则未安装。

第二布：去官网下载最新版本的源码编译http://www.nasm.us/，如nasm-X.XX. ta .gz，X.XX.是版本号。

第三步开始安装，

首先将下载得到的压缩包，解压：tar xzvf nasm-X.XX. ta .gz ；（可能会报错
gzip: stdin: not in gzip format
tar: Child returned status 1
tar: Error is not recoverable: exiting now  报错原因是这个压缩包没有用gzip格式压缩，所以不用加z指令就可以了）

然后cd  nasm-X. XX 并且 输入 ./configure {configure脚本会寻找最合适的C编译器，并生成相应的makefile文件}

接着输入make 创建nasm和ndisasm 的二进制代码

最后输入make install 进行安装（这一步需要root权限）

make install会将 nasm 和ndisasm 装进/usr/local/bin 并安装相应的man pages。

如果想验证是否安装成功的话，输入whereis nasm（见第一步）
```





## 2.编译过程

### 2.1事先编译一个hello.c源程序，代码如下：

#include <stdio.h>
int main(void)
 { printf("Hello World! \n"); return 0; }

### 2.2预处理命令，生成hello.i文件

gcc -E hello.c -o hello.i


### 2.3编译命令，生成hello.s文件（此时已经为汇编代码）

gcc -S hello.i -o hello.s

### 2.4汇编命令，将编译生成的 hello.s 文件汇编生成目标文件 hello.o

gcc -c hello.s -o hello.o

或调用 as 进行汇编

as -c hello.s -o hello.o

此时hello.o 目标文件为 ELF（Executable and Linkable Format）格式的可重定向文件。

### 2.5链接，将目标文件 hello.o生成 ELF 格式可执行文件

$ gcc hello.c -o hello

此时为动态链接。

gcc -static hello.c -o hello

此时为静态链接。
ELF 文件格式包含
1 .text：已编译程序的指令代码段；
2 .rodata：ro 代表 read only，即只读数据（譬如常数 const）；
3 .data：已初始化的 C 程序全局变量和静态局部变量；
4.bss：未初始化的 C 程序全局变量和静态局部变量；
5 .debug：调试符号表，调试器用此段的信息帮助调试。
可以使用如下代码查看汇编ELF各个部分：

readelf -S

可以使用如下代码查看反汇编ELF各个部分：

objdump -D 

## 3.汇编语言格式（.asm)

### 3.1写入hello.asm文件，具体代码如下：

; hello.asm 
section .data            ; 数据段声明
        msg db "Hello, world!", 0xA     ; 要输出的字符串
        len equ $ - msg                 ; 字串长度
section .text            ; 代码段声明
global _start            ; 指定入口函数
_start:                  ; 在屏幕上显示一个字符串
        mov edx, len     ; 参数三：字符串长度
        mov ecx, msg     ; 参数二：要显示的字符串
        mov ebx, 1       ; 参数一：文件描述符(stdout) 
        mov eax, 4       ; 系统调用号(sys_write) 
        int 0x80         ; 调用内核功能
                         ; 退出程序
        mov ebx, 0       ; 参数一：退出代码
        mov eax, 1       ; 系统调用号(sys_exit) 
        int 0x80         ; 调用内核功能



### 3.2由hello.asm生成可执行文件，输入如下命令

nasm -f elf64 hello.asm
ld -s -o hello hello.o
./hello



原文链接：https://blog.csdn.net/qq_44644740/article/details/109086520





## 4.( * (void ( * ) () )shellcode) ()

typedef void (*Fun) (void)

 Fun 是一个函数的指针，该函数的返回值为void类型，函数没有参数（没有参数需要指定void）。

​		void (*)()其函数返回类型为void，即是不返回任何内容，其后面为()，并没有指定确定的参数，也没有指定void，证明其需要不定量的参数个数

​	 (void (*)()) shellcode将shellcode强制转换成这种类型，即现在shellcode是一个指向函数的指针。

然后通过 (*函数指针)()来调用了这个函数。



