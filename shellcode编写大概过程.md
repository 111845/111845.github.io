# 1.首先确定将要使用的系统调用

常用 execve  ,编译完成后可以用strace追踪

# 2.用汇编语言中实现 系统调用

查看系统调用号

cat /usr/include/asm/unistd_64.h | grep   xxx

查询系统调用

https://hackeradam.com/x86-64-linux-syscalls/    64位

[OPEN - Linux手册页-之路教程 (onitroad.com)](https://www.onitroad.com/jc/linux/man-pages/linux/man2/open.2.html)

根据系统调用编写汇编语言

nasm -felf64 test.asm -o test.o   //汇编
ld test.o -o test	

objdump -d -M intel test   //查看汇编



# 3.提取机器码

```
objdump -M intel -D test | grep '[0-9a-f]:' | grep -v 'file' | cut -f2 -d: | cut -f1-7 -d' ' | tr -s ' ' | tr '\t' ' ' | sed 's/ $//g' | sed 's/ /\\\x/g' | paste -d '' -s
```

# 4.查看机器码中是否存在 bad character 若存在需要消除

消除赋值产生bad character    

消除有关地址产生的bad character 

python  //启动python
string = "//bin/sh"
string[::-1].encode('hex')

# 5.c语言验证



```
#include <stdio.h>
#include <string.h>
int main()
{
 	const char shellcode[] = "";
    printf("Shellcode length: %d bytes\n",					(int)strlen(shellcode));
    (*(void(*)())shellcode)();
}
```

gcc 1.c -o 1 -z execstack						-z     	norelro -no-pie -g