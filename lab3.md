

## Print a page table ([easy](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

vm.c：

```c
C// part 1 begin
// print page tage
static char* prefix[3] = {"..", ".. ..", ".. .. .."};
void vmprint(pagetable_t pagetable) {
  printf("page table %p\n", pagetable);
  vmprint_rec(pagetable, 0);
}

void vmprint_rec(pagetable_t pagetable, int l) {
  if (l > 2) {
    return;
  }
  for (int i = 0; i < 512; i++) {
    pte_t pte = pagetable[i];
    // 合法则继续遍历
    if (pte & PTE_V) {
      uint64 pa = PTE2PA(pte);
      printf("%s%d: pte %p pa %p\n", prefix[l], i, pte, pa);
      vmprint_rec((pagetable_t)pa, l + 1);
    }
  }
}
// part 1 end
```

测试：

```c
CODE./grade-lab-pgtbl pte
make: 'kernel/kernel' is up to date.
== Test pte printout == pte printout: OK (3.0s) 
```

## A kernel page table per process ([hard](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

目标：进程中添加内核页表。

这里使用的思路是共享内核页表，代码如下：

proc.h：

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
  // part 2 begin
  pagetable_t kernel_pagetable;
  // part 2 end
};
```

vm.c：

```c
C// part 2 begin
pagetable_t
proc_kvminit(void)
{
  pagetable_t pagetable = uvmcreate();
  /** 
   * 复制L2页表, kernel_pagetable[0] >= 0x0, < 0x4,0000,000,
   * 包含UART0, VIRTIO0, CLINT, PLIC，这部分不和kernel共享, 其余部分和kernel共享。
  **/
  for (int i = 1; i < 512; i++) {
    pagetable[i] = kernel_pagetable[i];
  }

  // uart registers
  kvmmap_(pagetable, UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  kvmmap_(pagetable, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  kvmmap_(pagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  kvmmap_(pagetable, PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  return pagetable;
}
// part 2 end

// part 2 begin
void
kvminithart_(pagetable_t pagetable)
{
  w_satp(MAKE_SATP(pagetable));
  sfence_vma();
}
// part 2 end

// part 2 begin
void
free_no_leaf(pagetable_t pagetable, int l)
{
  // 只处理到L1
  for (int i = 0; i < 1; i++) {
    pte_t pte = pagetable[i];
    pagetable_t level1 = (pagetable_t)PTE2PA(pte);
    for (int j = 0; j < 512; j++) {
      pte_t pte = level1[j];
      if (pte & PTE_V) {
        pagetable_t level2 = (pagetable_t)PTE2PA(pte);
        kfree((void *) level2);
        level1[j] = 0;
      }
    }
    kfree((void *) level1);
  }
  
  kfree((void*)pagetable);
}
// part 2 end
```

proc.c：

```c
C// Look in the process table for an UNUSED proc.
// If found, initialize state required to run in the kernel,
// and return with p->lock held.
// If there are no free procs, or a memory allocation fails, return 0.
static struct proc*
allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }


  return 0;

found:
  p->pid = allocpid();

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    release(&p->lock);
    return 0;
  }

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // part 2 begin
  // init
  p->kernel_pagetable = proc_kvminit();
  if(p->kernel_pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  // part 2 end

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}

// free a proc structure and the data hanging from it,
// including user pages.
// p->lock must be held.
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  // part 2 begin
  // 只删除到第二层
  if (p->kernel_pagetable) {
    free_no_leaf(p->kernel_pagetable, 0);
  }
  // part 2 end

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

// Per-CPU process scheduler.
// Each CPU calls scheduler() after setting itself up.
// Scheduler never returns.  It loops, doing:
//  - choose a process to run.
//  - swtch to start running that process.
//  - eventually that process transfers control
//    via swtch back to the scheduler.
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();
    
    int found = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;

        // part 2 begin
        // change to p->kernel_pagetable
        w_satp(MAKE_SATP(p->kernel_pagetable));
        sfence_vma();
        // part 2 end

        swtch(&c->context, &p->context);
        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;

        // part 2 begin
        // change to kernel_pagetable
        kvminithart();
        // part 2 end

        found = 1;
      }
      release(&p->lock);
    }
#if !defined (LAB_FS)
    if(found == 0) {
      intr_on();
      asm volatile("wfi");
    }
#else
    ;
