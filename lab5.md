## Eliminate allocation from sbrk() ([easy](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

修改xv6-labs-2020/kernel/sysproc.c/sys_sbrk，这里只更新sz的大小即可：

```c
Cuint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  // if(growproc(n) < 0)
  //   return -1;
  // part 1 begin
  myproc()->sz += n;
  // part 1 end
  return addr;
}
```

测试：

```c
make qemu
$ echo hi
page fault 0x0000000000004008
page fault 0x0000000000013f48
panic: uvmunmap: not mapped
```

## Lazy allocation ([moderate](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

修改xv6-labs-2020/kernel/trap.c/usertrap，当Load page fault, Store/AMO page fault错误产生时申请页面：

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
  } 
  // part 2 begin
  // Load page fault, Store/AMO page fault
  else if ((r_scause() == 13) || (r_scause() == 15)) {
    // 返回出错的虚拟地址
    uint64 va = r_stval();
    printf("page fault %p\n", va);
    uint64 ka = (uint64) kalloc();
    if (ka == 0) {
      p->killed = 1;
    } else {
      memset((void*)ka, 0, PGSIZE);
      va = PGROUNDDOWN(va);
      if(mappages(p->pagetable, va, PGSIZE, ka, PTE_W|PTE_R|PTE_U) != 0) {
        kfree((void*)ka);
        p->killed = 1;
      }
    }
  }
  // part 2 end
  else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

修改xv6-labs-2020/kernel/vm.c，跳过报错：

```c
C// Remove npages of mappings starting from va. va must be
// page-aligned. The mappings must exist.
// Optionally free the physical memory.
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    // if((*pte & PTE_V) == 0)
    //   panic("uvmunmap: not mapped");

    // part 2 begin
    // 跳过报错
    if((*pte & PTE_V) == 0) {
      continue;
    }
    // part 2 end
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```

测试：

```c
make qemu
$ echo hi
page fault 0x0000000000004008
page fault 0x0000000000013f48
hi
```

## Lazytests and Usertests ([moderate](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

修改xv6-labs-2020/kernel/sysproc.c/sys_sbrk，在负数情形直接使用growproc部分的代码进行处理：

```c
Cuint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  // if(growproc(n) < 0)
  //   return -1;
  // // part 1 begin
  // myproc()->sz += n;
  // // part 1 end
  // part 3 begin
  if (n >= 0) {
    myproc()->sz += n;
  } else {
    // n < 0时直接删除映射
    uint sz = myproc()->sz;
    myproc()->sz = uvmdealloc(myproc()->pagetable, sz, sz + n);
  }
  // part 3 end
  return addr;
}
```

修改xv6-labs-2020/kernel/trap.c/usertrap，增加判断va地址是否合法的部分：

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
  } 
  // part 3 begin
  // page fault
  else if ((r_scause() == 13) || (r_scause() == 15)) {
    // 返回出错的虚拟地址
    uint64 va = r_stval();
    // 不能添加printf, 否则答案不正确
    // va地址>=p->sz, 或<stack
    if ((va >= p->sz) || (va < p->trapframe->sp)) {
      p->killed = 1;
    } else {
      uint64 ka = (uint64) kalloc();
      if (ka == 0) {
        p->killed = 1;
      } else {
        memset((void*)ka, 0, PGSIZE);
        va = PGROUNDDOWN(va);
        if(mappages(p->pagetable, va, PGSIZE, ka, PTE_W|PTE_R|PTE_U) != 0) {
          kfree((void*)ka);
          p->killed = 1;
        }
      }
    }
  }
  // part 3 end

  else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

接着修改vm.c函数，注意需要查看实验中提到的fork, read, write函数调用了哪些内存相关的函数：

```c
fork -> uvmcopy
read -> sys_read -> fileread -> piperead -> copyout
write -> sys_write -> filewrite -> pipewrite -> copyin
```

在vm.c一开始添加：

```c
// part 3 begin
#include "spinlock.h"
#include "proc.h"
// part 3 end
```

修改uvmunmap：

```c
C// Remove npages of mappings starting from va. va must be
// page-aligned. The mappings must exist.
// Optionally free the physical memory.
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    // part 3 begin
    if((pte = walk(pagetable, a, 0)) == 0) {
      continue;
    }
    if((*pte & PTE_V) == 0) {
      continue;
    }
    // part 3 end
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```

修改uvmcopy：

```c
C// Given a parent process's page table, copy
// its memory into a child's page table.
// Copies both the page table and the
// physical memory.
// returns 0 on success, -1 on failure.
// frees any allocated pages on failure.
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    // part 3 begin
    if((pte = walk(old, i, 0)) == 0) {
      continue;
    }
    if((*pte & PTE_V) == 0) {
      continue;
    }
    // part 3 end
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if((mem = kalloc()) == 0)
      goto err;
    memmove(mem, (char*)pa, PGSIZE);
    if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
      kfree(mem);
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

copyin，copyout：

```c
C// Copy from kernel to user.
// Copy len bytes from src to virtual address dstva in a given page table.
// Return 0 on success, -1 on error.
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;
  // part 3 begin
  struct proc *p = myproc();
  // part 3 end

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    pa0 = walkaddr(pagetable, va0);
    // part 3 begin
    if (pa0 == 0) {
      if ((va0 >= p->sz) || (va0 < p->trapframe->sp)) {
        return -1;
      } else {
        pa0 = (uint64)kalloc();
        if (pa0 == 0) {
          p->killed = 1;
        } else {
          memset((void *)pa0, 0, PGSIZE);
          va0 = PGROUNDDOWN(va0);
          if(mappages(p->pagetable, va0, PGSIZE, pa0, PTE_W|PTE_R|PTE_U) != 0) {
            kfree((void *)pa0);
            p->killed = 1;
          }
        }
      }
    }
    // part 3 end
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}

