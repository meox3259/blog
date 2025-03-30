---
title: 编译->链接
draft: false
tags:
  - 操作系统
  - C++
---

链接（linking）是将各种代码和数据片段收集并组合成一个单一文件的过程，这个文件可以被加载到内存并运行。 链接可以执行于编译时、加载时、运行时。链接是由链接器程序自动执行的。

链接可以将大型项目分解成小部分，每次修改只用重新编译修改的模块，再重新链接即可。

本文是针对CSAPP链接以及自己手动实验的学习。

# 链接步骤

## 生成中间文件.i
对于一个test.cc文件
```cpp
#include <cstdio>

int main() {
  int a = 1;
  printf("%d\n", a);
}
```
执行以下命令可以获得test.i文件
```shell
g++ -E test.cc -o test.i
```
对于test.i文件，可以发现对头文件进行了展开
```cpp
extern int printf (const char *__restrict __format, ...);
```
执行完后发现，对于test.cc文件中的头文件进行了展开，生成了头文件中函数的声明。

## 生成汇编.S

对于预处理好的test.i文件，需要进一步变成汇编test.S，执行以下命令
```bash
g++ -S test.i -o test.S
```
可以发现生成了test.S文件
```
	.file	"test.cc"
	.text
	.section	.rodata
.LC0:
	.string	"%d\n"
	.text
	.globl	main
	.type	main, @function
main:
.LFB2:
	.cfi_startproc
	endbr64
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	subq	$16, %rsp
	movl	$1, -4(%rbp)
	movl	-4(%rbp), %eax
	movl	%eax, %esi
	leaq	.LC0(%rip), %rax
	movq	%rax, %rdi
	movl	$0, %eax
	call	printf@PLT
	movl	$0, %eax
	leave
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
.LFE2:
	.size	main, .-main
	.ident	"GCC: (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0"
	.section	.note.GNU-stack,"",@progbits
	.section	.note.gnu.property,"a"
	.align 8
	.long	1f - 0f
	.long	4f - 1f
	.long	5
0:
	.string	"GNU"
1:
	.align 8
	.long	0xc0000002
	.long	3f - 2f
2:
	.long	0x3
3:
	.align 8
4:
```

## ELF文件
ELF（Executable and Linkable Format）主要包含以下4种文件
1. **可执行文件**：包含可直接运行的程序指令和数据。
2. **可重定位文件.o**：由编译器生成的中间文件，需通过链接器与其他文件合并成可执行文件或共享库。
3. **共享目标文件.so**：动态链接库，支持多个程序在运行时共享代码。
4. **核心转储文件**：记录程序崩溃时的内存状态，用于调试分析。

以下是ELF格式文件的结构
![ELF](https://p.sda1.dev/22/9c13c6a5f9706701be3a572a9bc9ba59/ELF.png)

### ELF头
将ELF头通过readelf展开
![ELF Header](https://p.sda1.dev/22/8a21befffeba252cd51a6ffa8c7d3ef7/header.png)

包含以下字段
- Magic: 如下图所示
![Magic Number](https://im.gurl.eu.org/file/AgACAgEAAxkDAAI2GWfNy81AAtt9wTp3aANFiYiUHhfBAAIZrTEbloRwRjv0ThFXKcO4AQADAgADeQADNgQ.png)
- Class：表示文件采用的地址位宽，ELF64表示位宽64位，对应`e_ident[EI_CLASS]。
- Data：指定数据编码方式，这里表示2进制补码，且采用小端（低位在前），对应`e_ident[data]`。
- Version：采用的ELF格式版本号，对应`e_ident[EI_VERSION]`。
- OS/ABI：目标操作系统和ABI类型，此处为`V`，对应`e_ident[EI_OSABI]`。
- ABI Version：ABI版本，通常为0，对应
- Type：文件类型，此处为可重定位文件.o，对应`e_type`
- Machine：目标CPU架构，此处为`x86_64`，对应`e_machine`字段
- Version：文件版本，通常与上文的`Version`一致，此处为`1`
- Entry point address：程序入口地址，由于可重定位文件无入口点，此处为`0`
- Start of program headers：节头表的偏移量，此处为`616`
- Flag：处理器特定标志，此处为`0`
- Size of this header：ELF头大小，此处为64字节（ELF64标准）
- Size of program headers：程序头表大小，此处为`0`，表示无
- Number of program header：程序头表条目数量，此处为`0`
- Size of section headers：节头表每个条目的大小，此处为`64 Byte`
- Number of section headers：节头表包含的条目数，此处为`14`
- Section header string table index：节头表中字符串索引为`13`，即第`14`个节头

ELF中的Section：
- .text：已编译的机器码
- .rodata：只读数据，比如printf语句中的格式串和开关语句跳转表
- .data：已初始化的全局和静态C变量。局部C变量在运行时被保存在栈中，既不出现在.data节中，也不出现在.bss节中
- .bss：未初始化/已初始化为0的全局或静态变量，在目标文件中仅仅是一个占位符，不占用空间。运行时，在内存中分配这些变量，初始值为0
- .symtab：一个符号表，它存放在程序中定义和引用的函数和全局变量的信息。每个可重定位目标文件在.symtab中都有一张符号表，但是和编译器中的符号表不同，.symtab中不包含局部变量的条目
- .rel.text：一个.text节中位置的列表，当链接器把这个目标文件和其他文件组合时，需要修改这些位置。对于可执行文件，由于不需要重定位信息，因此通常省略
- .rel.data：被模块引用或定义的所有全局变量的重定位信息
- .debug：一个调试符号表，保存了调试的信息，需要通过`-g`生成
	- 变量名、类型、作用域、内存地址
	- 函数名、参数类型、返回值类型、入口地址
	- 结构体、枚举等复杂类型的信息



## 目标文件.o
