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

之后使用 0x10 加载 ds, es, fs, gs 四个数据段寄存器

```assembly
	mov %ax,%ds
	mov %ax,%es
	mov %ax,%fs
	mov %ax,%gs
	lss stack_start,%esp
```

```assembly
	call setup_idt
	call setup_gdt
	movl $0x10,%eax		# reload all the segment registers
	mov %ax,%ds		# after changing gdt. CS was already
	mov %ax,%es		# reloaded in 'setup_gdt'
	mov %ax,%fs
	mov %ax,%gs
	lss stack_start,%esp
	xorl %eax,%eax
1:	incl %eax		# check that A20 really IS enabled
	movl %eax,0x000000	# loop forever if it isn't
	cmpl %eax,0x100000
	je 1b
```

