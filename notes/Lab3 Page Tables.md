## Lab3 Page Tables

在进行任务之前，首先来阅读一下相关的源代码：

```C
// kernel/kalloc.c

struct run {
  struct run *next;
};

struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;
/*kmem 是一个内核内存分配器的结构，lock 是它的一个互斥锁。为了确保多个内核线程安全地访问内存分配器（即空闲内存列表 freelist），
需要先获取这个锁。这样可以防止多个线程同时修改 freelist 导致数据竞争或不一致*/

//释放页的物理内存
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)//PHYSTOP是内核物理地址的上限，end是下限，同时检查是否是页对齐地址
    panic("kfree");

  // Fill with junk to catch dangling refs.
  /*这行代码用 1 填充传入的页。这样做的目的是帮助捕获悬挂指针（dangling pointers）
  如果内存释放后仍然被访问，程序会访问到一个无效的内存区域，并且由于内存已被填充为 1，
  程序可以发现非法访问，这在调试时非常有用，能够及早发现错误*/
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;
  /*填充垃圾数据为了防止悬挂指针引用已经释放的内存，同时方便调试，但同时他也是有效的可分配的内存块，因此插入空闲页链表等待下次分配数据*/

  acquire(&kmem.lock);//获取kmem的自旋锁，防止线程竞争资源（关于锁相关的知识以后补充）
  r->next = kmem.freelist;//当前页的下一个节点指向空闲页链表头部，
  kmem.freelist = r;//空闲页链表头指针指向当前页，这两步将当前页插入空闲页链表中
  release(&kmem.lock);
}

//分配页的物理内存
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;//获取空闲页链表的头部
  if(r)
    kmem.freelist = r->next;//头部指针指向下一个页
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;//返回获取到的页地址，即取出空闲页链表的头指针
}
```

```C
// kernel/vm.c

//用于为页表项和物理地址创建映射关系
int
mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)//pagetable当前进程的页表，perm访问权限
{
  uint64 a, last;
  pte_t *pte;

  if(size == 0)
    panic("mappages: size");
  
  a = PGROUNDDOWN(va);//当前虚拟地址所在页的起始地址
  last = PGROUNDDOWN(va + size - 1);//末尾页的起始地址
  for(;;){
    if((pte = walk(pagetable, a, 1)) == 0)//页表项指针，如果该地址尚未映射过（没有对应页表项）返回NULL（返回的是页表项地址而不是物理地址）
        /*walk函数，返回当前虚拟地址在最低级页表对应的页表项
        pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA)
    panic("walk");

  for(int level = 2; level > 0; level--) {
    pte_t *pte = &pagetable[PX(level, va)];
    if(*pte & PTE_V) {
      pagetable = (pagetable_t)PTE2PA(*pte);
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  return &pagetable[PX(0, va)];
}*/
      return -1;
    if(*pte & PTE_V)
    /*检查页表项中的 PTE_V 位（有效位），如果该位为 1，
    表示该虚拟地址已经被映射到物理地址。如果已经映射过，再次映射同一地址是非法的*/
      panic("mappages: remap");
      
    *pte = PA2PTE(pa) | perm | PTE_V;//将物理地址 pa 转化成页表项设置到页表中，设置va，pa对应关系，并设置相应权限和有效位
    /*将 PA2PTE(pa)、perm 和 PTE_V 三个部分按位 OR 操作，
    最终得到页表项的完整值，表示虚拟地址 a 映射到物理地址 pa，并且具有相应的权限*/
    if(a == last)//检查是否已经映射完所有页
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}

//从虚拟地址va开始释放n个pages的物理内存，对于每个page必须已经存在
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)//未找到页表项
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)//页表项未映射
      panic("uvmunmap: not mapped");
    if(PTE_FLAGS(*pte) == PTE_V)//非叶子节点，即非映射到实际物理地址的页表项，只是中间页表项对下一级页表索引
      panic("uvmunmap: not a leaf");
    if(do_free){//dofree指示是否需要在解除映射后释放物理内存，如果为 1，则会释放相应的物理页面；
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}


```



**1.Speed up system calls (easy)**

任务：创建每个进程时，在 `USYSCALL`（在`memlayout.h` 中定义的 VA）处映射一个只读页面。在此页面的开头，存储一个`struct usyscall`（也在`memlayout.h`中定义），并将其初始化为存储当前进程的 `PID`。对于此实验，`ugetpid()`已在用户空间端提供，并将自动使用 `USYSCALL `映射

首先来观察一下题目中给出的`USYSCALL`和`usyscall`的定义：

