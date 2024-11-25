## Lab0：utils

**1.boot xv6（easy）**

配置环境，略



**2.sleep（easy）**

任务：为 xv6实现 UNIX 程序`sleep`；您的`sleep`应该暂停用户指定的滴答数。滴答是 xv6 内核定义的时间概念，即定时器芯片两次中断之间的时间

sleep原本有同名的系统调用提供的接口，该任务将系统调用二次封装成可在QEMU执行的用户程序

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

//sleep本身有系统调用的接口，在这里进行二次封装成一个可由用户在终端执行的用户程序
int main(int argc,char*argv[]){
    if(argc!=2){//判断参数数量
        fprintf(2,"usage:sleep time");
        exit(1);
    }
    int n=atoi(argv[1]);//转成整数，获取休眠时间
    sleep(n);//sleep系统调用
    exit(0);
}
```

需要注意的是头文件引用间可能有依赖，因此在这里顺序不能变



**3.pingpong（easy）**

任务：使用 UNIX 系统调用通过一对管道在两个进程之间`pingpong`一个字节，每个方向一个。父进程应向子进程发送一个字节；子进程应打印`<pid>: received ping`，其中 `<pid>` 是其进程 ID，将管道上的字节写入父进程，然后退出；父进程应从子进程读取该字节，打印`<pid>: received pong`，然后退出

管道是一种进程通信（IPC）的机制，用于有亲缘关系的进程之间数据传输，它是一个内核缓冲区，本质上是一个文件，用两个文件描述符引用读写端口，因此管道的通信机制是通过文件系统实现的。父进程创建了管道，然后创建的子进程会继承父进程的文件描述符，因此父子进程可以共用该管道通信

补充：每个进程都有独立的用户地址空间，因此一个进程的全局变量对其他进程来说完全不可见，进程间通信必须通过内核交换数据

举例：在`shell中`执行命令，经常会将上一个命令的输出作为下一个命令的输入，由多个命令配合完成一件事情。而这就是通过管道来实现的。`|`这个竖线就是管道符号    e.g.

```
ls -l | grep string
```

- ls命令（其实也是一个进程）会把当前目录中的文件都列出来
- 但它不会直接输出，而是把要输出到屏幕上的数据通过管道输出到`grep`这个进程中，作为`grep这个进程的输入`；
- 然后这个进程对输入的信息进行筛选`（grep)`，把存在`string`的信息的字符串（以行为单位）打印在屏幕上。

管道的特点：

* 管道采用先进先出（FIFO）机制，数据读取与写入顺序相同，同一个管道只能同时一边写入一边读取（管道由环形队列实现），因此管道只支持单向通信，但读写端口对于使用该管道的两个进程都是开放的

* 在有亲缘关系的进程间传递信息：进程有共同的祖先，，只要祖先进程调用了`pipe`，打开的管道文件就能被后代共享

* 管道的内容在`read`之后就会从管道中移除，不会保存

* 管道是字节流通信，没有消息边界，多个进程同时发送的字节流混在一起，则无法分辨消息，所有管道一般用于2个进程之间通信

管道使用更详细细节见注释（不完整，例如死锁等未学习，后面补充）：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc,char*argv[]){
    if(argc!=1){
        fprintf(2,"usage:pingpong\n");
        exit(1);
    }
    int p1[2],p2[2];//管道文件描述符,分别用于父传子和子传父
    //创建管道并处理失败情况
    if(pipe(p1)<0){
        fprintf(2,"pipe create failed\n");
        exit(1);
    }
    if(pipe(p2)<0){
        fprintf(2,"pipe create failed\n");
        exit(1);
    }
    char buf[10];//从管道读取出数据的缓冲区
    int pid=fork();
    if(pid==0){//子进程
        read(p1[0],buf,10);//先从父进程读取数据，在没有读取到父进程写入的数据之前会处于阻塞状态
        //子进程p1管道用于读取数据，不需要写入，读取完成后子进程不再使用该管道，关闭管道读写端口释放资源，防止资源泄露和不必要的阻塞
        //具体来讲，当子进程完成数据读取后，关闭管道的写端口可以通知父进程没有更多的数据会被写入
        //关闭写端口后，父进程在尝试读取管道数据时，如果管道已经空并且写端被关闭，读取操作会返回0，表示对端已关闭。这是一个明确的信号，指示父进程不再需要继续读取
        close(p1[0]);
        close(p1[1]);
        printf("%d:received ping\n",getpid());
        write(p2[1],"pong",4);
        close(p2[0]);
        close(p2[1]);
    }else if(pid>0){//父进程
        write(p1[1],"ping",4);
        close(p1[0]);
        close(p1[1]);
        read(p2[0],buf,10);
        close(p2[0]);
        close(p2[1]);
        printf("%d:received pong\n",getpid());
    }
    exit(0);


}
```



