---
title: 编译->链接
draft: false
tags:
  - 操作系统
  - C++
---

链接（linking）是将各种代码和数据片段收集并组合成一个单一文件的过程，这个文件可以被加载到内存并运行。 链接可以执行于编译时、加载时、运行时。链接是由链接器程序自动执行的。

链接可以将大型项目分解成小部分，每次修改只用重新编译修改的模块，再重新链接即可。

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
- Class：表示文件采用的地址位宽，ELF64表示位宽64位。
- Data：指定数据编码方式，这里表示2进制补码，且采用小端（低位在前）。
- Version：采用的ELF格式版本号。
- OS/ABI：



## 目标文件.o