```C
// kernel/memlayout.h

#define USYSCALL (TRAPFRAME - PGSIZE)
struct usyscall {
  int pid;  // Process ID
};
//可以看出这里USYSCALL是一个地址，usyscall是一个结构体，存储了进程id，由于需要和当前进程绑定，因此其应该为proc结构体的一部分：

// kernel/proc.h
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
  struct usyscall*usyscall;    //////////////////////////////////这里
};
```

要了解怎么怎么完成任务，需要先了解用户测试程序`pgtbltest`的测试原理：

```C
// user/pgtbltest.c
void
ugetpid_test()
{
  int i;

  printf("ugetpid_test starting\n");
  testname = "ugetpid_test";

  for (i = 0; i < 64; i++) {
    int ret = fork();
    if (ret != 0) {
      wait(&ret);
      if (ret != 0)
        exit(1);
      continue;
    }
    if (getpid() != ugetpid())
      err("missmatched PID");
    exit(0);
  }
  printf("ugetpid_test: OK\n");
}

//可以看到核心测试代码是 if (getpid() != ugetpid())，查看ugetpid（）定义：

// ulib.c
int
ugetpid(void)
{
  struct usyscall *u = (struct usyscall *)USYSCALL;
  return u->pid;
}
//ugetpid通过访问USYSCALL地址访问usyscall结构体实例，获取进程pid，，可见该地址对于用户程序可见，
```

接下来是对于进程创建时该变量初始化、该变量所在物理地址和USYSCALL的虚拟地址的映射和进程结束时的资源释放：

```C
// kernel/proc.c

//该函数用于寻找进程表中处于UNUSED的进程并初始化状态以在内核中运行
static struct proc*
allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {//遍历进程表
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;//转移到初始化部分
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();//设置pid和state
  p->state = USED;

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  //给usyscall分配一页内存，如果分配失败则释放当前进程，注意kalloc分配的是物理内存地址
  if((p->usyscall=(struct usyscall*)kalloc())==0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  p->usyscall->pid=p->pid;

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}

//该函数用于创建用户进程的页表
pagetable_t
proc_pagetable(struct proc *p)
{
  pagetable_t pagetable;

  // An empty page table.
  pagetable = uvmcreate();
  if(pagetable == 0)
    return 0;

  // map the trampoline code (for system call return)
  // at the highest user virtual address.
  // only the supervisor uses it, on the way
  // to/from user space, so not PTE_U.
  if(mappages(pagetable, TRAMPOLINE, PGSIZE,
              (uint64)trampoline, PTE_R | PTE_X) < 0){
    uvmfree(pagetable, 0);
    return 0;
  }

  // map the trapframe just below TRAMPOLINE, for trampoline.S.
  if(mappages(pagetable, TRAPFRAME, PGSIZE,
              (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }
  
  //执行usyscall映射，mappages函数已经在前文提及介绍，这里将进程的usyscall的物理地址和USYSCALL虚拟地址映射，如果失败则释放之前已经映射过的页的物理内存
  if(mappages(pagetable,USYSCALL,PGSIZE,(uint64)(p->usyscall),PTE_R|PTE_U)<0){
    uvmunmap(pagetable,TRAMPOLINE,1,0);
    uvmunmap(pagetable,TRAPFRAME,1,0);
    uvmfree(pagetable,0);
    return 0;
  }

  return pagetable;
}

//释放进程的物理内存并重置进程状态
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->usyscall)//这里
    kfree((void*)p->usyscall);
  p->usyscall=0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}

//删除页表虚拟地址和物理地址的映射，但不释放内存（do_free位为0）
void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable,USYSCALL,1,0);
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmfree(pagetable, sz);
}
```