**4.find（moderate）**

任务：编写一个简单版本的 UNIX find 程序：查找目录树中具有特定名称的所有文件

在完成该任务之前，需要先学习`ls`的实现：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

char*
fmtname(char *path)
{
  static char buf[DIRSIZ+1];
  char *p;

  for(p=path+strlen(path); p >= path && *p != '/'; p--)
    ;
  p++;//e.g. ./a/b/ccc 指向ccc的第一个c

  if(strlen(p) >= DIRSIZ)
    return p;
  memmove(buf, p, strlen(p));//将ccc的第一个c移动到buf开头
  memset(buf+strlen(p), ' ', DIRSIZ-strlen(p));//ccc后面的buf剩余部分以空格填充
  return buf;
}

void
ls(char *path)
{
  char buf[512], *p;
  int fd;
  struct dirent de;
  /* fs.h中的定义
  #define DIRSIZ 14  文件长度最大值

  struct dirent {
    ushort inum;  存储文件的inode号：文件系统内部用来标识该文件的唯一编号
    char name[DIRSIZ]; 文件名称
  };

  */
  struct stat st;
  /* ulib.c中的定义
  int
  stat(const char *n, struct stat *st)
  {
    int fd;
    int r;

    fd = open(n, O_RDONLY); 只读模式打开
    if(fd < 0)
      return -1;
    r = fstat(fd, st); 存储文件信息
    close(fd);
    return r;  fstat成功返回0 失败-1
  }
  */

  if((fd = open(path, 0)) < 0){//使用 open 系统调用来打开一个目录时，返回的文件描述符 fd 指向的是该目录的数据结构
    fprintf(2, "ls: cannot open %s\n", path);
    return;
  }

  if(fstat(fd, &st) < 0){
    /*
    system call 用于存储打开的文件的信息存储到st
    int fstat(int fd, struct stat*);
    */
    fprintf(2, "ls: cannot stat %s\n", path);
    close(fd);
    return;
  }

  switch(st.type){
  case T_FILE:
    printf("%s %d %d %l\n", fmtname(path), st.type, st.ino, st.size);//文件则输出
    break;

  case T_DIR:
    if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){//检查构建完整路径后的长度是否超过buf的大小限制
      printf("ls: path too long\n");
      break;
    }
    strcpy(buf, path);
    p = buf+strlen(buf);
    *p++ = '/';//假设目录是aaa =>./aaa/ 指向末尾/

    //对于每个有效的目录条目（即inum不为0），构建完整路径并使用stat获取文件状态信息
    //从目录中读取一个目录项，并将其存储在 de 结构体中。
    while(read(fd, &de, sizeof(de)) == sizeof(de)){
      if(de.inum == 0)
        continue;
      memmove(p, de.name, DIRSIZ);//拼接文件名到路径
      p[DIRSIZ] = 0;
      if(stat(buf, &st) < 0){//buf指向文件完整路径开头
        printf("ls: cannot stat %s\n", buf);
        continue;
      }
      printf("%s %d %d %d\n", fmtname(buf), st.type, st.ino, st.size);
    }
    break;
  }
  close(fd);
}

int
main(int argc, char *argv[])
{
  int i;

  if(argc < 2){
    ls(".");
    exit(0);
  }
  for(i=1; i<argc; i++)
    ls(argv[i]);
  exit(0);
}

