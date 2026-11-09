# tmp0ut 首刊第二篇文章知识点拆解

- 原文：https://tmpout.sh/1/2.html
- 本文是对原文的知识点拆解笔记，配图见同名 PDF：[pdf/01-elf-file-structure.pdf](../pdf/01-elf-file-structure.pdf)

---

## 一、ELF 文件简述

> 原文此处配图：左为构建视图（程序构建时需要的），右为执行视图（程序执行时需要的），见 PDF。

上图是 elf 程序的构成，左边是构建视图（即程序构建的时候需要的），右边是执行视图（即程序执行时需要的）。

程序构建时，需要 elf 头，以及节头表中描述的信息，即左侧箭头指向的，根据节头表去合并各个"节"，此时忽略程序头表。

程序执行时，则不需要节头表了，只需要 elf 头，程序头中描述的段信息，根据程序头表把多个"节"合并成的"段（Segment）"加载到内存中，忽略节头表。节头表主要提供给 readelf 等分析器使用。

### ELF 的类型

ELF 主要包括三种类型文件：

- 可重定位文件（relocatable）：编译器和汇编器产生的 `.o` 文件，被 Linker 所处理
- 可执行文件（executable）：Linker 对 `.o` 文件进行处理输出的文件，进程映像
- 共享对象文件（shared object）：动态库文件 `.so`

### ELF 头和程序头

> 原文此处配图，见 PDF。

在 Linux 源码中，这二者主要表现为这两个结构体，具体路径为 `kernel-4.4.94/include/uapi/linux/elf.h`。

其中，elf 头如下：

```c
#define EI_NIDENT 16
typedef struct{
    /*ELF 的一些标识信息，固定值 */
    unsigned char e_ident[EI_NIDENT];
    /* 目标文件类型： 1-可重定位文件， 2-可执行文件， 3-共享目标文件等 */
    Elf32_Half e_type;
    /* 文件的目标体系结构类型： 3-intel 80386*/
    Elf32_Half e_machine;
    /* 目标文件版本： 1-当前版本 */
    Elf32_Word e_version;
    /* 程序入口的虚拟地址，如果没有入口，可为 0*/
    Elf32_Addr e_entry;
    /* 程序头表 (segment header table) 的偏移量，如果没有，可为 0*/
    Elf32_Off e_phoff;
    /* 节区头表 (section header table) 的偏移量，没有可为 0*/
    Elf32_Off e_shoff;
    /* 与文件相关的，特定于处理器的标志 */
    Elf32_Word e_flags;
    /*ELF 头部的大小，单位字节 */
    Elf32_Half e_ehsize;
    /* 程序头表每个表项的大小，单位字节 */
    Elf32_Half e_phentsize;
    /* 程序头表表项的个数 */
    Elf32_Half e_phnum;
    /* 节区头表每个表项的大小，单位字节 */
    Elf32_Half e_shentsize;
    /* 节区头表表项的数目 */
    Elf32_Half e_shnum;
    /* 某些节区中包含固定大小的项目，如符号表。对于这类节区，此成员给出每个表项的长度字节数。 */
    Elf32_Half e_shstrndx;
}Elf32_Ehdr;
```

注意，上面的 `e_shstrndx` 需要额外讲一下，这是指向节头表（节头表是一个数组）某一位置的索引。该索引对应的节包含节头表中每一节的名称。

> 原文此处配图（示例），见 PDF。

程序头则如下：

程序头部分，不只是一份这个结构体，而是以一个结构体数组的形式存在，该数组的每个成员，都是下面的一个结构体，因此严格来说，应该是程序头表，表中的每一项是程序头，不过一般都直接说程序头。

> 原文此处配图（示例），见 PDF。

```c
typedef struct
{
    /*segment 的类型： PT_LOAD=1 可加载的段 */
    Elf32_Word p_type;
    /* 从文件头到该段第一个字节的偏移 */
    Elf32_Off p_offset;
    /* 该段第一个字节被放到内存中的虚拟地址 */
    Elf32_Addr p_vaddr;
    /* 在 linux 中这个成员没有任何意义，值与 p_vaddr 相同 */
    Elf32_Addr p_paddr;
    /* 该段在文件映像中所占的字节数 */
    Elf32_Word p_filesz;
    /* 该段在内存映像中占用的字节数 */
    Elf32_Word p_memsz;
    /* 段标志 */
    Elf32_Word p_flags;
    /*p_vaddr 是否对齐 */
    Elf32_Word p_align;
} Elf32_phdr;
```

