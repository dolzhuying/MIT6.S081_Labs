### Lab6 locks

**1.memory allocator**

任务：简单描述：在原本的 `kalloc()` 中，只有一个大锁，我们会维护一个 `freelist` 链表，如果有任何程序申请内存，都需要竞争 `freelist` 的锁，以修改 `freelist` 的内容，可以发现，不可能同时有多个核心去调用 `kalloc()` 函数以及 `kfree()` 函数，大大降低了内存分配的效率。经测试，可以发现这个大锁就是一个很大瓶颈（`kmem` 这个锁是所有锁中等待次数最多，竞争最激烈的），需要解决性能瓶颈的问题，，，基本思想是每个 CPU 维护一个空闲列表，每个列表都有自己的锁。不同 CPU 上的分配和释放可以并行运行，因为每个 CPU 将在不同的列表上操作。主要的挑战是处理一个 CPU 的空闲列表为空，但另一个 CPU 的列表有空闲内存的情况；在这种情况下，一个 CPU 必须“窃取”另一个 CPU 空闲列表的一部分。窃取可能会引发锁争用提示：

- 可以使用`kernel/param.h` 中的 常量`NCPU`
- 让`freerange将所有可用内存提供给运行``freerange 的`CPU 。
- 函数`cpuid`返回当前核心编号，但只有在关闭中断的情况下调用它并使用其结果才是安全的。您应该使用 `push_off()`和`pop_off()`来关闭和打开中断。
- 请查看`kernel/sprintf.c `中的`snprintf`函数，了解字符串格式化的思路。不过，将所有锁都命名为“`kmem`”也是可以的

修改`kalloc.c`

```C
//修改结构体，为每个cpu维护一个空闲页链表和锁
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem[NCPU];


const uint name_size=sizeof("kmem cpu 0");
char kmem_lk_n[NCPU][sizeof("kmem cpu 0")];

//kinit():对每个cpu初始化
void
kinit()
{
  for(int i=0;i<NCPU;i++){
    snprintf(kmem_lk_n[i],name_size,"kmem cpu %d",i);
    initlock(&kmem[i].lock,kmem_lk_n[i]);
  }
  freerange(end, (void*)PHYSTOP);
}


//kfree：使用cpuid()来获取当前cpu编号（注意pushoff(),popoff()调整中断允许来使用cpuid()），释放当前cpu的页
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;
  push_off();
  uint cpu=cpuid();

  acquire(&kmem[cpu].lock);
  r->next = kmem[cpu].freelist;
  kmem[cpu].freelist = r;
  release(&kmem[cpu].lock);
  pop_off();
}

//kalloc：首先判断当前cpu的空闲页链表是否有空，有的话正常分配，没有的话需要从其他有空闲页的cpu来获取，由于最后分配使用的是r，因此可以直接使用r来指向有空闲页的cpu的页，注意遍历时需要获取该cpu的lock，同时由于需要遍历时，说明当前cpu已经没有空闲页，其他cpu访问时也不会进入逻辑，因此不需要提前释放该cpu的锁
void *
kalloc(void)
{
  struct run *r;
  push_off();
  uint cpu=cpuid();
  acquire(&kmem[cpu].lock);
  r=kmem[cpu].freelist;
  if(r){
    kmem[cpu].freelist=r->next;
  }else{
    for(int i=0;i<NCPU;i++){
      if(i==cpu)continue;
      if(kmem[i].freelist){
        acquire(&kmem[i].lock);
        r=kmem[i].freelist;
        kmem[i].freelist=r->next;
        release(&kmem[i].lock);
        break;
      }
    }
  }
  release(&kmem[cpu].lock);
  pop_off();

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```



**2.buffer cache**

说明：如果多个进程密集使用文件系统，它们可能会争用`bcache.lock`，该锁用于保护 kernel/bio.c 中的磁盘块缓存。bcachetest `创建`了几个进程，这些进程重复读取不同的文件以生成对`bcache.lock`的争用，

任务：修改块缓存，使得运行`bcachetest时，bcache 中所有锁的``获取`循环迭代次数接近于零。理想情况下，块缓存中涉及的所有锁的计数总和应为零，但如果总和小于 500 也是可以的。修改`bget` 和`brelse`，使得 bcache 中不同块的并发查找和释放不太可能在锁上发生冲突（例如，不必全部等待 `bcache.lock`）。您必须保持不变，即每个块最多缓存一个副本

