## Lab4 Traps

源码阅读：

```C
#kernel/trampoline.S
uservec:    
	#
        # trap.c sets stvec to point here, so
        # traps from user space start here,
        # in supervisor mode, but with a
        # user page table.
        #
        # sscratch points to where the process's p->trapframe is
        # mapped into user space, at TRAPFRAME.
        #
        
//	# swap a0 and sscratch
      //  # so that a0 is TRAPFRAME
        csrrw a0, sscratch, a0

     //   # save the user registers in TRAPFRAME
        sd ra, 40(a0)
        sd sp, 48(a0)
        sd gp, 56(a0)
        sd tp, 64(a0)
        sd t0, 72(a0)
        sd t1, 80(a0)
        sd t2, 88(a0)
        sd s0, 96(a0)
        sd s1, 104(a0)
        sd a1, 120(a0)
        sd a2, 128(a0)
        sd a3, 136(a0)
        sd a4, 144(a0)
        sd a5, 152(a0)
        sd a6, 160(a0)
        sd a7, 168(a0)
        sd s2, 176(a0)
        sd s3, 184(a0)
        sd s4, 192(a0)
        sd s5, 200(a0)
        sd s6, 208(a0)
        sd s7, 216(a0)
        sd s8, 224(a0)
        sd s9, 232(a0)
        sd s10, 240(a0)
        sd s11, 248(a0)
        sd t3, 256(a0)
        sd t4, 264(a0)
        sd t5, 272(a0)
        sd t6, 280(a0)

//	# save the user a0 in p->trapframe->a0
        csrr t0, sscratch
        sd t0, 112(a0)

       // # restore kernel stack pointer from p->trapframe->kernel_sp
        ld sp, 8(a0)

       // # make tp hold the current hartid, from p->trapframe->kernel_hartid
        ld tp, 32(a0)

       // # load the address of usertrap(), p->trapframe->kernel_trap
        ld t0, 16(a0)

       // # restore kernel page table from p->trapframe->kernel_satp
        ld t1, 0(a0)
        csrw satp, t1
        sfence.vma zero, zero

        # a0 is no longer valid, since the kernel page
        # table does not specially map p->tf.

       // # jump to usertrap(), which does not return
        jr t0
       
// 用户空间和内核空间的切换，调用ecall后，`CPU`中的`mode`位更新为`supervisor`，，`pc`中的值会切换为`stvec`的内容（原本的值保存到`sepc`），执行trampoline.S中的uservec，在用户态访问trapframe并保存32个用户态寄存器的值，获取内核栈指针、内核页表地址、当前cpu核的id、usertrap（）的地址，，在该汇编函数中跳转到usertrap，对发生trap的情况进行处理：

//kernel/trap.c
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){//如果发生trap的原因是系统调用
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;//ecall下一条指令

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();//执行系统调用
  } else if((which_dev = devintr()) != 0){//devintr 0表示未知，1表示外部设备中断，2表示计时器中断
    // ok
  } else {//未知原因，杀死进程
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();//返回到用户空间
}

// return to user space
//
void
usertrapret(void)
{
  struct proc *p = myproc();

  // we're about to switch the destination of traps from
  // kerneltrap() to usertrap(), so turn off interrupts until
  // we're back in user space, where usertrap() is correct.
  intr_off();

  // send syscalls, interrupts, and exceptions to trampoline.S
  w_stvec(TRAMPOLINE + (uservec - trampoline));

  // set up trapframe values that uservec will need when
  // the process next re-enters the kernel.
  p->trapframe->kernel_satp = r_satp();         // kernel page table
  p->trapframe->kernel_sp = p->kstack + PGSIZE; // process's kernel stack
  p->trapframe->kernel_trap = (uint64)usertrap;
  p->trapframe->kernel_hartid = r_tp();         // hartid for cpuid()

  // set up the registers that trampoline.S's sret will use
  // to get to user space.
  
  // set S Previous Privilege mode to User.
  unsigned long x = r_sstatus();
  x &= ~SSTATUS_SPP; // clear SPP to 0 for user mode
  x |= SSTATUS_SPIE; // enable interrupts in user mode
  w_sstatus(x);

  // set S Exception Program Counter to the saved user pc.
  w_sepc(p->trapframe->epc);

  // tell trampoline.S the user page table to switch to.
  uint64 satp = MAKE_SATP(p->pagetable);

  // jump to trampoline.S at the top of memory, which 
  // switches to the user page table, restores user registers,
  // and switches to user mode with sret.
  uint64 fn = TRAMPOLINE + (userret - trampoline);
  ((void (*)(uint64,uint64))fn)(TRAPFRAME, satp);
}
```



**1.RISC-V assembly (easy)**

阅读 call.asm 中函数`g`、`f`和`main`的代码，回答问题：