// Copy from user to kernel.
// Copy len bytes to dst from virtual address srcva in a given page table.
// Return 0 on success, -1 on error.
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  uint64 n, va0, pa0;
  // part 3 begin
  struct proc *p = myproc();
  // part 3 end
  while(len > 0){
    va0 = PGROUNDDOWN(srcva);
    pa0 = walkaddr(pagetable, va0);
    // part 3 begin
    if (pa0 == 0) {
      if ((va0 >= p->sz) || (va0 < p->trapframe->sp)) {
        return -1;
      } else {
        pa0 = (uint64) kalloc();
        if (pa0 == 0) {
          p->killed = 1;
        } else {
          memset((void *)pa0, 0, PGSIZE);
          va0 = PGROUNDDOWN(va0);
          if(mappages(p->pagetable, va0, PGSIZE, pa0, PTE_W|PTE_R|PTE_U) != 0) {
            kfree((void *)pa0);
            p->killed = 1;
          }
        }
      }
    }
    // part 3 end
    n = PGSIZE - (srcva - va0);
    if(n > len)
      n = len;
    memmove(dst, (void *)(pa0 + (srcva - va0)), n);

    len -= n;
    dst += n;
    srcva = va0 + PGSIZE;
  }
  return 0;
}
```

测试1：

```
./grade-lab-lazy lazytests
== Test running lazytests == (1.0s)
```

测试2：

```
./grade-lab-lazy usertests
== Test usertests == (136.4s) 
== Test   usertests: pgbug == 
  usertests: pgbug: OK 
== Test   usertests: sbrkbugs == 
  usertests: sbrkbugs: OK 
== Test   usertests: argptest == 
  usertests: argptest: OK 
== Test   usertests: sbrkmuch == 
  usertests: sbrkmuch: OK 
== Test   usertests: sbrkfail == 
  usertests: sbrkfail: OK 
== Test   usertests: sbrkarg == 
  usertests: sbrkarg: OK 
== Test   usertests: stacktest == 
  usertests: stacktest: OK 
== Test   usertests: execout == 
  usertests: execout: OK 
== Test   usertests: copyin == 
  usertests: copyin: OK 
== Test   usertests: copyout == 
  usertests: copyout: OK 
== Test   usertests: copyinstr1 == 
  usertests: copyinstr1: OK 
== Test   usertests: copyinstr2 == 
  usertests: copyinstr2: OK 
== Test   usertests: copyinstr3 == 
  usertests: copyinstr3: OK 
== Test   usertests: rwsbrk == 
  usertests: rwsbrk: OK 
== Test   usertests: truncate1 == 
  usertests: truncate1: OK 
== Test   usertests: truncate2 == 
  usertests: truncate2: OK 
== Test   usertests: truncate3 == 
  usertests: truncate3: OK 
== Test   usertests: reparent2 == 
  usertests: reparent2: OK 
== Test   usertests: badarg == 
  usertests: badarg: OK 
== Test   usertests: reparent == 
  usertests: reparent: OK 
== Test   usertests: twochildren == 
  usertests: twochildren: OK 
== Test   usertests: forkfork == 
  usertests: forkfork: OK 
== Test   usertests: forkforkfork == 
  usertests: forkforkfork: OK 
== Test   usertests: createdelete == 
  usertests: createdelete: OK 
== Test   usertests: linkunlink == 
  usertests: linkunlink: OK 
== Test   usertests: linktest == 
  usertests: linktest: OK 
== Test   usertests: unlinkread == 
  usertests: unlinkread: OK 
== Test   usertests: concreate == 
  usertests: concreate: OK 
== Test   usertests: subdir == 
  usertests: subdir: OK 
== Test   usertests: fourfiles == 
  usertests: fourfiles: OK 
== Test   usertests: sharedfd == 
  usertests: sharedfd: OK 
== Test   usertests: exectest == 
  usertests: exectest: OK 
== Test   usertests: bigargtest == 
  usertests: bigargtest: OK 
== Test   usertests: bigwrite == 
  usertests: bigwrite: OK 
== Test   usertests: bsstest == 
  usertests: bsstest: OK 
== Test   usertests: sbrkbasic == 
  usertests: sbrkbasic: OK 
== Test   usertests: kernmem == 
  usertests: kernmem: OK 
== Test   usertests: validatetest == 
  usertests: validatetest: OK 
== Test   usertests: opentest == 
  usertests: opentest: OK 
== Test   usertests: writetest == 
  usertests: writetest: OK 
== Test   usertests: writebig == 
  usertests: writebig: OK 
== Test   usertests: createtest == 
  usertests: createtest: OK 
== Test   usertests: openiput == 
  usertests: openiput: OK 
== Test   usertests: exitiput == 
  usertests: exitiput: OK 
== Test   usertests: iput == 
  usertests: iput: OK 
== Test   usertests: mem == 
  usertests: mem: OK 
== Test   usertests: pipe1 == 
  usertests: pipe1: OK 
== Test   usertests: preempt == 
  usertests: preempt: OK 
== Test   usertests: exitwait == 
  usertests: exitwait: OK 
== Test   usertests: rmdot == 
  usertests: rmdot: OK 
== Test   usertests: fourteen == 
  usertests: fourteen: OK 
== Test   usertests: bigfile == 
  usertests: bigfile: OK 
== Test   usertests: dirfile == 
  usertests: dirfile: OK 
== Test   usertests: iref == 
  usertests: iref: OK 
== Test   usertests: forktest == 
  usertests: forktest: OK 
```