在 32 位 mips 中，其内存分布为：

```text
      ├─ p_type    4 字节
      ├─ p_offset  4 字节
      ├─ p_vaddr   4 字节
      ├─ p_paddr   4 字节
      ├─ p_filesz  4 字节
      ├─ p_memsz   4 字节
      ├─ p_flags   4 字节
      └─ p_align   4 字节
```

> 原文此处配图（示例），见 PDF。

---

## 二、ELF 注入步骤

### 1. 解析 elf 头和程序头

拿到目标 elf 文件，使用解析器解析其 elf 头和程序头。

> 原文此处配图：红框中是需要关注的信息，见 PDF。

### 2. 构建裸二进制文件

即要注入的文件。建议使用汇编构造，如果使用 C 的话，可能存在依赖问题（需要依赖 uclibc 什么的），而且不利于控制生成物大小；对汇编不熟的可以使用 AI 辅助构建，很简单的。不过在不同架构中，需要尽量规避可能影响生成物大小的点，以精确控制生成物大小。

示例以及详解（此代码为 MIPS 架构）：

```asm
// 二进制文件源码
.set noreorder // 关闭自动重排，MIPS 存在分支延迟槽，若不关闭这个，则汇编器会在跳转指令后自动
               // 填充，从而增大生成物大小
.set noat      // 禁止使用 at 临时寄存器，防止其进行命令展开，从而增大生成物大小
.text          // 声明后续属于可执行代码段，方便后续提取
.globl _start
_start:
    li $v0, 4004
    li $a0, 1
    lui $a1, %hi(msg)
    addiu $a1, $a1, %lo(msg)
    li $a2, msg_len
    syscall
    lui $t9, %hi(0x004004b0)
    addiu $t9, $t9, %lo(0x004004b0)
    jr $t9   // 跳转到最开始的 elf 加载入口，即前面所标出的 Entry point address
    nop
msg:
    .ascii "hi hi hi,this is made by zcy!\n"
msg_len = . - msg
```

#### 2-1. 剥离其余部分，只留下裸二进制执行程序部分

将上述代码编译后，使用这个指令剥离其余部分：

```text
mips-linux-gnu-objcopy -O binary -j .text payload.elf payload.bin
```

不同架构具体命令可能不一样，但基本都是使用 objcopy。

这个命令的原理是提取 elf 文件中的 `.text` 段，这个信息来源于节头表，数据存储在内容区，不懂的可以返回文档顶看一下那个图：

```text
        文件格式： ELF
         │
         ├─ ELF 头                             64 字节
         │
         ├─ 程序头表                    13 × 56 = 728 字节
         │    └─ 第 4 条 LOAD（R E）：0x4000 ~ 0x17581
         │          └─ 把 .text 那 75,730 字节包进可执行段
         │
         ├─ 内容区                         139,409 字节
         │    ├─ .interp
         │    ├─ .note.*
         │    ├─ .gnu.hash / .dynsym / .dynstr
         │    ├─ .rela.dyn / .rela.plt
         │    ├─ .init / .plt / .plt.sec
         │    ├─ .text      0x4DA0 起，75,730 字节    ◄── 就是它
         │    ├─ .fini
         │    ├─ .rodata / .eh_frame
         │    ├─ .init_array / .dynamic / .got
         │    ├─ .data / .bss
         │    └─ … （共 30 个节）
         │
         └─ 节头表                      30 × 64 = 1,920 字节
              └─ 第 16 条记录：
                   sh_name  = 0xA9  →  ".text"
                   sh_flags = A | X
                   sh_addr  = 0x4DA0
                   sh_size  = 0x127D2
```

#### 2-2. 将上述文件连接到目标 elf 尾部

```text
cat elf_practice/payload.bin >> hello_mips32
```

更进阶的方式是直接塞入 elf 文件中由于填充和对齐产生的 padding 中，这样可以在不改变文件大小的前提下完成注入，但此处不予演示了。

#### 2-3. 修改 p_type

