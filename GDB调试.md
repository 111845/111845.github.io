# GDB调试

## 1. 概述

 GDB 全称“GNU symbolic debugger”

诞生于 GNU 计划（同时诞生的还有 GCC、Emacs 等），是 Linux 下常用的程序调试器。发展至今，GDB 已经迭代了诸多个版本，当下的 GDB 支持调试多种编程语言编写的程序，包括 C、C++、Go、Objective-C、OpenCL、Ada 等。实际场景中，GDB 更常用来调试 C 和 C++ 程序。一般来说，GDB主要帮助我们完成以下四个方面的功能：

1. 启动你的程序，可以按照你的自定义的要求随心所欲的运行程序。
2. 在某个指定的地方或条件下暂停程序。
3. 当程序被停住时，可以检查此时你的程序中所发生的事。
4. 在程序执行过程中修改程序中的变量或条件，将一个bug产生的影响修正从而测试其他bug。

使用GDB调试程序，有以下两点需要注意：

1. 要使用GDB调试某个程序，该程序编译时必须加上编译选项 **`-g`**，否则该程序是不包含调试信息的；
2. GCC编译器支持 **`-O`** 和 **`-g`** 一起参与编译。GCC编译过程对进行优化的程度可分为5个等级，分别为 ：

- **-O/-O0**： 不做任何优化，这是默认的编译选项 ；
- **-O1**：使用能减少目标文件大小以及执行时间并且不会使编译时间明显增加的优化。 该模式在编译大型程序的时候会花费更多的时间和内存。在 -O1下：编译会尝试减少代 码体积和代码运行时间，但是并不执行会花费大量时间的优化操作。
- **-O2**：包含 -O1的优化并增加了不需要在目标文件大小和执行速度上进行折衷的优化。 GCC执行几乎所有支持的操作但不包括空间和速度之间权衡的优化，编译器不执行循环 展开以及函数内联。这是推荐的优化等级，除非你有特殊的需求。 -O2会比 -O1启用多 一些标记。与 -O1比较该优化 -O2将会花费更多的编译时间当然也会生成性能更好的代 码。
- **-O3**：打开所有 -O2的优化选项并且增加 -finline-functions, -funswitch-loops,-fpredictive-commoning, -fgcse-after-reload and -ftree-vectorize优化选项。这是最高最危险 的优化等级。用这个选项会延长编译代码的时间，并且在使用 gcc4.x的系统里不应全局 启用。自从 3.x版本以来 gcc的行为已经有了极大地改变。在 3.x，，-O3生成的代码也只 是比 -O2快一点点而已，而 gcc4.x中还未必更快。用 -O3来编译所有的 软件包将产生更 大体积更耗内存的二进制文件，大大增加编译失败的机会或不可预知的程序行为（包括 错误）。这样做将得不偿失，记住过犹不及。在 gcc 4.x.中使用 -O3是不推荐的。
- **-Os**：专门优化目标文件大小 ,执行所有的不增加目标文件大小的 -O2优化选项。同时 -Os还会执行更加优化程序空间的选项。这对于磁盘空间极其紧张或者 CPU缓存较小的 机器非常有用。但也可能产生些许问题，因此软件树中的大部分 ebuild都过滤掉这个等 级的优化。使用 -Os是不推荐的。

## 2. 启用GDB调试

GDB调试主要有三种方式：

1. 直接调试目标程序：gdb ./hello_server
2. 附加进程id：gdb attach pid
3. 调试core文件：gdb filename corename

## 3. 退出GDB

- 可以用命令：**q（quit的缩写）或者 Ctr + d** 退出GDB。
- 如果GDB attach某个进程，退出GDB之前要用命令 **detach** 解除附加进程。

## 4. 常用命令

