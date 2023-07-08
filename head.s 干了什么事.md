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

先来看 setup_idt，

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



```assembly
	movl $0x10,%eax		# reload all the segment registers
	mov %ax,%ds		# after changing gdt. CS was already
	mov %ax,%es		# reloaded in 'setup_gdt'
	mov %ax,%fs
	mov %ax,%gs
	lss stack_start,%esp
```