提示：

- 可以使用固定数量的 bucket，并且不动态调整哈希表的大小。使用素数的 bucket（例如 13）可以降低哈希冲突的可能性。
- 在哈希表中搜索缓冲区并在未找到缓冲区时为该缓冲区分配一个条目必须是原子的。
- 删除所有缓冲区的列表（`bcache.head`等），改为使用缓冲区上次使用的时间（即使用kernel/trap.c 中的`ticks ）对缓冲区进行时间戳记。通过此更改， ``brelse`不需要获取 bcache 锁，而`bget`可以根据时间戳记选择最近最少使用的块。
- `在bget`中序列化驱逐是可以的（即，当在缓存中查找失败时， `bget`的部分会选择一个缓冲区来重新使用）。
- 在某些情况下，您的解决方案可能需要持有两个锁；例如，在驱逐期间，您可能需要持有 bcache 锁和每个 bucket 的锁。确保避免死锁。
- 替换块时，您可能会将`struct buf`从一个存储桶移动到另一个存储桶，因为新块的哈希值会移至另一个存储桶。您可能会遇到一种棘手的情况：新块的哈希值可能会与旧块移至同一个存储桶。在这种情况下，请确保避免死锁。
- 一些调试技巧：实现 bucket lock，但将全局 bcache.lock 获取/释放保留在 bget 的开始/结束处以序列化代码。一旦您确定它是正确的，没有竞争条件，请删除全局锁并处理并发问题。您还可以运行`make CPUS=1 qemu`以使用一个核心进行测试

参考：https://blog.miigon.net/posts/s081-lab8-locks/