#endif
  }
}
```

测试：

```c
CODE./grade-lab-pgtbl usertests
riscv64-unknown-elf-gcc -Wall -Werror -O -fno-omit-frame-pointer -ggdb -DSOL_PGTBL -DLAB_PGTBL -MD -mcmodel=medany -ffreestanding -fno-common -nostdlib -mno-relax -I. -fno-stack-protector -fno-pie -no-pie   -c -o kernel/vm.o kernel/vm.c
riscv64-unknown-elf-gcc -Wall -Werror -O -fno-omit-frame-pointer -ggdb -DSOL_PGTBL -DLAB_PGTBL -MD -mcmodel=medany -ffreestanding -fno-common -nostdlib -mno-relax -I. -fno-stack-protector -fno-pie -no-pie   -c -o kernel/proc.o kernel/proc.c
riscv64-unknown-elf-ld -z max-page-size=4096 -T kernel/kernel.ld -o kernel/kernel kernel/entry.o kernel/start.o kernel/console.o kernel/printf.o kernel/uart.o kernel/kalloc.o kernel/spinlock.o kernel/string.o kernel/main.o kernel/vm.o kernel/proc.o kernel/swtch.o kernel/trampoline.o kernel/trap.o kernel/syscall.o kernel/sysproc.o kernel/bio.o kernel/fs.o kernel/log.o kernel/sleeplock.o kernel/file.o kernel/pipe.o kernel/exec.o kernel/sysfile.o kernel/kernelvec.o kernel/plic.o kernel/virtio_disk.o kernel/vmcopyin.o kernel/stats.o kernel/sprintf.o 
riscv64-unknown-elf-ld: warning: cannot find entry symbol _entry; defaulting to 0000000080000000
riscv64-unknown-elf-objdump -S kernel/kernel > kernel/kernel.asm
riscv64-unknown-elf-objdump -t kernel/kernel | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$/d' > kernel/kernel.sym
== Test usertests == (177.0s) 
== Test   usertests: copyin == 
  usertests: copyin: OK 
== Test   usertests: copyinstr1 == 
  usertests: copyinstr1: OK 
== Test   usertests: copyinstr2 == 
  usertests: copyinstr2: OK 
== Test   usertests: copyinstr3 == 
  usertests: copyinstr3: OK 
== Test   usertests: sbrkmuch == 
  usertests: sbrkmuch: OK 
== Test   usertests: all tests == 
  usertests: all tests: OK (0.6s)