```asm
void main(void) {
  1c:	1141                	addi	sp,sp,-16
  1e:	e406                	sd	ra,8(sp)
  20:	e022                	sd	s0,0(sp)
  22:	0800                	addi	s0,sp,16
  printf("%d %d\n", f(8)+1, 13);
  24:	4635                	li	a2,13
  26:	45b1                	li	a1,12
  28:	00000517          	auipc	a0,0x0
  2c:	7b050513          	addi	a0,a0,1968 # 7d8 <malloc+0xea>
  30:	00000097          	auipc	ra,0x0
  34:	600080e7          	jalr	1536(ra) # 630 <printf>
  exit(0);
  38:	4501                	li	a0,0
  3a:	00000097          	auipc	ra,0x0
  3e:	27e080e7          	jalr	638(ra) # 2b8 <exit>

```

```
Q:哪些寄存器包含函数的参数？例如，在 main 调用printf时，哪个寄存器保存 13 ？
A：a2 ( 24:	4635                	li	a2,13)

Q：在 main 的汇编代码中，对函数f 的调用在哪里？对g 的 调用在哪里？（提示：编译器可能会内联函数。）
A：内联后无函数调用

Q：函数printf位于什么地址？
A：0x630   (34:	600080e7          	jalr	1536(ra) # 630 <printf>)

Q:在main中的jalr到printf 之后， 寄存器ra中的值是什么？
A:0x630  ( 3a:	00000097          	auipc	ra,0x0)

Q:运行以下代码:
unsigned int i = 0x00646c72;
	printf("H%x Wo%s", 57616, &i);
输出是什么？ 这是将字节映射到字符的 ASCII 表。
输出取决于 RISC-V 是小端字节序的事实。如果 RISC-V 是大端字节序，你会设置什么i来产生相同的输出？你需要更改 57616为不同的值吗？
A:输出"He110,World",大端序：i=0x726c6400,57616不变

Q：在下面的代码中，后面会打印什么 'y='？（注意：答案不是一个具体的值。）为什么会发生这种情况？
	printf("x=%dy=%d", 3);
A："x=3y=<随机值>"  未定义行为，可能读取栈上的旧数据，可能读取寄存器的随机值，也可能访问非法地址而直接崩溃

```



**2.backtrace(moderate)**

任务：对于调试来说，进行回溯通常很有用：在错误发生点上方的堆栈上的函数调用列表，在kernel/printf.c`中 实现`backtrace()`函数。在`sys_sleep`中插入对此函数的调用，然后运行，调用`sys_sleep编译器在每个堆栈帧中放入一个帧指针，该指针保存调用者的帧指针的地址。您的`回溯` 应该使用这些帧指针遍历堆栈并在每个堆栈帧中打印保存的返回地址，

首先在`kernel/defs.h`中添加声明：

```C
// printf.c
void            printf(char*, ...);
void            panic(char*) __attribute__((noreturn));
void            printfinit(void);
void            backtrace(void);
```

在`kernel/riscv.h`中添加函数（GCC 编译器将当前正在执行函数的帧指针存储在寄存器`s0中`，在`backtrace` 中调用此函数读取当前帧指针。此函数使用内联汇编读取`s0`）:

```C
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x) );
  return x;
}

```

然后在`kernel/printf.c`中实现`backtrace`,实现对栈帧的回溯，输出返回地址：

```C
void backtrace(){
  printf("backtrace:\n");
  uint64 fp=r_fp();//获取当前帧指针
  uint64 page_start=PGROUNDDOWN(fp);//栈帧所在页的边界
  uint64 page_end=PGROUNDUP(fp);
  while(page_start<=fp&&fp<=page_end){
    uint64 ra=*(uint64*)(fp-8);//当前栈帧的返回地址，注意是栈帧里存储了返回地址，fp-8只是栈帧中ra的内存地址，解引用才是ra的值
    fp=*(uint64*)(fp-16);//旧栈帧的指针
    printf("%p\n",ra);
  }
}
```

在`kernel/sysproc.c`中的`sys_sleep`的实现中添加对`backtrace`的调用：

```C
uint64
sys_sleep(void)
{
  int n;
  uint ticks0;

  if(argint(0, &n) < 0)
    return -1;
  acquire(&tickslock);
  ticks0 = ticks;
  while(ticks - ticks0 < n){
    if(myproc()->killed){
      release(&tickslock);
      return -1;
    }
    sleep(&ticks, &tickslock);
  }
  release(&tickslock);
  backtrace();//这里

  return 0;
}
```

编译并使用`bttest`命令，输出：

```
backtrace:
0x0000000080002144
0x0000000080001fa6
0x0000000080001c90
0x0000000000000012

```



**3.alarm(hard)**

`test0`

观察测试代码(`user/alarmtest.c`)：

