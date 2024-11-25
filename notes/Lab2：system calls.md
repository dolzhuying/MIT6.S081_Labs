## Lab2：system calls

**1.system call tracing（moderate）**

任务：创建一个新的`trace`系统调用来控制跟踪。它应该接受一个参数，即一个整数“掩码”，其位指定要跟踪哪些系统调用。例如，要跟踪 `fork` 系统调用，程序会调用`trace(1 << SYS_fork)`，其中`SYS_fork是``kernel/syscall.h`中的系统调用编号。如果系统调用的编号在掩码中设置，则必须修改 xv6 内核以在每个系统调用即将返回时打印出一行。该行应包含进程 ID、系统调用的名称和返回值；您不需要打印系统调用参数。`trace`系统调用应该为调用它的进程及其随后分叉的任何子进程启用跟踪，但不应影响其他进程。

分析：首先，在`riscv`中，用户程序进行系统调用的过程是，在用户态中使用了系统调用接口，会调用`ecall <n>`，其中n使用的的系统调用的参数，`ecall`指令会将程序从用户态转到内核态，执行`system call`，检查参数n，并执行对应的系统调用，，，

在本任务中，`trace`的用户程序已经封装好，但是`trace`的系统调用还未实现，因此首先需要为用户态空间添加`trace`系统调用接口：

```C
// user/user.h
// system calls
int fork(void);
int exit(int) __attribute__((noreturn));
int wait(int*);
int pipe(int*);
int write(int, const void*, int);
int read(int, void*, int);
int close(int);
int kill(int);
int exec(char*, char**);
int open(const char*, int);
int mknod(const char*, short, short);
int unlink(const char*);
int fstat(int fd, struct stat*);
int link(const char*, const char*);
int mkdir(const char*);
int chdir(const char*);
int dup(int);
int getpid(void);
char* sbrk(int);
int sleep(int);
int uptime(void);
int trace(int);//添加用户空间下的trace系统调用接口
```

在`user/usys.pl`脚本中，添加`trace`系统调用接口，该脚本定义了系统调用的存根，使得内核能够处理用户程序发起的系统调用（生成汇编文件`usys.S`,调用CPU提供的`ecall`指令转入内核态）

```plsql
sub entry {
    my $name = shift;
    print ".global $name\n";
    print "${name}:\n";
    print " li a7, SYS_${name}\n";
    print " ecall\n";
    print " ret\n";
}
	
entry("fork");
entry("exit");
entry("wait");
entry("pipe");
entry("read");
entry("write");
entry("close");
entry("kill");
entry("exec");
entry("open");
entry("mknod");
entry("unlink");
entry("fstat");
entry("link");
entry("mkdir");
entry("chdir");
entry("dup");
entry("getpid");
entry("sbrk");
entry("sleep");
entry("uptime");
entry("trace");#生成trace汇编代码存根
```

j接下来在内核态，首先需要定义`trace`系统调用的参数，在`kernel/syscall.h`中：

```C
// System call numbers
#define SYS_fork    1
#define SYS_exit    2
#define SYS_wait    3
#define SYS_pipe    4
#define SYS_read    5
#define SYS_kill    6
#define SYS_exec    7
#define SYS_fstat   8
#define SYS_chdir   9
#define SYS_dup    10
#define SYS_getpid 11
#define SYS_sbrk   12
#define SYS_sleep  13
#define SYS_uptime 14
#define SYS_open   15
#define SYS_write  16
#define SYS_mknod  17
#define SYS_unlink 18
#define SYS_link   19
#define SYS_mkdir  20
#define SYS_close  21
#define SYS_trace  22//trace系统调用参数

```

接着对进程状态结构体进行修改，添加一个变量`syscall_trace`用于跟踪是否使用了某系统调用的状态，这里可以理解为，该`syscall_trace`是一个二进制数，每一个位置是否为1代表了是否需要跟踪该位置对应的系统调用，（二进制状态压缩）：

```C
// kernel/proc.h

// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // wait_lock must be held when using this:
  struct proc *parent;         // Parent process

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
  uint64 syscall_trace;        // //////////////////////////////////////////////////////////////////trace mask
};
```

