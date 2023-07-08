## head.s 干了什么事

head.s 执行前的内存布局如下所示，此时 head.s 和操作系统剩余部分的代码被移动到物理地址 0 到 0x80000 处，0x90000 处存放着 setup.s 中获取的一些设备信息，过后要用到。0x90020 开始是 setup.s 的代码和数据，里面包括之前设置的临时 idt 和 gdt，此时 cpu 中的 idtr，gdtr 寄存器指向之前的临时 idt 和 gdt。

```c
        +------------------+
        |                  |
        |                  |
        +------------------+                 +-----------+
        |              idt <--------------+  |           |
        |  setup           |              |  | +-------+ |
        |              gdt <----------+   +----+ idtr  | |
        |                  |          |      | +-------+ |
0x90020 +------------------+          +--------+ gdtr  | |
        |                  |                 | +-------+ |
        |                  |                 |           |
        +------------------+                 +-----------+
        |临时存放的变量     |                      CPU
0x90000 +------------------+
        |                  |
0x80000 +------------------+
        |                  |
        |system(操作系统全 |
        |部代码)           |
        |                  |
      0 +------------------+

```

进入head.s

```assembly
pg_dir:
.globl startup_32
startup_32:
	movl $0x10,%eax
```

pg_dir是页目录，位于物理地址 0 处，后面的代码在之后会被页目录的内容覆盖，但是那时这段代码已经执行完毕，所以无所谓。首先把 0x10 赋值给 eax。0x10 对应的段选择符如下，TI = 0表示在 gdt 中，请求特权级为 0，gdt 中的第二个描述符对应的是之前 setup.s 中临时设置的数据段描述符。 

```c
 描述符索引=2     TI RPL
+-----------------------+
|0000 0000 0001 0| 0| 00|
+-----------------------+
```

之后使用 0x10 加载 ds, es, fs, gs 四个数据段寄存器，使用 stack_start 加载 ss 和 esp。

```assembly
	mov %ax,%ds
	mov %ax,%es
	mov %ax,%fs
	mov %ax,%gs
	lss stack_start,%esp
```

stack_start 定义在 sched.c 中，是一个48位的长指针。低4字节是 user_stack 的末端地址，高2字节 0x10 是数据段的段选择符。

```c
long user_stack [ PAGE_SIZE>>2 ] ;

struct {
	long * a;
	short b;
} stack_start = { & user_stack [PAGE_SIZE>>2] , 0x10 };
```

LSS 指令的作用是从源操作数指定的内存中加载一个32位或48位的长指针到 ss:目的操作数指定的寄存器中，在这里就是 ss:esp 中。

| Opcode         | Instruction    | Op/En | Description                               |
| -------------- | -------------- | ----- | ----------------------------------------- |
| 0F B2 /r       | LSS r16,m16:16 | RM    | Load SS:r16 with far pointer from memory. |
| 0F B2 /r       | LSS r32,m16:32 | RM    | Load SS:r32 with far pointer from memory. |
| REX + 0F B2 /r | LSS r64,m16:64 | RM    | Load SS:r64 with far pointer from memory. |

所以 lss stack_start,%esp 的作用就是将 stack_start 处的低32位，也就是 user_stack 的末端地址赋值给 esp，并将 stack_start 的高16字节，也就是数据段的段选择符赋值给 ss 寄存器。从而将 user_stack 设置为当前使用的栈。

接下来重新设置 idt 和 gdt。因为原先在 setup.s 中的 idt 和 gdt 之后会被缓冲区覆盖掉，所以肯定要把 idt 和 gdt 挪到 system 部分。

```assembly
	call setup_idt
	call setup_gdt
```

先来看 setup_idt，ignore_int 是个只输出错误信息的哑中断子程序，setup_idt 让 idt 中的256个中断描述符都指向 ignore_int。其中 2~5 行是将要设置的中断门内容填充到 eax 和 ebx 中，其中 eax 存放着低4字节，ebx 存放着高4字节。

```assembly
setup_idt:
	lea ignore_int,%edx
	movl $0x00080000,%eax
	movw %dx,%ax		/* selector = 0x0008 = cs */
	movw $0x8E00,%dx	/* interrupt gate - dpl=0, present */

	lea idt,%edi
	mov $256,%ecx
rp_sidt:
	movl %eax,(%edi)
	movl %edx,4(%edi)
	addl $8,%edi
	dec %ecx
	jne rp_sidt
	lidt idt_descr
	ret
```

中断门格式如下：

```c
31                             15 14 13 12       5 4     0
+-----------------------------+--+-----+----------+--------+
|     过程入口点偏移 31...16    | P| DPL | 01110000 |       |
+-----------------------------+--+-----+----------+--------+
31                         16 15                          0
+----------------------------+-----------------------------+
|         段选择符            |    过程入口点偏移 0...15     |
+----------------------------+-----------------------------+

```

2~5 行执行完后 eax 和 ebx 的内容如下，可以和上面的中断门格式对应上

```c
                             P DPL
     +----------------------+---------------------+
ebx: | ignore_int 地址高16位 |1| 00 |01110000|00000|
     +----------------------+---------------------+
          段选择符(代码段)
     +---------------------+----------------------+
eax: |      0x0008         | ignore_int 地址低16位 |
     +---------------------+----------------------+

```

然后让 edi 指向 idt 表第一项，循环256次，依次填充 idt 表的每一项。因为一个中断门8个字节，所以每次循环 edi + 8，指向表中下一项。最后 lidt 指令加载 idtr 寄存器。

setup_gdt 非常简单，直接用 lgdt 指令加载 gdtr 寄存器，因为 gdt 的内容已经硬编码好了，第一项是空描述符，第二项是代码段描述符，第三项是数据段描述符，后面还预留了 252 个空项，用于之后的 LDT 和 TSS。

```assembly
setup_gdt:
	lgdt gdt_descr
	ret
```

```assembly
.align 2
.word 0
idt_descr:
	.word 256*8-1		# idt contains 256 entries
	.long idt
.align 2
.word 0
gdt_descr:
	.word 256*8-1		# so does gdt (not that that's any
	.long gdt		    # magic number, but it works for me :^)

	.align 8
idt:	.fill 256,8,0		# idt is uninitialized

gdt:	.quad 0x0000000000000000	/* NULL descriptor */
	.quad 0x00c09a0000000fff	/* 16Mb */ /* 00000000 11000000 10011010 00000000 00000000 00000000 										   00001111 11111111 */
	.quad 0x00c0920000000fff	/* 16Mb */ /* 00000000 11000000 10010010 00000000 00000000 00000000 										   00001111 11111111 */
	.quad 0x0000000000000000	/* TEMPORARY - don't use */
	.fill 252,8,0			/* space for LDT's and TSS's etc */
```

在重新设置 gdt 后要重新加载各个段寄存器：

```assembly
	movl $0x10,%eax		# reload all the segment registers
	mov %ax,%ds		# after changing gdt. CS was already
	mov %ax,%es		# reloaded in 'setup_gdt'
	mov %ax,%fs
	mov %ax,%gs
	lss stack_start,%esp
```