```C
volatile static int count;

void
periodic()
{
  count = count + 1;
  printf("alarm!\n");
  sigreturn();
}

// tests whether the kernel calls
// the alarm handler even a single time.
void
test0()
{
  int i;
  printf("test0 start\n");
  count = 0;
  sigalarm(2, periodic);
  for(i = 0; i < 1000*500000; i++){
    if((i % 1000000) == 0)
      write(2, ".", 1);
    if(count > 0)
      break;
  }
  sigalarm(0, 0);
  if(count > 0){
    printf("test0 passed\n");
  } else {
    printf("\ntest0 failed: the kernel never called the alarm handler\n");
  }
}
```

可以看到周期为2，时钟数达到2之后执行`periodic`将`count++`，则成功，而每执行一次系统调用，时钟数+1，

添加系统调用的过程略，直接实现系统调用：

`kernel/sysproc.c`

```C
uint64 sys_sigalarm(void){//获取用户态传递的参数（周期数和处理函数指针）
  int times;
  uint64 handler;
  if(argint(0,&times)<0){
    return -1;
  }
  if(argaddr(1,&handler)<0){
    return -1;
  }
  struct proc*p=myproc();
  p->handler=handler;
  p->times=times;
  return 0;
}

//test0
 uint64 sys_sigreturn(void){
   return 0;
 }

```

`kernel/proc.h`添加如下变量

```C
 int times;//周期
 uint64 handler;//处理函数地址
 int time_passed;//时钟数
```

在`kernel/trap.c`中修改实现，使得在每次执行系统调用后时钟数+1，并判断是否需要修改`trapframe epc`指向处理函数：

```C
if(which_dev==2){
    p->time_passed+=1;
    if(p->time_passed%p->times==0){
      p->trapframe->epc=p->handler;
    }
    yield();
  }
```

`test1 2`

观察测试代码：

```C
void __attribute__ ((noinline)) foo(int i, int *j) {
  if((i % 2500000) == 0) {
    write(2, ".", 1);
  }
  *j += 1;
}

//
// tests that the kernel calls the handler multiple times.
//
// tests that, when the handler returns, it returns to
// the point in the program where the timer interrupt
// occurred, with all registers holding the same values they
// held when the interrupt occurred.
//
void
test1()
{
  int i;
  int j;

  printf("test1 start\n");
  count = 0;
  j = 0;
  sigalarm(2, periodic);
  for(i = 0; i < 500000000; i++){
    if(count >= 10)
      break;
    foo(i, &j);
  }
  if(count < 10){
    printf("\ntest1 failed: too few calls to the handler\n");
  } else if(i != j){
    // the loop should have called foo() i times, and foo() should
    // have incremented j once per call, so j should equal i.
    // once possible source of errors is that the handler may
    // return somewhere other than where the timer interrupt
    // occurred; another is that that registers may not be
    // restored correctly, causing i or j or the address ofj
    // to get an incorrect value.
    printf("\ntest1 failed: foo() executed fewer times than it was called\n");
  } else {
    printf("test1 passed\n");
  }
}

//
// tests that kernel does not allow reentrant alarm calls.
void
test2()
{
  int i;
  int pid;
  int status;

  printf("test2 start\n");
  if ((pid = fork()) < 0) {
    printf("test2: fork failed\n");
  }
  if (pid == 0) {
    count = 0;
    sigalarm(2, slow_handler);
    for(i = 0; i < 1000*500000; i++){
      if((i % 1000000) == 0)
        write(2, ".", 1);
      if(count > 0)
        break;
    }
    if (count == 0) {
      printf("\ntest2 failed: alarm not called\n");
      exit(1);
    }
    exit(0);
  }
  wait(&status);
  if (status == 0) {
    printf("test2 passed\n");
  }
}

void
slow_handler()
{
  count++;
  printf("alarm!\n");
  if (count > 1) {
    printf("test2 failed: alarm handler called more than once\n");
    exit(1);
  }
  for (int i = 0; i < 1000*500000; i++) {
    asm volatile("nop"); // avoid compiler optimizing away loop
  }
  sigalarm(0, 0);
  sigreturn();
}

```

可以知道`test1`和0并无太大差别，只是需要执行十次`periodic`，也就是执行二十次系统调用，触发，但是`test2`期望`slow_handler`只被执行一次

给`proc.h`增加字段：