```

## Simplify `copyin/copyinstr` ([hard](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

将用户映射添加到每个进程的内核页表，区别：

copyin在用户进程使用，va != pa，copyin_new在kernel中使用，va = pa。

代码：

defs.h：

```c
C// part 3 begin
int             copyin_new(pagetable_t, char*, uint64, uint64);
int             copyinstr_new(pagetable_t, char*, uint64, uint64);
void            copy_proc_to_kernel(pagetable_t, pagetable_t, uint64, uint64);
// part 3 end
```

exec.c：

```c
Cint
exec(char *path, char **argv)
{
  char *s, *last;
  int i, off;
  uint64 argc, sz = 0, sp, ustack[MAXARG+1], stackbase;
  struct elfhdr elf;
  struct inode *ip;
  struct proghdr ph;
  pagetable_t pagetable = 0, oldpagetable;
  struct proc *p = myproc();

  begin_op();

  if((ip = namei(path)) == 0){
    end_op();
    return -1;
  }
  ilock(ip);

  // Check ELF header
  if(readi(ip, 0, (uint64)&elf, 0, sizeof(elf)) != sizeof(elf))
    goto bad;
  if(elf.magic != ELF_MAGIC)
    goto bad;

  if((pagetable = proc_pagetable(p)) == 0)
    goto bad;

  // Load program into memory.
  for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
    if(readi(ip, 0, (uint64)&ph, off, sizeof(ph)) != sizeof(ph))
      goto bad;
    if(ph.type != ELF_PROG_LOAD)
      continue;
    if(ph.memsz < ph.filesz)
      goto bad;
    if(ph.vaddr + ph.memsz < ph.vaddr)
      goto bad;
    uint64 sz1;
    if((sz1 = uvmalloc(pagetable, sz, ph.vaddr + ph.memsz)) == 0)
      goto bad;
    sz = sz1;
    if(ph.vaddr % PGSIZE != 0)
      goto bad;
    if(loadseg(pagetable, ph.vaddr, ip, ph.off, ph.filesz) < 0)
      goto bad;
  }
  iunlockput(ip);
  end_op();
  ip = 0;

  p = myproc();
  uint64 oldsz = p->sz;

  // Allocate two pages at the next page boundary.
  // Use the second as the user stack.
  sz = PGROUNDUP(sz);
  uint64 sz1;
  if((sz1 = uvmalloc(pagetable, sz, sz + 2*PGSIZE)) == 0)
    goto bad;
  sz = sz1;
  uvmclear(pagetable, sz-2*PGSIZE);
  sp = sz;
  stackbase = sp - PGSIZE;

  // Push argument strings, prepare rest of stack in ustack.
  for(argc = 0; argv[argc]; argc++) {
    if(argc >= MAXARG)
      goto bad;
    sp -= strlen(argv[argc]) + 1;
    sp -= sp % 16; // riscv sp must be 16-byte aligned
    if(sp < stackbase)
      goto bad;
    if(copyout(pagetable, sp, argv[argc], strlen(argv[argc]) + 1) < 0)
      goto bad;
    ustack[argc] = sp;
  }
  ustack[argc] = 0;

  // push the array of argv[] pointers.
  sp -= (argc+1) * sizeof(uint64);
  sp -= sp % 16;
  if(sp < stackbase)
    goto bad;
  if(copyout(pagetable, sp, (char *)ustack, (argc+1)*sizeof(uint64)) < 0)
    goto bad;

  // arguments to user main(argc, argv)
  // argc is returned via the system call return
  // value, which goes in a0.
  p->trapframe->a1 = sp;

  // Save program name for debugging.
  for(last=s=path; *s; s++)
    if(*s == '/')
      last = s+1;
  safestrcpy(p->name, last, sizeof(p->name));
    
  // Commit to the user image.
  oldpagetable = p->pagetable;
  p->pagetable = pagetable;
  p->sz = sz;
  p->trapframe->epc = elf.entry;  // initial program counter = main
  p->trapframe->sp = sp; // initial stack pointer
  proc_freepagetable(oldpagetable, oldsz);

  // part 1 begin
  if(p->pid==1) {
    vmprint(p->pagetable);
  }
  // part 1 end

  // part 3 begin
  copy_proc_to_kernel(p->pagetable, p->kernel_pagetable, 0, sz);
  // part 3 end

  return argc; // this ends up in a0, the first argument to main(argc, argv)

 bad:
  if(pagetable)
    proc_freepagetable(pagetable, sz);
  if(ip){
    iunlockput(ip);
    end_op();
  }
  return -1;
}
```

proc.c：

```c
C// Set up first user process.
void
userinit(void)
{
  struct proc *p;

  p = allocproc();
  initproc = p;
  
  // allocate one user page and copy init's instructions
  // and data into it.
  uvminit(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;

  // prepare for the very first "return" from kernel to user.
  p->trapframe->epc = 0;      // user program counter
  p->trapframe->sp = PGSIZE;  // user stack pointer

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  p->state = RUNNABLE;

  // part 3 begin
  copy_proc_to_kernel(p->pagetable, p->kernel_pagetable, 0, p->sz);
  // part 3 end

  release(&p->lock);
}

// Grow or shrink user memory by n bytes.
// Return 0 on success, -1 on failure.
int
growproc(int n)
{
  uint sz;
  struct proc *p = myproc();

  sz = p->sz;
  if(n > 0){
    if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
      return -1;
    }
  } else if(n < 0){
    sz = uvmdealloc(p->pagetable, sz, sz + n);
  }
  // part 3 begin
  copy_proc_to_kernel(p->pagetable, p->kernel_pagetable, p->sz, sz);
  // part 3 end
  p->sz = sz;

  return 0;
}