```C
//bio.c

#define NBUFMAP_BUCKET 13
#define BUFMAP_HASH(dev, blockno) ((((dev)<<27)|(blockno))%NBUFMAP_BUCKET)

struct {
  struct spinlock eviction_lock;
  struct buf buf[NBUF];

  struct spinlock bufmap_locks[NBUFMAP_BUCKET];
  struct buf bufmap[NBUFMAP_BUCKET];
} bcache;

// 初始化缓冲区缓存。
void
binit(void)
{
  for(int i=0;i<NBUFMAP_BUCKET;i++){
    initlock(&bcache.bufmap_locks[i],"bcache_bufmap");
    bcache.bufmap[i].next=0;
  }

  for(int i=0;i<NBUF;i++){
    struct buf *b = &bcache.buf[i];
    initsleeplock(&b->lock, "buffer");
    b->lastuse = 0;
    b->refcnt = 0;
    // put all the buffers into bufmap[0]
    b->next = bcache.bufmap[0].next;
    bcache.bufmap[0].next = b;
  }

  initlock(&bcache.eviction_lock, "bcache_eviction");
  
}

// Look through buffer cache for block on device dev.
// If not found, allocate a buffer.
// In either case, return locked buffer.
// 获取指定设备号和块号的缓冲区
// 如果缓冲区已缓存，则增加引用计数并返回。
// 如果未缓存，则回收最近最少使用的未使用缓冲区，并初始化其字段。
// 返回锁定的缓冲区。
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  uint key = BUFMAP_HASH(dev, blockno);

  acquire(&bcache.bufmap_locks[key]);

  // Is the block already cached?
  for(b = bcache.bufmap[key].next; b; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.bufmap_locks[key]);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // 为了获取一个可重用的缓存块，我们需要在所有的 bucket 中搜索，
// 这意味着需要获取它们的 bucket 锁。
// 但在持有一个 bucket 锁的情况下尝试获取所有 bucket 锁是不安全的。
// 这样很容易导致循环等待，从而产生死锁。

release(&bcache.bufmap_locks[key]);
// 我们需要释放当前的 bucket 锁，这样在遍历所有 bucket 时不会导致循环等待和死锁。
// 然而，释放 bucket 锁会带来一个副作用：其他 CPU 可能会同时请求相同的 blockno，
// 在最坏的情况下，可能会多次创建相同的缓存块。
// 因为多个并发的 bget 请求可能会同时通过“该块是否已被缓存？”的检测，
// 并且都开始执行驱逐（eviction）和重用（reuse）过程。

// 因此，在获取 eviction_lock 之后，我们需要再次检查“该 blockno 的缓存是否已存在”，
// 以确保不会创建重复的缓存块。
acquire(&bcache.eviction_lock);

// 再次检查，该块是否已被缓存？
// 在持有 eviction_lock 时，不会有其他的驱逐和重用操作发生，
// 这意味着任何 bucket 的链表结构都不会发生变化。
// 因此，在这里遍历 bcache.bufmap[key] 时，不需要持有对应的 bucket 锁，
// 因为我们持有的是更强的 eviction_lock。

for(b = bcache.bufmap[key].next; b; b = b->next){
  if(b->dev == dev && b->blockno == blockno){
    acquire(&bcache.bufmap_locks[key]); // must do, for `refcnt++`
    b->refcnt++;
    release(&bcache.bufmap_locks[key]);
    release(&bcache.eviction_lock);
    acquiresleep(&b->lock);
    return b;
  }
}

struct buf *before_least = 0; 
uint holding_bucket = -1;
for(int i = 0; i < NBUFMAP_BUCKET; i++){
  // before acquiring, we are either holding nothing, or only holding locks of
  // buckets that are *on the left side* of the current bucket
  // so no circular wait can ever happen here. (safe from deadlock)
  acquire(&bcache.bufmap_locks[i]);
  int newfound = 0; // new least-recently-used buf found in this bucket
  for(b = &bcache.bufmap[i]; b->next; b = b->next) {
    if(b->next->refcnt == 0 && (!before_least || b->next->lastuse < before_least->next->lastuse)) {
      before_least = b;
      newfound = 1;
    }
  }
  if(!newfound) {
    release(&bcache.bufmap_locks[i]);
  } else {
    if(holding_bucket != -1) release(&bcache.bufmap_locks[holding_bucket]);
    holding_bucket = i;
    // keep holding this bucket's lock....
  }
}
if(!before_least) {
  panic("bget: no buffers");
}
b = before_least->next;

if(holding_bucket != key) {
  // remove the buf from it's original bucket
  before_least->next = b->next;
  release(&bcache.bufmap_locks[holding_bucket]);
  // rehash and add it to the target bucket
  acquire(&bcache.bufmap_locks[key]);
  b->next = bcache.bufmap[key].next;
  bcache.bufmap[key].next = b;
}

b->dev = dev;
b->blockno = blockno;
b->refcnt = 1;
b->valid = 0;
release(&bcache.bufmap_locks[key]);
release(&bcache.eviction_lock);
acquiresleep(&b->lock);
return b;

}

// Return a locked buf with the contents of the indicated block.
//获取一个包含指定磁盘块内容的锁定缓冲区
// 调用 bget 获取缓冲区。
// 如果缓冲区内容无效，则从磁盘读取数据并标记为有效。
// 返回缓冲区。
struct buf*
bread(uint dev, uint blockno)
{
  struct buf *b;

  b = bget(dev, blockno);
  if(!b->valid) {
    virtio_disk_rw(b, 0);
    b->valid = 1;
  }
  return b;
}

// Write b's contents to disk.  Must be locked.
//将缓冲区内容写回磁盘
void
bwrite(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("bwrite");
  virtio_disk_rw(b, 1);
}

// Release a locked buffer.
// Move to the head of the most-recently-used list.
//释放锁定的缓冲区
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  uint key=BUFMAP_HASH(b->dev, b->blockno);
  acquire(&bcache.bufmap_locks[key]);
  b->refcnt--;
  if(b->refcnt==0){

    b->lastuse=ticks;
  }
  release(&bcache.bufmap_locks[key]);
}

void
bpin(struct buf *b) {
  uint key = BUFMAP_HASH(b->dev, b->blockno);

  acquire(&bcache.bufmap_locks[key]);
  b->refcnt++;
  release(&bcache.bufmap_locks[key]);
}

void
bunpin(struct buf *b) {
  uint key = BUFMAP_HASH(b->dev, b->blockno);

  acquire(&bcache.bufmap_locks[key]);
  b->refcnt--;
  release(&bcache.bufmap_locks[key]);
}
```