在进程创建时，需要对当前进程的`syscall_trace`设置为0，防止初始状态下的垃圾数据：

```C
// kernel/proc.c

//allocproc函数中：
p->syscall_trace=0;//设置默认值0
```

然后在`kernel/sysproc.c`（与进程相关的系统调用）中声明一个新函数`sys_trace`，该函数用于设置当前进程的`syscall_trace`为命令行传入的参数，由于内核与用户进程的页表不同，寄存器也不互通，所以参数无法直接通过 C 语言参数的形式传过来，而是需要使用 `argaddr、argint、argstr` 等系列函数，从进程的 `trapframe` 中读取用户进程寄存器中的参数,`trapframe`是一个数据结构，用于保存进程在发生异常或者系统调用时的上下文，确保进程能够恢复到原来的状态

```C
uint64
sys_trace(void){
  int mask;
  if(argint(0,&mask)<0){
   /*这里argint为另一个系统调用，用于将从用户空间的系统调用接口获取的参数绑定到mask上，定义如下：
   int
   argint(int n, int *ip)
   {
     *ip = argraw(n);
     return 0;
   } 
   其中argraw为另一个系统调用：
   static uint64
   argraw(int n)
   {
     struct proc *p = myproc();
     switch (n) {
     case 0:
       return p->trapframe->a0;
     case 1:
       return p->trapframe->a1;
     case 2:
       return p->trapframe->a2;
     case 3:
       return p->trapframe->a3;
     case 4:
       return p->trapframe->a4;
     case 5:
       return p->trapframe->a5;
     }
     panic("argraw");
     return -1;
   }
   这里a012345为寄存器，代表了该寄存器存储的数据， agrint中传入0代表选择当前进程中的a0中的数据绑定到mask，0作为参数索引，表示希望获取第一个参数
   这在xv6书的4.4节有提及：
   The kernel trap code saves user registers to the current process’s trap frame, where kernel code can find them. The kernel functions argint, argaddr,and argfd retrieve the n ’th system call argument from the trap frame as an integer, pointer, or a file descriptor. They all call argraw to retrieve the appropriate saved user register (kernel/syscall.c:35)
   */
    return -1;
  }
  myproc()->syscall_trace=mask;//为进程的syscall_trace设置为命令行传入的mask
  return 0;
}

```

接着为`fork`创建的子进程拷贝父进程的`syscall_trace`
```C
// kernel/proc.c

//fork函数中：
np->syscall_trace=p->syscall_trace;//子进程继承父进程的syscall_trace
```

然后在`kernel/syscall.c`中`extern`声明新的系统调用并加入到从系统调用参数到对应系统调用指针的映射表：

```C
extern uint64 sys_chdir(void);
extern uint64 sys_close(void);
extern uint64 sys_dup(void);
extern uint64 sys_exec(void);
extern uint64 sys_exit(void);
extern uint64 sys_fork(void);
extern uint64 sys_fstat(void);
extern uint64 sys_getpid(void);
extern uint64 sys_kill(void);
extern uint64 sys_link(void);
extern uint64 sys_mkdir(void);
extern uint64 sys_mknod(void);
extern uint64 sys_open(void);
extern uint64 sys_pipe(void);
extern uint64 sys_read(void);
extern uint64 sys_sbrk(void);
extern uint64 sys_sleep(void);
extern uint64 sys_unlink(void);
extern uint64 sys_wait(void);
extern uint64 sys_write(void);
extern uint64 sys_uptime(void);
extern uint64 sys_trace(void);//加入新的系统调用

static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
[SYS_exit]    sys_exit,
[SYS_wait]    sys_wait,
[SYS_pipe]    sys_pipe,
[SYS_read]    sys_read,
[SYS_kill]    sys_kill,
[SYS_exec]    sys_exec,
[SYS_fstat]   sys_fstat,
[SYS_chdir]   sys_chdir,
[SYS_dup]     sys_dup,
[SYS_getpid]  sys_getpid,
[SYS_sbrk]    sys_sbrk,
[SYS_sleep]   sys_sleep,
[SYS_uptime]  sys_uptime,
[SYS_open]    sys_open,
[SYS_write]   sys_write,
[SYS_mknod]   sys_mknod,
[SYS_unlink]  sys_unlink,
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,
[SYS_trace]   sys_trace//加入新的映射
};

//由于要打印系统待用名称，这里多定义一个系统调用参数和对应名称的映射表
const char*syscalls_names[]={
[SYS_fork]    "fork",
[SYS_exit]    "exit",
[SYS_wait]    "wait",
[SYS_pipe]    "pipe",
[SYS_read]    "read",
[SYS_kill]    "kill",
[SYS_exec]    "exec",
[SYS_fstat]   "fstat",
[SYS_chdir]   "chdir",
[SYS_dup]     "dup",
[SYS_getpid]  "getpid",
[SYS_sbrk]    "sbrk",
[SYS_sleep]   "sleep",
[SYS_uptime]  "uptime",
[SYS_open]    "open",
[SYS_write]   "write",
[SYS_mknod]   "mknod",
[SYS_unlink]  "unlink",
[SYS_link]    "link",
[SYS_mkdir]   "mkdir",
[SYS_close]   "close",
[SYS_trace]   "trace"
};
```

