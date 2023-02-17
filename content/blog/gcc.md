+++
title = "GCC 笔记"
description = ""
date = 2022-02-10
+++


## 常用OPTIONS

* -c: Compile or assemble the source files, but do not link.  The linking stage simply is not done.  The ultimate output is in the form of an object file for each source file.

对源文件进行编译或汇编，但不进行链接。源文件不包含main函数时，需要添加该选项才能编译通过。

* -m32: The -m32 option sets "int", "long", and pointer types to 32 bits, and generates code that runs on any i386 system.

该选项使编译器生成能够在i386系统上运行的代码

* -fno-pie: Don't produce a dynamically linked position independent executable.

没有这个选项，生成的文件会包含一些奇怪的东西 https://www.codeleading.com/article/21275815238/
https://stackoverflow.com/questions/50105581/how-do-i-get-rid-of-call-x86-get-pc-thunk-ax



## 示例

### 编译适用于32位机器的代码库

```bash
> cat func_call_test1.c
int add(int x, int y) {
        return x + y;
}

int caller() {
        int t1 = 125;
        int t2 = 80;
        int sum = add(t1, t2);

        return sum;
}

> gcc -c -m32 -fno-pie func_call_test1.c

> objdump -d func_call_test1.o

func_call_test1.o:     file format elf32-i386


Disassembly of section .text:

00000000 <add>:
   0:   55                      push   %ebp
   1:   89 e5                   mov    %esp,%ebp
   3:   8b 55 08                mov    0x8(%ebp),%edx
   6:   8b 45 0c                mov    0xc(%ebp),%eax
   9:   01 d0                   add    %edx,%eax
   b:   5d                      pop    %ebp
   c:   c3                      ret

0000000d <caller>:
   d:   55                      push   %ebp
   e:   89 e5                   mov    %esp,%ebp
  10:   83 ec 10                sub    $0x10,%esp
  13:   c7 45 f4 7d 00 00 00    movl   $0x7d,-0xc(%ebp)
  1a:   c7 45 f8 50 00 00 00    movl   $0x50,-0x8(%ebp)
  21:   ff 75 f8                pushl  -0x8(%ebp)
  24:   ff 75 f4                pushl  -0xc(%ebp)
  27:   e8 fc ff ff ff          call   28 <caller+0x1b>
  2c:   83 c4 08                add    $0x8,%esp
  2f:   89 45 fc                mov    %eax,-0x4(%ebp)
  32:   8b 45 fc                mov    -0x4(%ebp),%eax
  35:   c9                      leave
  36:   c3
```