```text
printf '\x01\x00\x00\x00' | dd of=hello_mips32 bs=1 seek=$((0x114)) conv=notrunc
```

把 `PT_NOTE` 段的 `p_type` 修改为 `PT_LOAD`，这样就可以用来执行了。

上述语句的含义是：将 `\x01\x00\x00\x00` 四个字节，直接写入从文件（`hello_mips32`，即目标文件）开头处偏移 `0x114` 处，每次写一字节，`conv=notrunc` 的含义是不截断之后的数据，这个必须要加，否则写完之后文件会废掉。

为什么是 `0x114`？

`52+32*7=276`，其 16 进制就是 `0x114`，具体我就不展开了，可以看前面的 ELF 头部分标红框的那几个信息。

为什么修改 `PT_NOTE` 这一段？

因为绝大部分 elf 中，这一段在真正运行的时候不起作用，若修改其他段，可能导致原 elf 无法运行。

此处我这么组织这四个字节，是因为 MIPS 架构的小端序，其他架构应该不可以直接复制，可让 AI 适配一下。

修改后，再用 readelf 读取，即可发现其已经变为 LOAD，修改成功。

> 原文此处配图（readelf 结果），见 PDF。

#### 2-4. 修改 p_offset，使其指向 payload 起点

```text
printf '\x8C\x18\x00\x00' | dd of=hello_mips32 bs=1 seek=$((0x118)) conv=notrunc
```

输入的字节为 `0x188c`，即文件大小的 16 进制，也即 payload 的起点。

修改完可以发现：

> 原文此处配图，见 PDF。

#### 2-5. 修改 p_vaddr 和 p_paddr（虚拟内存地址基址和物理内存地址基址）

```text
printf '\x8C\x18\x00\x0C' | dd of=hello_mips32 bs=1 seek=$((0x11C)) conv=notrunc
printf '\x8C\x18\x00\x0C' | dd of=hello_mips32 bs=1 seek=$((0x120)) conv=notrunc
```

为什么是 `0x11c` 和 `0x120`？

在前面计算好的 `0x114`（这个也就是当前修改的这个程序头）的基础上，根据结构体中每个成员的数据类型大小，递推。由于该结构体中所有成员都是 4 字节，是对齐的，因此无需考虑结构体的对齐。

> PS：什么是结构体的对齐，可以看附录 1，我有讲。

为什么是 `0x0c00188c`？

`0c000000` 处基本没人用，因此将 payload 放到此处执行，不会损坏其他程序，从而导致报错。

而加上 `188c`（即 offset），是因为内核计算这一段时，会通过下述方式计算出页边界，完成页取整的操作，再通过 mmap 完成虚拟地址与物理地址之间的转换。若基址选择 `0c000000`，则因为其页取整后偏移为 0，而文件页取整后偏移为 `0x88c`，而违背了一一对应的原则：

```text
页内偏移      = p_vaddr & 0xFFF = 0x0C00188C & 0xFFF = 0x88C
// 0xFFF=4095，而 4096 正好是一页的大小，页则是内存操作的最小单位
映射的内存起始 = p_vaddr - 页内偏移 = 0x0C00188C - 0x88C = 0x0C001000
映射的文件偏移 = p_offset - 页内偏移 = 0x188C - 0x88C = 0x1000
映射长度       = p_filesz + 页内偏移 = 0x54 + 0x88C = 0x8E0
// 为什么要这么计算，因为页是最小的操作单位，因此必须整页复制，而又因为为了维持物理内存和虚拟
// 内存的一一对应，因此直接使用 p_vaddr & 0xFFF 作为页内偏移进行运算
```

于是内核执行下面这个：

```text
mmap(地址 = 0x0C001000, 长度 = 0x8E0, 权限 = 读+执行, 文件偏移 = 0x1000)
```

#### 2-6. 修改 p_filesz 和 p_memsz（程序段待执行内容的大小，和其在 mem 上所需要占用的大小）

```text
printf '\x54\x00\x00\x00' | dd of=hello_mips32 bs=1 seek=$((0x124)) conv=notrunc
printf '\x54\x00\x00\x00' | dd of=hello_mips32 bs=1 seek=$((0x128)) conv=notrunc
```

> PS：此处的标准是 `p_memsz` 必须大于 `p_filesz`，此处采取的值是相等，因为使用汇编构建，因此不需要冗余。