最后是`syscall`的修改，因为在进入内核态后所有系统调用都会到这里执行，由于要跟踪所有系统调用，只需在这里进行判断然后打印即可：

```C
// kernel/syscall.c

void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {//若系统调用编号有效
    p->trapframe->a0 = syscalls[num]();// 通过系统调用编号，获取系统调用处理函数的指针，调用并将返回值存到用户进程的 a0 寄存器中
    
    if((p->syscall_trace>>num)&1){//这里理解syscall_trace为需要查找的系统调用的二进制状态压缩，num为当前的系统调用编号
      printf("%d:syscall %s -> %d\n",p->pid,syscalls_names[num],p->trapframe->a0);
    }

  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}

```

至此我们可以总结出系统调用的全过程，假设某个在命令行写入的用户程序`xxx`，其中调用了一个系统调用接口`yyy()`（假设与进程相关），发生如下事件：

```
user/user.h  //其中定义了在用户态下的系统调用接口yyy
user/usys.pl //其中为yyy系统调用定义了存根，生成usys.S汇编文件调用ecall进入内核态
kernel.syscall.c //切换到内核态之后统一调用syscall()，所有的系统调用都会在这里处理，根据对应的的系统调用参数在syscall[]表中查询对应的系统调用并执行
kernel/sysproc.c //执行 sys_yyy()函数
```

该过程实现了用户态与内核态的强隔离，保证了用户程序的执行不会影响系统的运行

**2.sysinfo(moderate)**

任务：在此作业中，您将添加一个系统调用`sysinfo`，用于收集有关正在运行的系统的信息。该系统调用接受一个参数：指向 `struct sysinfo`的指针 （请参阅`kernel/sysinfo.h`）。内核应填写此结构的各个字段：`freemem`字段应设置为可用内存的字节数，`nproc字段应设置为状态`不是`UNUSED` 的进程数

该任务同样需要添加系统调用，前面步骤同上一个任务，在`user/user.h`中声明用户空间的系统调用接口（注意这里要多声明`sysinfo`结构体，因为该结构体也定义在内核），在`user/usys.pl`中添加`sysinfo`的存根，在`kernel/syscall.h`定义`sysinfo`的系统调用参数，在`kernel/syscall.c`的`syscalls[]`映射表添加相应表项，，，接下来是系统调用的实现

首先看看`sysinfo`结构体的定义

```C
struct sysinfo {
  uint64 freemem;   // amount of free memory (bytes)
  uint64 nproc;     // number of process
};
```

该结构体存储系统状态的信息，包括可用内存和非`unused`的进程数量

然后看一下用户程序里对该系统调用接口的访问