```

其中出现的结构体：

`stat`：存储文件系统中文件或目录的信息，具体成员如下：

* `dev`:文件所在的物理设备编号

* `ino`：文件在文件系统中的唯一标识

* `type`：文件类型，`T-DIR`目录，`T-FILE`：文件，`T-DEVICE`：设备文件

* `nlink`：指向这个文件的硬链接数量，一个文件可以有多个名称（硬链接），`nlink` 存储了这些链接的数量。当这个数量降为 0 时，文件的实际数据才会被释放（即当所有链接都被删除时）。

* `size`：文件大小，字节为单位

`de`：存储文件名称及文件唯一标识

在了解了`ls`的实现原理之后，`find`的实现也就简单了，对`ls`的代码进行一下改写即可：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

void find(char*path,char*filename);

int main(int argc,char*argv[]){
    if(argc!=3){
        fprintf(2,"usage:find root_dir target_fileanme");
        exit(1);
    }
    find(argv[1],argv[2]);//起始目录及目标文件名
    exit(0);

}

void find(char*path,char*target){
    char *p,buf[1024];
    int fd;
    struct dirent de;
    struct stat st;
    
    if((fd=open(path,0))<0){
      fprintf(2,"open failed\n");
      exit(1);
    }
    if(fstat(fd,&st)<0){//将当前path信息存储到st中
      fprintf(2,"fstat error\n");
      close(fd);
      exit(1);
    }
    if(strlen(path)+1+DIRSIZ+1>sizeof(buf)){
      fprintf(2,"path too long\n");
      close(fd);
      exit(1);
    }
    while(read(fd, &de, sizeof(de)) == sizeof(de)){//对于目录，读取目录下存储文件的数据结构
      if(de.inum==0||!strcmp(de.name,".")||!strcmp(de.name,".."))continue;//避免不存在或损坏文件，及. ..的循环递归
      strcpy(buf,path);
      p=buf+strlen(buf);
      *p++='/';//./aa/bb/vv/ 指向最后的/
      memmove(p,de.name,DIRSIZ);
      p[DIRSIZ]=0;
      
      if(stat(path,&st)<0)continue;
      switch(st.type){
        case T_FILE:
          if(!strcmp(de.name,target)){//符合条件则输出
            printf("%s\n",path);
          }
          break;
        case T_DIR:
          find(buf,target);//遇到目录则递归
          break;
      }
    }
    close(fd);
}
```



**5.xargs（moderate）**

任务：编写 UNIX xargs 程序的简单版本：从标准输入读取行并对每行运行一个命令，并将行作为参数提供给命令

`xargs`介绍：以命令 `echo hello too | xargs echo bye`为例，`|`左半边的命令将标准输出`hello too`写入管道，`xargs`从管道读取标准输入，并将其作为参数执行自己的命令，该命令输出`bye hello too`

```c
#include "kernel/param.h"
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

// 创建子进程执行命令
void execargs(char *argv[], char *args[MAXARG]) {
    int pid = fork();
    if (pid < 0) {
        fprintf(2, "fork error\n");
        exit(-1);
    } else if (pid == 0) {
        exec(argv[1], args);
        exit(1);
    } else {
        wait(0);
    }
}

//在这里，argv为xargs开始的参数
int main(int argc, char *argv[]) {
     if (argc < 2) {
        fprintf(2, "usage: xargs (command args)\n");
        exit(-1);
    }
    if (argc + 1 > MAXARG) {
        fprintf(2, "too many args\n");
        exit(-1);
    }
    char *args[MAXARG];//存储参数
    char buf[1024];
    for(int i=1;i<argc;i++){
        args[i-1]=argv[i]; //最后一位 args[argc-1] 留给从标准输入读取的参数
    }
    while(1){
        int arg_size=0;
        while(1){//循环逐个字符读取来自管道的标准输入的参数，以换行符为分界，逐个字符读取可保证一定能读取完，也可以避免buf溢出的问题
            int n=read(0,&buf[arg_size],1);
            if(buf[arg_size]=='\n'||n==0)break;
            arg_size++;
        }
        if(arg_size==0)break;//全部读取完
        buf[arg_size]=0;//尾部\0
        args[argc-1]=buf;////最后一位 args[argc-1] 留给从标准输入读取的参数
        execargs(argv,args);//创建子进程并传递参数执行指令
    }
    exit(0);
}


```



**6.primes（moderate/hard）**

