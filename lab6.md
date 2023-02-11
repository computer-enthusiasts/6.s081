## Implement copy-on write([hard](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

### 注意点

- 需要增加记录应用次数的数组；
- 只有当引用计数为0时，才能真正free；
- 注意设置flag；

### 代码

kalloc.c，注意需要初始化锁以及cnt数组：

```c
C// part 1 begin
struct spinlock cnt_lock;
int cnt[PHYSTOP >> 12];
// part 1 end

struct run {
  struct run *next;
};

struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;

void
kinit()
{
  initlock(&kmem.lock, "kmem");
  // part 1 begin
  initlock(&cnt_lock, "cow");
  // part 1 end
  freerange(end, (void*)PHYSTOP);
}

void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);

  // part 1 begin
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE) {
    // 这里要设置为1, 因为后续会free
    cnt[((uint64)p) >> 12] = 1;
    kfree(p);
  }
  // part 1 end
}

// Free the page of physical memory pointed at by v,
// which normally should have been returned by a
// call to kalloc().  (The exception is when
// initializing the allocator; see kinit above.)
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // part 1 begin
  acquire(&cnt_lock);
  int index = ((uint64)pa) >> 12;
  if (cnt[index] >= 1) {
    cnt[index] -= 1;
  }
  release(&cnt_lock);
  // 如果引用计数大于0, 则不free
  if (cnt[index] > 0) {
    return;
  }
  // part 1 end

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}

// Allocate one 4096-byte page of physical memory.
// Returns a pointer that the kernel can use.
// Returns 0 if the memory cannot be allocated.
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if (r) {
    kmem.freelist = r->next;
    // part 1 begin
    acquire(&cnt_lock);
    // 更新计数
    cnt[((uint64)r) >> 12] = 1;
    release(&cnt_lock);
    // part 1 end
  }
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk


  return (void*)r;
}
```

riscv.h：

```c
C// part 1 begin
#define PTE_RSW (1L << 8)
#define GET_PTE_RSW(pte) (((pte) >> 8) & 1)
// part 1 end
```

trap.c：

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
  // part 1 begin
  // 写时才报错
  else if ((r_scause() == 15)) {
    uint64 va = r_stval();
    if (cow(p->pagetable, va) < 0) {
      p->killed = 1;
    }
  }
  // part 1 end
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

vm.c：

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
  // char *mem;

  // printf("uvmcopy start\n");
  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);

    // part 1 begin
    // 更新计数
    acquire(&cnt_lock);
    cnt[((uint64)pa) >> 12] += 1;
    release(&cnt_lock);

    // flag
    *pte = *pte & (~PTE_W);
    // RSW
    *pte = *pte | (PTE_RSW);
    flags = PTE_FLAGS(*pte);
    // 将虚拟地址i映射到物理地址pa
    if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
      // kfree(mem);
      goto err;
    }
    // part 1 end
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}

// Copy from kernel to user.
// Copy len bytes from src to virtual address dstva in a given page table.
// Return 0 on success, -1 on error.
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;

    // part 1 begin
    pte_t* pte = walk(pagetable, va0, 0);
    if (pte == 0) {
      return -1;
    }
    // 只有在不可写时才调用cow
    if ((*pte & PTE_W) == 0) {
      if (cow(pagetable, va0) < 0) {
        return -1;
      }
    }
    // 更新pa0
    pa0 = PTE2PA(*pte);
    // part 1 end

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

// part 1 begin
int cow(pagetable_t pagetable, uint64 va) {
    if (va >= MAXVA) {
      return -1;
    }
    // 找到va对应的pte
    pte_t* pte = walk(pagetable, va, 0);
    if (pte == 0) {
      return -1;
    }
    if ((*pte & PTE_V) == 0) {
      return -1;
    }
    if ((*pte & PTE_RSW) == 0) {
      return -1;
    }
    if ((*pte & PTE_U) == 0) {
      return -1;
    }
    // 物理地址
    uint64 pa = PTE2PA(*pte);
    uint64 ka = (uint64) kalloc();
    if (ka == 0) {
      return -1;
    } else {
      // 复制pa到ka
      memmove((char*)ka, (char*)pa, PGSIZE);
      uint64 flags = PTE_FLAGS(*pte);
      *pte = PA2PTE(ka) | flags | PTE_W;
      *pte &= (~PTE_RSW);
      // 更新计数
      kfree((void *)pa);

      return 0;
    }
}
// part 1 end
```

### 测试

```c
./grade-lab-cow usertest
== Test running cowtest == (6.8s) 

./grade-lab-cow cowtest 
== Test usertests == (130.9s) 
== Test   usertests: copyin == 
  usertests: copyin: OK 
== Test   usertests: copyout == 
```