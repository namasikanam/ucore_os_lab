# Lab1 实验报告

## 练习1

Q1：操作系统镜像文件 ucore.img 是如何一步一步生成的？(需要比较详细地解释 Makefile 中每一条相关命令和命令参数的含义，以及说明命令导致的结果)

```makefile

KINCLUDE	+= kern/debug/ \
			   kern/driver/ \
			   kern/trap/ \
			   kern/mm/

KSRCDIR		+= kern/init \
			   kern/libs \
			   kern/debug \
			   kern/driver \
			   kern/trap \
			   kern/mm

KCFLAGS		+= $(addprefix -I,$(KINCLUDE))

# 批量生成 .o 文件
$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))

KOBJS	= $(call read_packet,kernel libs)

# 得到 kernel 的准确相对路径 bin/kernel
kernel = $(call totarget,kernel)

# 
$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
    # ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/stdio.o obj/kern/libs/readline.o obj/kern/debug/panic.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/picirq.o obj/kern/driver/intr.o obj/kern/trap/trap.o obj/kern/trap/vectors.o obj/kern/trap/trapentry.o obj/kern/mm/pmm.o  obj/libs/string.o obj/libs/printfmt.o
    # -m elf_i386 模仿 elf_i386 链接器。
    # -nostdlib 只搜索在命令行中显式声明的库目录。
    # -T tools/kernel.ld 用 tools/kernel.ld 作为链接脚本
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
    # objdump -S bin/kernel
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
    # objdump -t bin/kernel
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

# -------------------------------------------------------------------

# bootfiles 包括 bootasm 和 bootmain
bootfiles = $(call listf_cc,boot)

# gcc -Iboot/ -march=i686 -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
# gcc -Iboot/ -march=i686 -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
# -Iboot/ 将 boot/ 加入到预处理时搜索头文件的目录列表中
# -march i686 目标架构是 i686
# -fno-builtin 只将以 __builtin__ 开头的函数视为内置函数
# -fno-PIC 在 x86 环境下无影响
# -m32 在 x86 的机器上，表示生成32位环境下的代码
# -gstabs 生成stabs格式的调试信息（如果支持该格式的话），忽略GDB的特有扩展
# -nostdinc 不在标准系统文件夹中搜索头文件
# -fno-stack-protector 不检查缓冲区溢出
# -Ilibs/ 将 libs/ 加入到预处理时搜索头文件的目录列表中
# -Os 依据代码大小来优化
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

# 得到 bootblock 的准确相对路径 bin/bootblock
bootblock = $(call totarget,bootblock)

# 构建 bootblock
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
    # ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
    # -m elf_i386 模仿 elf_i386 链接器。
    # -nostdlib 只搜索在命令行中显式声明的库目录。
    # -N 将text段和data段设为可读和可写的。
    # -e start 用 start 作为用于程序执行的显式符号，以替换此前的默认进入节点。
    # -Ttext 0x7c00 指定text段的开始位置为0x7c00
    # 将 obj/boot/bootasm.o 和 obj/boot/bootmain.o 链接起来，作为 obj/bootblock.o
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
    # objdump -S obj/bootblock.o > obj/bootblock.asm
    # -S 表示输出混有反汇编的源代码
    # 从 bootblock.o 中抓取信息放到 bootblock.asm
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
    # objcopy -S -O binary obj/bootblock.o obj/bootblock.out
    # -S 表示不从源代码中复制重定位和符号信息
    # -O binary 表示以 binary 格式写输出文件
    # 用 objcopy 将 bootblock.o 转成binary格式的 bootblock.out
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
    # bin/sign obj/bootblock.out bin/bootblock
    # 用 sign 将 bootblock.out 转成最终的执行文件 bootblock
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

# -------------------------------------------------------------------

# gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
# -I tools/ 将 tools/ 加入到预处理时搜索头文件的目录列表中
# 生成 sign.o
$(call add_files_host,tools/sign.c,sign,sign)

# gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
# 生成 sign
$(call create_target_host,sign,sign)

# -------------------------------------------------------------------

# 得到 UCOREIMG 的准确相对路径 bin/ucore.img
UCOREIMG	:= $(call totarget,ucore.img)

# 生成 bin/ucore.img
$(UCOREIMG): $(kernel) $(bootblock)
    # dd if=/dev/zero of=bin/ucore.img count=10000
    # if 是输入文件
    # of 是输出文件
    # count 是输入的块的上限
    # 这里用10000块 0 来填充 ucore.img 作为初始化
	$(V)dd if=/dev/zero of=$@ count=10000
    # dd if=bin/bootblock of=bin/ucore.img conv=notrunc
    # notrunc 表示不会依据输入文件的长度来对输出文件作截断
    # 这里将 bootblock 拷贝进 ucore 中
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
    # dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
    # seek=1 表示跳过输出文件的第1块，因为这部分已经拷贝进 bookblock 了
    # 从第2块开始，将 kernel 也复制到 ucore.img 中
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
```

Q2: 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

从sign.c的代码来看，一个磁盘主引导扇区只有512字节。且
- 第510个（倒数第二个）字节是0x55，
- 第511个（倒数第一个）字节是0xAA。

其关键代码部分见：
```c++
if (st.st_size > 510) {
    fprintf(stderr, "%lld >> 510!!\n", (long long)st.st_size);
    return -1;
}
// ...
buf[510] = 0x55;
buf[511] = 0xAA;
```

## 练习2