| 命令名称    | 命令缩写  | 命令说明                                         |
| ----------- | --------- | ------------------------------------------------ |
| run         | r         | 运行一个待调试的程序                             |
| continue    | c         | 让暂停的程序继续运行                             |
| next        | n         | 运行到下一行                                     |
| step        | s         | 单步执行，遇到函数会进入                         |
| until       | u         | 运行到指定行停下来                               |
| finish      | fi        | 结束当前调用函数，回到上一层调用函数处           |
| return      | return    | 结束当前调用函数并返回指定值，到上一层函数调用处 |
| jump        | j         | 将当前程序执行流跳转到指定行或地址               |
| print       | p         | 打印变量或寄存器值                               |
| backtrace   | bt        | 查看当前线程的调用堆栈                           |
| frame       | f         | 切换到当前调用线程的指定堆栈                     |
| thread      | thread    | 切换到指定线程                                   |
| break       | b         | 添加断点                                         |
| tbreak      | tb        | 添加临时断点                                     |
| delete      | d         | 删除断点                                         |
| enable      | enable    | 启用某个断点                                     |
| disable     | disable   | 禁用某个断点                                     |
| watch       | watch     | 监视某一个变量或内存地址的值是否发生变化         |
| list        | l         | 显示源码                                         |
| info        | i         | 查看断点 / 线程等信息                            |
| ptype       | ptype     | 查看变量类型                                     |
| disassemble | dis       | 查看汇编代码                                     |
| set args    | set args  | 设置程序启动命令行参数                           |
| show args   | show args | 查看设置的命令行参数                             |
| nexti       | ni        | 执行下一行(以汇编代码为单位)                     |
| stepi       | si        | 执行下一行(以汇编代码为单位)     进入函数        |

## 5. 常用命令示例

### 5.1 run命令

 默认情况下，以 `gdb ./filename` 方式启用GDB调试只是附加了一个调试文件，并没有启动这个程序，需要输入run命令（简写为r）启动这个程序

### 5.2 continue命令

当GDB触发断点后，想让程序继续运行，只要输入 continue（简写为c）命令即可。

### 5.3 break命令

break命令（简写为b）用于添加断点，可以使用以下几种方式添加断点：

- **break FunctionName**，在函数的入口处添加一个断点；	**（目前用到）**
- **break LineNo**，在**当前文件**行号为**LineNo**处添加断点；
- **break FileName:LineNo**，在**FileName**文件行号为**LineNo**处添加一个断点；
- **break FileName:FunctionName**，在**FileName**文件的**FunctionName**函数的入口处添加断点；
- **break -/+offset**，在当前程序暂停位置的前/后 offset 行处下断点；

### 5.4 info break、enable、disable和delete命令

 命令格式及作用：

- **info break**，也可简写为 **i b**，作用是显示当前所有断点信息；
- **disable 断点编号**，禁用某个断点，使得断点不会被触发；
- **enable 断点编号**，启用某个被禁用的断点；
- **delete 断点编号**，删除某个断点。

### 5.5 backtrace和frame命令

 命令格式及作用：

- **backtrace**，也可简写为 **bt**，用于查看当前调用堆栈。
- **frame 堆栈编号**，也可简写为 **f 堆栈编号**，用于切换到其他堆栈处。

### 5.6 list命令

命令格式及作用：

- **list**，输出上一次list命令显示的代码后面的代码，如果是第一次执行list命令，则会显示当前正在执行代码位置附近的代码；
- **list -**，带一个减号，显示上一次list命令显示的代码前面的代码；
- **list LineNo**，显示当前代码文件第 **LineNo** 行附近的代码；
- **list FileName:LineNo**，显示 **FileName** 文件第 **LineNo** 行附近的代码；
- **list FunctionName**，显示当前文件的 **FunctionName** 函数附近的代码；
- **list FileName:FunctionName**，显示 **FileName** 文件的 **FunctionName** 函数附件的代码；
- **list from,to**，其中**from**和**to**是具体的代码位置，显示这之间的代码；



**list**命令默认只会输出 **10** 行源代码，也可以使用如下命令修改：

- **show listsize**，查看 **list** 命令显示的代码行数；
- **set listsize count**，设置 **list** 命令显示的代码行数为 **count**;



参考文献： https://zhuanlan.zhihu.com/p/297925056

https://firmianay.gitbooks.io/ctf-all-in-one/content/doc/2.3.1_gdb.html#gdb