**2.Print a page table (easy）**

任务：`定义一个名为vmprint()` 的函数。它应该接受一个`pagetable_t`参数，并以下面描述的格式打印该页表：

```
page table 0x0000000087f6e000
 ..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
 .. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
 .. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
 .. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
 .. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
 ..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
 .. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
 .. .. ..509: pte 0x0000000021fdd813 pa 0x0000000087f76000
 .. .. ..510: pte 0x0000000021fddc07 pa 0x0000000087f77000
 .. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
```

显然，函数需要递归到三级页表进行打印

```C
// kernel/defs.h
//vm.c
void             vmprint(pagetable_t,uint64);//声明函数，在vm.c中定义
```

观察`riscv.h`中的声明：

```C
typedef uint64 pte_t;
typedef uint64 *pagetable_t; // 512 PTEs
//这里pte_t是对uint64的别名  pagetable_t是对uint64指针的别名
```

```C
// kernel/vm.c

// Recursively free page-table pages.
// All leaf mappings must already have been removed.
void
freewalk(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
      pagetable[i] = 0;
    } else if(pte & PTE_V){
      panic("freewalk: leaf");
    }
  }
  kfree((void*)pagetable);
}

//大体结构与freewalk相似，参考riscv.h和
void 
vmprint(pagetable_t pagetable,uint64 dep){//接收参数：pagetable_t为uint64的别名，参数为页表的物理地址，dep为当前页表的层级
  if(dep==0){
    printf("page table %p",pagetable);
  }
  if(dep>2)return;//已经超过第三层页表，返回
   for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];//获取页表项（pte_t是对uint64的别名）
    if(pte&PTE_V){//只对有效的页表项进行操作
      printf(".. ");
      for(int j=0;j<dep;j++){
        printf(".. ");//根据层级来打印..
      }
      printf("%d:pte %p pa %p",i,pte,PTE2PA(pte));//打印虚拟地址和物理地址，这里PTE2PA是宏定义，用于将页表项映射为物理地址

      /*当页表项没有设置任何读、写或执行权限时，通常意味着该页表项并不是指向一个实际的物理内存页，而是指向下一级的页表的指针*/
      if((pte & (PTE_R|PTE_W|PTE_X)) == 0){ // 如果页表项没有设置读/写/执行权限
      uint64 child = PTE2PA(pte);//获取页表项指向的物理地址
      vmprint((pagetable_t)child,dep+1);//遍历下一层页表
    }
    }
  }
  kfree((void*)pagetable);// 释放当前页表的内存
}

```

在`exec.c`的`return argc`前加上`if(p->pid==1) vmprint(p->pagetable,0)`即可

* **3.Detecting which pages have been accessed (hard）**

任务：您的任务是实现`pgaccess()`，这是一个报告已访问过哪些页面的系统调用。该系统调用需要三个参数。首先，它需要检查的第一个用户页面的起始虚拟地址。其次，它需要检查的页面数。最后，它需要缓冲区的用户地址，以将结果存储到位掩码（一种每页使用一位的数据结构，其中第一页对应于最低有效位）中

对于标记是否已经使用过该页的标志位`PTE_A`，查找资料得知：为6，因此，在`riscv`中定义：

```C
// kernel/riscv.h
#define PTE_A (1L<<6)
```

查看一下测试程序：

```C
// user/pgtbltest.c
void
pgaccess_test()
{
  char *buf;
  unsigned int abits;
  printf("pgaccess_test starting\n");
  testname = "pgaccess_test";
  buf = malloc(32 * PGSIZE);
  if (pgaccess(buf, 32, &abits) < 0)
    err("pgaccess failed");
  buf[PGSIZE * 1] += 1;
  buf[PGSIZE * 2] += 1;
  buf[PGSIZE * 30] += 1;
  if (pgaccess(buf, 32, &abits) < 0)
    err("pgaccess failed");
  if (abits != ((1 << 1) | (1 << 2) | (1 << 30)))
    err("incorrect access bits set");
  free(buf);
  printf("pgaccess_test: OK\n");
}
```

可以看到调用了`pgacess`系统调用的参数有三个，传入的指针，页数的大小，掩码的地址，由于用户态和内核态不共享页表，需要使用`agrint agraddr`获取参数

然后是实现`sys_pgacess()`

```C
// kernel/sysproc.c
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc);//由于需要用到vm.c中的walk函数，需要在此提前声明以链接
#ifdef LAB_PGTBL
int
sys_pgaccess(void)
{
  //lab pgtbl: your code here.
  uint64 addr;//传入的起始地址
  int npages;//页数
  uint64 mask;//用户进程的掩码地址

  //获取用户进程参数
  if(argaddr(0,&addr)<0){
    return -1;
  }
  if(argint(1,&npages)<0){
    return -1;
  }
  if(argaddr(2,&mask)){
    return -1;
  }

  struct proc*p=myproc();
  uint64 now_bitmask=0;//准备传递给用户进程的掩码
  for(int i=0;i<npages;i++){//遍历从起始地址开始的每一个页
    pte_t*pte=walk(p->pagetable,addr+i*PGSIZE,0);//获取当前页的页表项
    if((*pte&PTE_A)!=0){//如果已经使用过，即满足条件
      now_bitmask|=(1<<i);//对应位置的掩码位置为1
      *pte&=(~PTE_A);//访问过之后清除PTE_A置为0，即标记访问过
    }
  }

  if(copyout(p->pagetable,mask,(char*)&now_bitmask,sizeof(now_bitmask))<0){//将设置好的掩码传递给用户进程
    return -1;
  }

  return 0;
}
#endif
```