```C
  // test 1/2 begin
  /*  40 */ uint64 ra;
  /*  48 */ uint64 sp;
  /*  56 */ uint64 gp;
  /*  64 */ uint64 tp;
  /*  72 */ uint64 t0;
  /*  80 */ uint64 t1;
  /*  88 */ uint64 t2;
  /*  96 */ uint64 s0;
  /* 104 */ uint64 s1;
  /* 112 */ uint64 a0;
  /* 120 */ uint64 a1;
  /* 128 */ uint64 a2;
  /* 136 */ uint64 a3;
  /* 144 */ uint64 a4;
  /* 152 */ uint64 a5;
  /* 160 */ uint64 a6;
  /* 168 */ uint64 a7;
  /* 176 */ uint64 s2;
  /* 184 */ uint64 s3;
  /* 192 */ uint64 s4;
  /* 200 */ uint64 s5;
  /* 208 */ uint64 s6;
  /* 216 */ uint64 s7;
  /* 224 */ uint64 s8;
  /* 232 */ uint64 s9;
  /* 240 */ uint64 s10;
  /* 248 */ uint64 s11;
  /* 256 */ uint64 t3;
  /* 264 */ uint64 t4;
  /* 272 */ uint64 t5;
  /* 280 */ uint64 t6;
  // saved user program counter
  uint64 epc;
  // 判断是否在handler
  int in_handler;
```

修改`sigreturn`的实现：

```C
//test1 2
uint64 sys_sigreturn(void) {
  struct proc *p = myproc();
  // 恢复状态
  p->trapframe->ra = p->ra;
  p->trapframe->sp = p->sp;
  p->trapframe->gp = p->gp;
  p->trapframe->tp = p->tp;
  p->trapframe->t0 = p->t0;
  p->trapframe->t1 = p->t1;
  p->trapframe->t2 = p->t2;
  p->trapframe->s0 = p->s0;
  p->trapframe->s1 = p->s1;
  p->trapframe->a0 = p->a0;
  p->trapframe->a1 = p->a1;
  p->trapframe->a2 = p->a2;
  p->trapframe->a3 = p->a3;
  p->trapframe->a4 = p->a4;
  p->trapframe->a5 = p->a5;
  p->trapframe->a6 = p->a6;
  p->trapframe->a7 = p->a7;
  p->trapframe->s2 = p->s2;
  p->trapframe->s3 = p->s3;
  p->trapframe->s4 = p->s4;
  p->trapframe->s5 = p->s5;
  p->trapframe->s6 = p->s6;
  p->trapframe->s7 = p->s7;
  p->trapframe->s8 = p->s8;
  p->trapframe->s9 = p->s9;
  p->trapframe->s10 = p->s10;
  p->trapframe->s11 = p->s11;
  p->trapframe->t3 = p->t3;
  p->trapframe->t4 = p->t4;
  p->trapframe->t5 = p->t5;
  p->trapframe->t6 = p->t6;
  // 恢复计数器
  p->trapframe->epc = p->epc;

  // 退出handler
  p->in_handler = 0;

  return 0;
}
```

修改`trap.c`中的`usertrap`逻辑：

```C
//test1 2
  if((which_dev == 2) && (p->in_handler == 0)) {
    p->time_passed += 1;
    // 防止p->ticks = 0
    if (p->times && (p->time_passed == p->times)) {
      // 恢复计数器
      p->time_passed = 0;
      p->in_handler = 1;
      // 保存状态
      p->ra = p->trapframe->ra;
      p->sp = p->trapframe->sp;
      p->gp = p->trapframe->gp;
      p->tp = p->trapframe->tp;
      p->t0 = p->trapframe->t0;
      p->t1 = p->trapframe->t1;
      p->t2 = p->trapframe->t2;
      p->s0 = p->trapframe->s0;
      p->s1 = p->trapframe->s1;
      p->a0 = p->trapframe->a0;
      p->a1 = p->trapframe->a1;
      p->a2 = p->trapframe->a2;
      p->a3 = p->trapframe->a3;
      p->a4 = p->trapframe->a4;
      p->a5 = p->trapframe->a5;
      p->a6 = p->trapframe->a6;
      p->a7 = p->trapframe->a7;
      p->s2 = p->trapframe->s2;
      p->s3 = p->trapframe->s3;
      p->s4 = p->trapframe->s4;
      p->s5 = p->trapframe->s5;
      p->s6 = p->trapframe->s6;
      p->s7 = p->trapframe->s7;
      p->s8 = p->trapframe->s8;
      p->s9 = p->trapframe->s9;
      p->s10 = p->trapframe->s10;
      p->s11 = p->trapframe->s11;
      p->t3 = p->trapframe->t3;
      p->t4 = p->trapframe->t4;
      p->t5 = p->trapframe->t5;
      p->t6 = p->trapframe->t6;
      // epc
      p->epc = p->trapframe->epc;
      // change pc
      p->trapframe->epc = p->handler;
      // p->in_handler = 0;
    } else {

    }

    yield();
  }
```

最后输出：

```
test1 start
.......alarm!
........alarm!
......alarm!
...alarm!
.......alarm!
...alarm!
.......alarm!
......alarm!
.......alarm!
...alarm!
test1 passed
test2 start
.......................................alarm!
test2 passed
```



