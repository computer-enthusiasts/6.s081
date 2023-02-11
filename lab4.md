## Backtrace([moderate](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

在xv6-labs-2020/kernel/defs.h中添加：

```c
C// part 2 begin
void backtrace();
// part 2 end
```

修改learning/xv6-labs-2020/kernel/printf.c/panic：

```c
Cvoid
panic(char *s)
{
  pr.locking = 0;
  printf("panic: ");
  printf(s);
  printf("\n");
  // part 2 begin
  backtrace();
  // part 2 end
  panicked = 1; // freeze uart output from other CPUs
  for(;;)
    ;
}

// part 2 begin
void backtrace() {
  printf("backtrace\n");
  // 获得fp
  uint64 fp = r_fp();
  // start, end
  uint64 start = PGROUNDDOWN(fp);
  uint64 end = PGROUNDUP(fp);
  while((start <= fp) && (fp <= end)) {
    uint64 ra = *(uint64 *)(fp - 8);
    fp = *(uint64 *)(fp - 16);
    printf("%p\n", ra);
  }
}
// part 2 end
```



修改xv6-labs-2020/kernel/sysproc.c/sys_sleep：

```c
Cuint64
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

  // part 2 begin
  backtrace();
  // part 2 end
  return 0;
}
```

测试：

```
CODE./grade-lab-traps backtrace
== Test backtrace test == backtrace test: OK (1.5s) 
```



## Alarm([hard](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

修改Makefile：

```c
MAKEFILEUPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_rm\
	$U/_sh\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_alarmtest\
# part 3
```

修改proc.h：

```c
C// Per-process state
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
  // part 3 begin
  // test 0 begin
  // 周期
  int ticks;
  // handler地址
  uint64 handler;
  // 经过多久
  int ticks_pass;
  // test 0 end

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
  // test 1/2 end
  // part 3 end
};
```

修改syscall.h，syscall.c，user.h，user.pl来添加系统调用。

syscall.h：

```
C// part 3 begin
#define SYS_sigalarm  22
#define SYS_sigreturn  23
// part 3 end
```

syscall.c：

```c
C// part 3 begin
extern uint64 sys_sigalarm(void);
extern uint64 sys_sigreturn(void);
// part 3 end

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
// part 3 begin
[SYS_sigalarm]  sys_sigalarm,
[SYS_sigreturn]  sys_sigreturn,
// part 3 end
};
```

user.h：

```
C// part 3 begin
int sigalarm(int ticks, void (*handler)());
int sigreturn(void);
// part 3 end
```

usys.pl：

```c
C# part 3 begin
entry("sigalarm");
entry("sigreturn");
# part 3 end
```

修改trap.c，使得满足lab要求的功能：

```c
C//
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
  
  if(r_scause() == 8){
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  // if(which_dev == 2)
  //   yield();

  // part 3 begin
  // // test 0 begin
  // if(which_dev == 2) {
  //   p->ticks_pass += 1;
  //   if (p->ticks_pass % p->ticks == 0) {
  //     // change pc
  //     p->trapframe->epc = p->handler;
  //   }
  //   yield();
  // }
  // // test 0 end
  // test 1/2 begin
  // 不在handler中
  if((which_dev == 2) && (p->in_handler == 0)) {
    p->ticks_pass += 1;
    // 防止p->ticks = 0
    if (p->ticks && (p->ticks_pass == p->ticks)) {
      // 恢复计数器
      p->ticks_pass = 0;
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
  // test 1/2 end
  // part 3 end

  usertrapret();
}
```

sysproc.c：

```c
C// part 3 begin
uint64 sys_sigalarm(void) {
  int ticks;
  uint64 handler;
  // 第一个参数
  if(argint(0, &ticks) < 0)
    return -1;
  // 第二个参数
  if(argaddr(1, &handler) < 0)
    return -1;
  struct proc *p = myproc();
  p->ticks = ticks;
  p->handler = handler;

  return 0;
}

// test 0 begin
// uint64 sys_sigreturn(void) {
//   return 0;
// }
// test 0 end

// test 1/2 begin
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
// test 1/2 end
// part 3 end
```

测试：

```c
BASH./grade-lab-traps alarmtest
make: 'kernel/kernel' is up to date.
== Test running alarmtest == (3.9s) 
== Test   alarmtest: test0 == 
  alarmtest: test0: OK 
== Test   alarmtest: test1 == 
  alarmtest: test1: OK 
== Test   alarmtest: test2 == 
  alarmtest: test2: OK
```