```C
// user/sysinfotest.c

void
sinfo(struct sysinfo *info) {
  if (sysinfo(info) < 0) {
    printf("FAIL: sysinfo failed");
    exit(1);
  }
}

void
testmem() {
  struct sysinfo info;
  uint64 n = countfree();
  
  sinfo(&info);//传入测试函数内定义的info结构体的地址
    。。。
}
```

可以看到用户程序中定义的`info`结构体的地址被作为参数传递给了`sysinfo`系统调用接口，又测试程序是在用户态下，因此需要在内核态获取的系统状态的`sysinfo info`结构体的信息绑定给用户态的`info`，系统调用实现如下：

```C
// kernel/sysproc.c

uint64 
sys_sysinfo(void){
  uint64 addr;
  if(argaddr(0,&addr)<0){//将用户程序中的sysinfo系统调用接口的info的地址绑定到addr，info指向的是用户进程空间中的地址
    return -1;
  }
  struct sysinfo sinfo;
  sinfo.freemem=count_left_memory();//获取系统可用内存
  sinfo.nproc=count_processes();//获取系统非unused的进程数量
  if(copyout(myproc()->pagetable,addr,(char*)&sinfo,sizeof(sinfo))<0){//将内核函数内定义的sinfo的信息拷贝到用户进程地址空间的addr
    return -1;
  }
  return 0;
}

```

（这里`copyout`用于将内核态的信息拷贝到用户地址空间页表的指定虚拟地址上，，，由于内核态和用户态进程的地址空间不同，内核态不能对用户态的指针进行解引用，用户态也不能获取内核态的指针，因此想要将内核态下获取的系统状态信息传递给用户进程，需要使用`copyout`，并结合进程的页表才能将内核态的`info`传递给用户进程的具体物理地址）：

```C
// Copy from kernel to user.
// Copy len bytes from src to virtual address dstva in a given page table.
// Return 0 on success, -1 on error.
int
copyout(pagetable_t pagetable, uint64 dstva/*目标虚拟地址*/, char *src/*内核态的数据传输起始地址*/, uint64 len/*要复制的字节数*/)
{
  uint64 n, va0, pa0;

  while(len > 0){//要复制的字节数
    va0 = PGROUNDDOWN(dstva);//目标虚拟地址的页起始地址
    pa0 = walkaddr(pagetable, va0);//虚拟地址对应的物理地址
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);//当前页剩余的字节数
    if(n > len)//剩余字节数大于要拷贝的数据则赋值
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);//将n字节数据复制到用户地址空间的dstva，这里的是经过计算的物理地址

    len -= n;//要拷贝字节数减去n
    src += n;//要传输的数据的起始地址偏移n
    dstva = va0 + PGSIZE;//下一页起始地址
  }
  return 0;
}
```

然后是`count_left_memory()`和`count_processes()`的实现，首先需要对这两个函数进行声明：

```C
// kernel/defs.h

// proc.c
。。。
uint64          count_processes(void);//计算进程数

// kalloc.c
。。。
uint64          count_left_memory(void);//计算系统剩余可用内存
```

对于`count_left_memory()`

```C
// kernel/kalloc.c

uint64 count_left_memory(void){//计算系统剩余内存量
  acquire(&kmem.lock);//先上锁，防止对freelist的竞争访问
  uint64 left_mem=0;
  struct run*r;
  r=kmem.freelist;//空闲页链表的根节点
  while(r){
    left_mem+=PGSIZE;//空闲页数量乘以pagesize
    r=r->next;
  }
  release(&kmem.lock);
  return left_mem;
}
```

需要注意的是，`xv6` 中，空闲内存页的记录方式是，将空虚内存页本身直接用作链表节点，形成一个空闲页链表，每次需要分配，就把链表根部对应的页分配出去。每次需要回收，就把这个页作为新的根节点，把原来的 `freelist `链表接到后面

对于`count_processes()`

```C
// kernel/proc.c

uint64 count_processes(void){
  uint64 pro_num=0;
  struct proc*p;
  for(p=proc;p<&proc[NPROC];p++){//遍历进程数组
    if(p->state==UNUSED){
      continue;
    }
    pro_num++;
  }
  return pro_num;
}

```