任务：使用`pipe`和`fork`来设置管道。第一个进程将数字 2 到 35 输入管道。对于每个素数，安排创建一个进程，该进程通过管道从其左侧邻居读取，并通过另一个管道写入其右侧邻居。由于 xv6 的文件描述符和进程数量有限，因此第一个进程可以在 35 处停止。

注：小心关闭进程不需要的文件描述符，否则在第一个进程达到 35 之前，程序就会耗尽 xv6 的资源；一旦第一个进程达到 35，它就应该等到整个管道终止，包括所有子进程、孙进程等。因此，主 primes 进程应该在所有输出都已打印并且所有其他 primes 进程都已退出后才退出

首先需要了解一下埃氏筛基本思想：从2开始，将每个质数的倍数都标记成合数，以达到筛选素数的目的，因为随便一个合数的约数都不会大于自己，且必然存在有约数是素数的情况，那么我对规定范围内的数进行从小到大的判断，正好是能“划掉大的合数”且不会出现遗漏。

```C
const int N = 1e8 + 10;
int st[N], primes[N];
void Ey(int n){//埃氏筛
	st[0], st[1] = 1;
	for (int i = 2; i <= n / i; i++){
		if (!st[i]){
			for (int j = i * i; j <= n; j += i){
				st[j] = 1;
			}
		}
	}
}
```

因此对于本任务，只需从第二个进程开始，每个进程先读取从上一个进程传递的第一个数字（即为质数）并打印，然后对后面读取的所有数字进行筛选，不是当前质数的倍数的就向下一个进程传递即可

```C
#include"kernel/types.h"
#include"user/user.h"

void pass_primes(int*pf);

int main(int argc,int *argv[]){
    if(argc!=1){
        fprintf(2,"usage:prime\n");
        exit(1);
    }
    //pipe_latter，当前进程与下一个进程通信的管道
    int pl[2];
    pipe(pl);
    int pid=fork();
    if(pid>0){
        close(pl[0]);//关闭pipelatter读端
        for(int i=2;i<=35;i++){
            write(pl[1],&i,sizeof(i));//向第一个子进程传递所有数字
        }
        close(pl[1]);//关闭写端
        wait(0);//等待子进程结束
        exit(0);
    }
    else if(pid==0){
        pass_primes(pl);
        exit(0);
    }
    else{
        close(pl[0]);
        close(pl[1]);
        exit(1);
    }
}

void pass_primes(int*pf){//pipe_former，当前进程与上一个进程通信的管道
    close(pf[1]);//关闭pipeformer的写端
    int prime;
    int n=read(pf[0],&prime,sizeof(prime));//读取读取第一个数字，必定是素数
    if(n==0){//已经全部读取完毕，返回
        close(pf[0]);
        return;
    }
    printf("prime %d\n",prime);
    int pl[2];//当前进程的pipelatter
    pipe(pl);
    int pid=fork();
    int read_num,t;
    if(pid>0){
        close(pl[0]);//当前进程为父进程，关闭pipelatter的读端
        while(read(pf[0],&read_num,4)==4)){//逐个读取父进程传递的数字，是素数则向下一个进程传递(32位int)
            if(read_num%prime){
                t=read_num;
                write(pl[1],&t,sizeof(t));
            }
        }
        close(pf[0]);//读取完毕，关闭pipeformer的读端
        close(pl[1]);//写入完毕，关闭pipelatter写端
        wait(0);
        exit(0);
    }
    else if(pid==0){
        pass_primes(pl);//递归。子进程递归为下一阶段的当前进程
        exit(0);
    }
    else{//创建进程失败
        close(pf[0]);
        close(pl[0]);
        close(pl[1]);
        exit(1);
    }
}
```

在编写的时候需要注意的是进程之间的管道的文件描述符的close，释放资源，，对于除去开头和结尾的任何一个进程，首先需要关闭与上一个进程通信的管道`pipe_former`的写端，创建进程向下一个进程传递数据时，首先关闭与下一个进程的管道`pipe_latter`的读端，在读取`pipe_former`传递的数据并向`pipe_latter`写入完成以后，再关闭`pipe_former`的读端和`pipe_latter`的写端

```
        pipe_former      |-----------|     pipe_latter
             ------------|           |-----------
            1         0  |-----------|  1        0
```