正常情况下，二者之差会被填充为 0，同时作为 C 语言里未初始化的全局变量（`.bss` 节）的存放位置。

`0x54` 是 84，即 `payload.bin` 的大小。

#### 2-7. 修改 p_flags（添加执行权限）

```text
printf '\x05\x00\x00\x00' | dd of=hello_mips32 bs=1 seek=$((0x12C)) conv=notrunc
```

`5 = PF_R(4) | PF_X(1)`，表示"可读 + 可执行"。原来的值是 `4`（只有可读）。

#### 2-8. 修改 p_align

```text
printf '\x00\x10\x00\x00' | dd of=hello_mips32 bs=1 seek=$((0x130)) conv=notrunc
```

`0x1000 = 4096`，页大小。这告诉内核这个段需要按 4KB 对齐。

为什么要修改这个？

因为内存的最小操作单位是页，我们前面有 mmap 操作，需要按照这个标准对齐。

为什么原来作为 NOTE 段时，是按照字节对齐的？

NOTE 段因为其在节头表中含有 A 标志，因此被前面的 LOAD 一起带入内存。而我们修改之后，该段变成了新的 LOAD，因此需要更改。

#### 2-9. 修改 e_entry

```text
printf '\x90\x18\x00\x0C' | dd of=hello_mips32 bs=1 seek=$((0x18)) conv=notrunc
```

目的是让 elf 先去执行我们的代码，然后再执行它自己的。

### 3. 板端运行现象

> 原文此处配图：上述分别为原版和注入版的修改现象，见 PDF。

上述分别为原版和注入版的修改现象。可以看到，程序确实先去执行了我们的注入程序，再执行了其自己自带的程序。

不过后面还有 bus error 报错，这个问题是由于注入的那段汇编程序中，不小心将值覆盖进了链接器要使用的一个寄存器 v0，因此产生此报错。修复后正常。（由于我对 MIPS 汇编不熟，因此涉及到汇编，都是使用 AI 辅助写的。）

---

## 附录 1：C 语言结构体的大小、对齐，以及偏移——以及 padding 泄露

> 原文此处配图，见 PDF。

### C 语言结构体的偏移计算方式

C 语言结构体成员的偏移量，是从 0 开始，根据上一个成员结束的位置，按当前成员自身的对齐值向上取整得到的；而结构体的总大小，则必须按所有成员中最大的对齐值向上取整补齐。

此处提到的对齐值，是根据 ABI、编译器，以及编译选项和配置，规定好的。

举例说明：

```c
struct demo { char a; int b; char c; };
// char 的大小是 1 字节，int 的则是 4 字节
/*
偏移依次为
a:0
b:4  因为 1 不为 4 的倍数，所以添加到 4
c:8  b 起始为 4，自身大小为 4，因此 c 起始为 8
*/
```

> 原文此处配图，见 PDF。

### C 语言结构体的大小计算方式和对齐方式

在大小上，C 语言结构体的最终大小，是在偏移的计算结果上，基于最大的成员类型的大小，进行补齐，使总大小为最大的成员大小的倍数。

举例说明：

```c
struct demo { char a; int b; char c; };
// char 的大小是 1 字节，int 的则是 4 字节
/*
根据上面偏移所计算，三个成员总大小占据了 9 字节
但是 9 并非最大的成员（也就是 int）的大小的倍数，因此填补到 12。因此该结构体大小为 12 字节
*/
```

### padding 泄露

> 原文此处配图，见 PDF。

可以注意到，上述过程中，存在很多空洞，理论上在 ABI、编译器、编译配置，以及内核配置等一系列环境对齐的情况下，这些空洞是可预测的。

而当用户使用结构体时，没有进行初始化，该结构体的空洞中留存的是栈上之前的数据，而此时将结构体整体拷贝，则可以将该数据带出。

> 上述提到的初始化，是指下面的两种，为结构体的每一个成员赋值，不算此处提到的初始化。因为空洞的值你赋不进去哈哈。

```c
struct demo d = {0};
// 或者
struct demo d;
memset(&d, 0, sizeof(d));
```

但是更常用的，是利用用户未初始化、且未赋值的结构体成员，这些往往更好预测，不过原理与这个一致。
