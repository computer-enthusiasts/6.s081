桩代码:

# System call tracing (moderate)

​	首先在 proc.h 中修改 proc 结构的定义，添加 syscall_trace field，用 mask 的方式记录要 trace 的 system call。

```c
// kernel/proc.h
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
  uint64 syscall_trace;        // Mask for syscall tracing (新添加的用于标识追踪哪些 system call 的 mask)
};
```

在 proc.c 中，创建新进程的时候，为新添加的 syscall_trace 附上默认值 0（否则初始状态下可能会有垃圾数据）。

```c
// kernel/proc.c
static struct proc*
allocproc(void)
{
  ......

  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  p->syscall_trace = 0; // (newly added) 为 syscall_trace 设置一个 0 的默认值

  return p;
}
```

在 sysproc.c 中，实现 system call 的具体代码，也就是设置当前进程的 syscall_trace mask：

```c
// kernel/sysproc.c
uint64
sys_trace(void)
{
  int mask;

  if(argint(0, &mask) < 0) // 通过读取进程的 trapframe，获得 mask 参数
    return -1;
  
  myproc()->syscall_trace = mask; // 设置调用进程的 syscall_trace mask
  return 0;
}
```

修改 fork 函数，使得子进程可以继承父进程的 syscall_trace mask：

```c
// kernel/proc.c
int
fork(void)
{
  ......

  safestrcpy(np->name, p->name, sizeof(p->name));

  np->syscall_trace = p->syscall_trace; // HERE!!! 子进程继承父进程的 syscall_trace

  pid = np->pid;

  np->state = RUNNABLE;

  release(&np->lock);

  return pid;
}
```

根据上方提到的系统调用的全流程，可以知道，所有的系统调用到达内核态后，都会进入到 syscall() 这个函数进行处理，所以要跟踪所有的内核函数，只需要在 syscall() 函数里埋点就行了。

