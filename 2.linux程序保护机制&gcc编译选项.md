# linux程序保护机制&gcc编译选项

## 总览

```
Canary：-fno-stack-protector /-fstack-protector / -fstack-protector-all (关闭 / 开启 / 全开启)
NX：-z execstack / -z noexecstack (关闭 / 开启)
PIE：-no-pie / -pie (关闭 / 开启)
RELRO：-z norelro / -z lazy / -z now (关闭 / 部分开启 / 完全开启)
fority：-D_FORTIFY_SOURCE=1  /  -D_FORTIFY_SOURCE=2 	// (较弱的检查/较强的检查)
```



# 一：canary（栈保护）

​		栈溢出保护是一种缓冲区溢出攻击缓解手段，当函数存在缓冲区溢出攻击漏洞时，攻击者可以覆盖栈上的返回地址来让shellcode能够得到执行。

原理

​		当启用栈保护后，函数开始执行的时候会先往栈里插入cookie信息，当函数真正返回的时候会验证cookie信息是否合法，如果不合法就停止程序运行。攻击者在覆盖返回地址的时候往往也会将cookie信息给覆盖掉，导致栈保护检查失败而阻止shellcode的执行。在Linux中我们将cookie信息称为canary。

​	gcc在4.2版本中添加了-fstack-protector 和 -fstack-protector-all编译参数以支持栈保护功能，在编译时**可以控制是否开启栈保护以及程度**，例如：

```
1、gcc -o test test.c // 默认情况下，不开启Canary保护
2、gcc **-fno-stack-protector** -o test test.c //禁用栈保护
3、gcc **-fstack-protector** -o test test.c //启用堆栈保护，不过只为局部变量中含有 char 数组的函数插入保护代码
4、gcc **-fstack-protector-all** -o test test.c //启用堆栈保护，为所有函数插入保护代码
```

# 二：NX（DEP）

​	NX即No-eXecute（不可执行）的意思，NX（DEP）的基本原理是将数据所在内存页标识为不可执行，当程序溢出成功转入shellcode时，程序会尝试在数据页面上执行指令，此时CPU就会抛出异常，而不是去执行恶意指令。![image-20220901092607526](C:\Users\jinwe\AppData\Roaming\Typora\typora-user-images\image-20220901092607526.png)

gcc编译器默认开启了NX选项，如果需要关闭NX选项，可以给gcc编译器添加-z execstack参数。 例如：

```
1、gcc -o test test.c // 默认情况下，开启NX保护

2、gcc -z execstack -o test test.c // 禁用NX保护

3、gcc -z noexecstack -o test test.c // 开启NX保护
```

在Windows下，类似的概念为DEP（数据执行保护）

# 三：PIE（position-independent executables）    (windows平台上ASLR)

 一般情况下NX（Windows平台上称其为DEP）和地址空间分布随机化（ASLR）会同时工作	

​	**位置独立的可执行区域**。这样使得在利用缓冲溢出和移动操作系统中存在的其他内存崩溃缺陷时采用面向返回的编程（return-oriented programming）方法变得难得多。一般情况下NX（Windows平台上称其为DEP）和地址空间分布随机化（ASLR）会同时工作。内存地址随机化机制（address space layout randomization)，有以下三种情况：

```
0 - 表示关闭进程地址空间随机化。

1 - 表示将mmap的基址，stack和vdso页面随机化。

2 - 表示在1的基础上增加栈（heap）的随机化。
```

liunx下关闭PIE的命令如下：

```
sudo -s echo 0 > /proc/sys/kernel/randomize_va_space
```

gcc编译命令：

```
1、gcc -o test test.c // 默认情况下，不开启PIE

2、gcc -fpie -pie -o test test.c // 开启PIE，此时强度为1

3、gcc -fPIE -pie -o test test.c // 开启PIE，此时为最高强度2

4、gcc -fpic -o test test.c // 开启PIC，此时强度为1，不会开启PIE

5、gcc -fPIC -o test test.c // 开启PIC，此时为最高强度2，不会开启PIE
```

- 使用-fPIE编译的对象就能通过连接器得到位置无关可执行程序。
- -fpie和-fPIE选项和fpic及fPIC很相似，但不同的是，除了生成为位置无关代码外，还能假定代码是属于本程序。

# 四：RELRO（ read only relocation）

​		在Linux系统安全领域，数据可以写的存储区就会是攻击的目标，尤其是存储函数指针的区域。 所以在安全防护的角度来说尽量减少可写的存储区域对安全会有极大的好处。

**原理**

​	GCC, GNU linker以及Glibc-dynamic linker一起配合实现了一种叫做relro的技术:。大概实现就是由linker指定binary的一块经过dynamic linker处理过 relocation之后的区域为只读.设置符号重定向表格为只读或在程序启动时就解析并绑定所有动态符号，从而减少对GOT（Global Offset Table）攻击。RELRO为” Partial RELRO”，说明我们对GOT表具有写权限。

gcc编译：

```
gcc -o test test.c // 默认情况下，是Partial RELRO

gcc -z norelro -o test test.c // 关闭，即No RELRO

gcc -z lazy -o test test.c // 部分开启，即Partial RELRO

gcc -z now -o test test.c // 全部开启
```

# 五：fority

​		 fority其实非常轻微的检查，用于检查是否存在缓冲区溢出的错误。适用情形是程序采用大量的字符串或者内存操作函数，如`memcpy，memset，stpcpy，strcpy，strncpy，strcat，strncat，sprintf，snprintf，vsprintf，vsnprintf，gets`以及宽字符的变体。

**原理**

​	 Fortify 技术是GCC在编译源码时判断程序的哪些buffer会存在可能的溢出，在buffer大小已知的情况下，GCC会把 `strcpy`、`memcpy`、`memset`等函数自动替换成相应的`__strcpy_chk`(`dst`, `src`, `dstlen`)等函数，达到防止缓冲区溢出的作用。



`gcc -D_FORTIFY_SOURCE=1`仅仅只会在编译时进行检查 (特别像某些头文件 #include <string.h>)
 `gcc -D_FORTIFY_SOURCE=2` 程序执行时也会有检查 (如果检查到缓冲区溢出，就终止程序)

**gcc编译：**

```
gcc -o test test.c							// 默认情况下，不会开这个检查
gcc -D_FORTIFY_SOURCE=1 -o test test.c		// 较弱的检查
gcc -D_FORTIFY_SOURCE=2 -o test test.c		// 较强的检查
```

 FORTIFY_SOURCE机制**对格式化字符串有两个限制**：

 (1)包含%n的格式化字符串不能位于程序内存中的可写地址；

 (2)当使用位置参数时，必须使用范围内的所有参数。例如要使用%4$x，则必须同时使用1、2、3。





https://www.cnblogs.com/yidianhan/p/13996928.html#

https://www.jianshu.com/p/91fae054f922