1. 在一个bash中开启qemu
```bash
qemu -S -s -d in_asm -D q.log -monitor stdio -hda bin/ucore.img -serial null -curses
```
2. 在另一个bash中开启gdb并连接qemu
```bash
target remote:1234
file bin/kernel
file obj/bootblock.o
```
3. 在gdb中设置断点并开始单步调试
```bash
break *0x7c00
c
si
...
```
4. 可以看到其反汇编代码与 bootblock.S 和 bootblock.asm 中的相同。在 q.log 中，可以看到从 0x7c00 开始的一段机器码：
```as
IN: 
0x0000000000007c00:  cli    

----------------
IN: 
0x0000000000007c00:  cli    

----------------
IN: 
0x0000000000007c01:  cld    

----------------
IN: 
0x0000000000007c02:  xor    %ax,%ax
0x0000000000007c04:  mov    %ax,%ds
0x0000000000007c06:  mov    %ax,%es
0x0000000000007c08:  mov    %ax,%ss
```

---

由于我没有，与参考答案有以下几点不同：
- 由于我没有配置GUI，故我的 qemu 需要在 terminal 中呈现。
- 我直接使用的 x86 的 qemu 模拟器，不需要再在gdb中将指令集设为 i8086。

## 练习3

Q1: 为何开启A20，以及如何开启A20

大致步骤如下：
1. 等待8042 Input buffer为空；
2. 发送Write 8042 Output Port （P2）命令到8042 Input buffer；
3. 等待8042 Input buffer为空；
4. 将8042 Output Port（P2）得到字节的第2位置1，然后写入8042 Input buffer；
```as
    # Enable A20:
    #  For backwards compatibility with the earliest PCs, physical
    #  address line 20 is tied low, so that addresses higher than
    #  1MB wrap around to zero by default. This code undoes this.
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to P2 port of 8042

seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set A20 bit (the 1 bit) of 8042 to 1
```

Q2: 如何初始化GDT表

一个简单的GDT表和其描述符已静态储存在引导区中，载入即可
```as
# Switch from real to protected mode, using a bootstrap GDT
    # and segment translation that makes virtual addresses
    # identical to physical addresses, so that the
    # effective memory map does not change during the switch.
    lgdt gdtdesc
```

Q3: 如何使能和进入保护模式

实模式和保护模式的使能信号在寄存器CR0的PE位（Protected Mode Enable，即第0位）。
```as
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
```

初始化保护模式下的数据段寄存器。
```as
    # Jump to next instruction, but in 32-bit code segment.
    # Switches processor into 32-bit mode.
    ljmp $PROT_MODE_CSEG, $protcseg

.code32                                             # Assemble for 32-bit mode
protcseg:
    # Set up the protected-mode data segment registers
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment
```

初始化堆栈指针。
```as
    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebp
    movl $start, %esp
```

至此，已完全进入保护模式。

## 练习4

Q1: bootloader如何读取硬盘扇区的？

1. 等待磁盘准备好
2. 发出读取扇区的命令
3. 等待磁盘准备好
4. 把磁盘扇区数据读到指定内存

```c
#define SECTSIZE        512
#define ELFHDR          ((struct elfhdr *)0x10000)      // scratch space

/* waitdisk - wait for disk ready */
static void
waitdisk(void) {
    while ((inb(0x1F7) & 0xC0) != 0x40)
        /* do nothing */;
}

/* readsect - read a single sector at @secno into @dst */
static void
readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    waitdisk();

    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0); // main disk
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);
}
```

Q2: bootloader是如何加载ELF格式的OS？

```c
/* bootmain - the entry of bootloader */
void
bootmain(void) {
    // read the 1st page off disk
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```

## 练习5

其结果如下图所示：
```bash
ebp:0x0000007b28 eip:0x0000100a63 args:0x00010094 0x00010094 0x00007b58 0x00100092
    kern/debug/kdebug.c:297: print_stackframe+21
ebp:0x0000007b38 eip:0x0000100d4f args:0x00000000 0x00000000 0x00000000 0x00007ba8
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x0000007b58 eip:0x0000100092 args:0x00000000 0x00007b80 0xffff0000 0x00007b84
    kern/init/init.c:48: grade_backtrace2+33
ebp:0x0000007b78 eip:0x00001000bc args:0x00000000 0xffff0000 0x00007ba4 0x00000029
    kern/init/init.c:53: grade_backtrace1+38
ebp:0x0000007b98 eip:0x00001000db args:0x00000000 0x00100000 0xffff0000 0x0000001d
    kern/init/init.c:58: grade_backtrace0+23
ebp:0x0000007bb8 eip:0x0000100101 args:0x001032dc 0x001032c0 0x0000130a 0x00000000
    kern/init/init.c:63: grade_backtrace+34
ebp:0x0000007be8 eip:0x0000100055 args:0x00000000 0x00000000 0x00000000 0x00007c4f
    kern/init/init.c:28: kern_init+84
ebp:0x0000007bf8 eip:0x0000007d72 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
    <unknow>: -- 0x00007d71 --
```

## 练习6

Q: 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

8字节。

第0~15位表示段偏移的低16位，第16~31位表示段偏移的高16位；第16~31位表示段选择子。

---

我的代码实现与标准实现的区别：依据[文档](https://chyyuu.gitbooks.io/ucore_os_docs/lab1/lab1_2_1_6_ex6.html)的要求，我将`T_SYSCALL`设在了用户态，而标准代码将`T_SWITCH_TOK`设在了用户态

## 总结

### 本次实验中涉及的知识点

- 机器启动的过程：从BIOS, bootloader 到 kernel
- makefile 语法
- qemu 环境配置
- gdb 的使用
- 实模式与保护模式
- 硬盘访问
- ELF(Executable and linking format)
- 函数堆栈
- 中断
- IDT (Interrupt descriptor table)

### 本次实验应涉及但未涉及的知识点

- 系统调用