```c
// kernel/syscall.c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) { // 如果系统调用编号有效
    p->trapframe->a0 = syscalls[num](); // 通过系统调用编号，获取系统调用处理函数的指针，调用并将返回值存到用户进程的 a0 寄存器中
	// 如果当前进程设置了对该编号系统调用的 trace，则打出 pid、系统调用名称和返回值。
    if((p->syscall_trace >> num) & 1) {
      printf("%d: syscall %s -> %d\n",p->pid, syscall_names[num], p->trapframe->a0); // syscall_names[num]: 从 syscall 编号到 syscall 名的映射表
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

上面打出日志的过程还需要知道系统调用的名称字符串，在这里定义一个字符串数组进行映射：

```c
// kernel/syscall.c
const char *syscall_names[] = {
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
[SYS_trace]   "trace",
};
```

编译执行：

```c
$ trace 32 grep hello README
3: syscall read -> 1023
3: syscall read -> 966
3: syscall read -> 70
3: syscall read -> 0
$
$ trace 2147483647 grep hello README
4: syscall trace -> 0
4: syscall exec -> 3
4: syscall open -> 3
4: syscall read -> 1023
4: syscall read -> 966
4: syscall read -> 70
4: syscall read -> 0
4: syscall close -> 0
$
```

成功追踪并打印出相应的系统调用。



# Sysinfo (moderate)



添加一个系统调用，返回空闲的内存、以及已创建的进程数量。大多数步骤和上个实验是一样的，所以不再描述。唯一不同就是需要把结构体从内核内存拷贝到用户进程内存中。其他的难点可能就是在如何获取空闲内存和如何获取已创建进程上面了，因为涉及到了一些后面的知识。

## 获取空闲内存

在内核的头文件中声明计算空闲内存的函数，因为是内存相关的，所以放在 kalloc、kfree 等函数的的声明之后。

```
// kernel/defs.h
void*           kalloc(void);
void            kfree(void *);
void            kinit(void);
uint64 			count_free_mem(void); // here
```

在 kalloc.c 中添加计算空闲内存的函数：

```c
// kernel/kalloc.c
uint64
count_free_mem(void) // added for counting free memory in bytes (lab2)
{
  acquire(&kmem.lock); // 必须先锁内存管理结构，防止竞态条件出现
  
  // 统计空闲页数，乘上页大小 PGSIZE 就是空闲的内存字节数
  uint64 mem_bytes = 0;
  struct run *r = kmem.freelist;
  while(r){
    mem_bytes += PGSIZE;
    r = r->next;
  }

  release(&kmem.lock);

  return mem_bytes;
}
```

xv6 中，空闲内存页的记录方式是，将空虚内存页**本身**直接用作链表节点，形成一个空闲页链表，每次需要分配，就把链表根部对应的页分配出去。每次需要回收，就把这个页作为新的根节点，把原来的 freelist 链表接到后面。注意这里是**直接使用空闲页本身**作为链表节点，所以不需要使用额外空间来存储空闲页链表，在 kalloc() 里也可以看到，分配内存的最后一个阶段，是直接将 freelist 的根节点地址（物理地址）返回出去了：

```c
// kernel/kalloc.c
// Allocate one 4096-byte page of physical memory.
// Returns a pointer that the kernel can use.
// Returns 0 if the memory cannot be allocated.
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist; // 获得空闲页链表的根节点
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r; // 把空闲页链表的根节点返回出去，作为内存页使用（长度是 4096）
}
```

**常见的记录空闲页的方法有：空闲表法、空闲链表法、位示图法（位图法）、成组链接法**。这里 xv6 采用的是空闲链表法。

## 获取运行的进程数

同样在内核的头文件中添加函数声明：

```c
// kernel/defs.h
......
void            sleep(void*, struct spinlock*);
void            userinit(void);
int             wait(uint64);
void            wakeup(void*);
void            yield(void);
int             either_copyout(int user_dst, uint64 dst, void *src, uint64 len);
int             either_copyin(void *dst, int user_src, uint64 src, uint64 len);
void            procdump(void);
uint64			count_process(void); // here
```

在 proc.c 中实现该函数：

```c
uint64
count_process(void) { // added function for counting used process slots (lab2)
  uint64 cnt = 0;
  for(struct proc *p = proc; p < &proc[NPROC]; p++) {
    // acquire(&p->lock);
    // 不需要锁进程 proc 结构，因为我们只需要读取进程列表，不需要写
    if(p->state != UNUSED) { // 不是 UNUSED 的进程位，就是已经分配的
        cnt++;
    }
  }
  return cnt;
}
```

## 实现 sysinfo 系统调用

添加系统调用的流程与实验 1 类似，不再赘述。

这是具体系统信息函数的实现，其中调用了前面实现的 count_free_mem() 和 count_process()：

```c
uint64
sys_sysinfo(void)
{
  // 从用户态读入一个指针，作为存放 sysinfo 结构的缓冲区
  uint64 addr;
  if(argaddr(0, &addr) < 0)
    return -1;
  
  struct sysinfo sinfo;
  sinfo.freemem = count_free_mem(); // kalloc.c
  sinfo.nproc = count_process(); // proc.c
  
  // 使用 copyout，结合当前进程的页表，获得进程传进来的指针（逻辑地址）对应的物理地址
  // 然后将 &sinfo 中的数据复制到该指针所指位置，供用户进程使用。
  if(copyout(myproc()->pagetable, addr, (char *)&sinfo, sizeof(sinfo)) < 0)
    return -1;
  return 0;
}
```

在 user.h 提供用户态入口：

```c
// user.h
char* sbrk(int);
int sleep(int);
int uptime(void);
int trace(int);
struct sysinfo; // 这里要声明一下 sysinfo 结构，供用户态使用。
int sysinfo(struct sysinfo *);
```

编译运行：

```c
$ sysinfotest
sysinfotest: start
sysinfotest: OK

```