// Create a new process, copying the parent.
// Sets up child kernel stack to return as if from fork() system call.
int
fork(void)
{
  int i, pid;
  struct proc *np;
  struct proc *p = myproc();

  // Allocate process.
  if((np = allocproc()) == 0){
    return -1;
  }

  // Copy user memory from parent to child.
  if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
  }
  np->sz = p->sz;

  np->parent = p;

  // copy saved user registers.
  *(np->trapframe) = *(p->trapframe);

  // Cause fork to return 0 in the child.
  np->trapframe->a0 = 0;

  // increment reference counts on open file descriptors.
  for(i = 0; i < NOFILE; i++)
    if(p->ofile[i])
      np->ofile[i] = filedup(p->ofile[i]);
  np->cwd = idup(p->cwd);

  safestrcpy(np->name, p->name, sizeof(p->name));

  pid = np->pid;

  np->state = RUNNABLE;

  // part 3 begin
  // 复制np
  copy_proc_to_kernel(np->pagetable, np->kernel_pagetable, 0, np->sz);
  // part 3 end

  release(&np->lock);

  return pid;
}
```

vm.c：

```c
C// part 3 begin
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  return copyin_new(pagetable, dst, srcva, len);
}
// part 3 end

// part 3 begin
void copy_proc_to_kernel(pagetable_t proc_pt, pagetable_t kernel_pt, uint64 oldsz, uint64 newsz) {
  uint64 a;

  if (newsz >= PLIC) {
    panic("user processes exceed PLIC");
  }

  for(a = oldsz; a < newsz; a += PGSIZE){
    pte_t *pte1 = walk(proc_pt, a, 0);
    if (pte1 == 0) {
      panic("no user pte");
    }
    if ((*pte1 & PTE_V) == 0) {
      panic("no valid user pte");
    }
    pte_t *pte2 = walk(kernel_pt, a, 1);
    if (pte2 == 0) {
      panic("no kernel pte");
    }
    
    // 复制
    *pte2 = *pte1;
    // 关闭读写权限, 关闭用户权限, 否则kernel无法使用
    *pte2 &= ~(PTE_U|PTE_W|PTE_X);
  }
}
// part 3 end
```

测试：

```c
CODE./grade-lab-pgtbl usertests
riscv64-unknown-elf-gcc -Wall -Werror -O -fno-omit-frame-pointer -ggdb -DSOL_PGTBL -DLAB_PGTBL -MD -mcmodel=medany -ffreestanding -fno-common -nostdlib -mno-relax -I. -fno-stack-protector -fno-pie -no-pie   -c -o kernel/vm.o kernel/vm.c
riscv64-unknown-elf-gcc -Wall -Werror -O -fno-omit-frame-pointer -ggdb -DSOL_PGTBL -DLAB_PGTBL -MD -mcmodel=medany -ffreestanding -fno-common -nostdlib -mno-relax -I. -fno-stack-protector -fno-pie -no-pie   -c -o kernel/proc.o kernel/proc.c
riscv64-unknown-elf-gcc -Wall -Werror -O -fno-omit-frame-pointer -ggdb -DSOL_PGTBL -DLAB_PGTBL -MD -mcmodel=medany -ffreestanding -fno-common -nostdlib -mno-relax -I. -fno-stack-protector -fno-pie -no-pie   -c -o kernel/exec.o kernel/exec.c
riscv64-unknown-elf-ld -z max-page-size=4096 -T kernel/kernel.ld -o kernel/kernel kernel/entry.o kernel/start.o kernel/console.o kernel/printf.o kernel/uart.o kernel/kalloc.o kernel/spinlock.o kernel/string.o kernel/main.o kernel/vm.o kernel/proc.o kernel/swtch.o kernel/trampoline.o kernel/trap.o kernel/syscall.o kernel/sysproc.o kernel/bio.o kernel/fs.o kernel/log.o kernel/sleeplock.o kernel/file.o kernel/pipe.o kernel/exec.o kernel/sysfile.o kernel/kernelvec.o kernel/plic.o kernel/virtio_disk.o kernel/vmcopyin.o kernel/stats.o kernel/sprintf.o 
riscv64-unknown-elf-ld: warning: cannot find entry symbol _entry; defaulting to 0000000080000000
riscv64-unknown-elf-objdump -S kernel/kernel > kernel/kernel.asm
riscv64-unknown-elf-objdump -t kernel/kernel | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$/d' > kernel/kernel.sym
== Test usertests == (110.7s) 
== Test   usertests: copyin == 
  usertests: copyin: OK 
== Test   usertests: copyinstr1 == 
  usertests: copyinstr1: OK 
== Test   usertests: copyinstr2 == 
  usertests: copyinstr2: OK 
== Test   usertests: copyinstr3 == 
  usertests: copyinstr3: OK 
== Test   usertests: sbrkmuch == 
  usertests: sbrkmuch: OK 
== Test   usertests: all tests == 
  usertests: all tests: OK 
```

