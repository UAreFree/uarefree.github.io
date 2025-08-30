# MIT6.1810 Lab:Copy-on-Write Fork for Xv6


MIT6.1810 lab xv6 2024 cow

<!--more--> 

写时复制版的fork，思路是在fork时，uvmcopy会copy父进程的页表到子进程，建立对应的映射，但不去分配物理内存，也就是建立映射时，指向的物理内存是父进程的物理页，父子进程的虚拟地址同时映射到同一物理页。设置PTE为不可写，设置COW标志位。这样父进程或子进程其中一个试图写时，触发缺页故障，陷入内核，在usertrap中根据trap的类型处理trap，这里就是解除原PTE的映射，新分配一个物理页，建立到该物理页的映射，并且对应的PTE清除COW标志位，设为可写。
此外，还需要考虑的点是，因为同一物理页可能被多个进程映射了，那么kfree就需要判断什么时候正确释放物理页。这里使用引用计数，建立一个引用计数数组，维护每个物理页的引用计数（有多少进程虚拟地址映射到了该物理页）。kalloc时设置所分配的物理页引用计数为1，每当fork时uvmcopy复制映射就会将引用计数加一，调用kfree时先将引用计数减一，判断引用计数是否为0，从而再释放物理页面。（这里和智能指针的引用计数机制如出一辙）
按照这个思路simple的样例是可以通过的，但之后的就不会通过了。是因为多进程下对全局共享的引用计数数组需要加锁避免竞态条件下的错误。更改之后，file样例又不能通过，出现pipe() panic，是因为还需更改copyout()，这个函数是xv6实现的软件访问页表，将内核数据拷贝到用户态，这里不会触发缺页异常，而显然，如果要拷贝到用户态不可写的物理页上，就需要实现上述COW，所以在此函数增加检测COW物理页，并对应进程COW处理。至此，COW所有的test都可以通过。以下是代码实现的具体细节。
更改uvmcopy()：
```C
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  //char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    // 可读可写 不可读可写
    // 可读不可写 不可读不可写
    // 对于可写页进行COW
    if (*pte & PTE_W) {
      *pte = *pte & ~PTE_W;
      *pte = *pte | PTE_COW;
    }
    // 把父进程的权限标志位复制到子进程
    flags = PTE_FLAGS(*pte);
    // 建立到原物理页的映射
    if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
      goto err;
    }
    // 原物理页的引用计数加一
    acquire(&refcntlock);
    refcnt[GETPAINDEX((uint64)pa)]++;
    release(&refcntlock);

    // if((mem = kalloc()) == 0)
    //   goto err;
    // memmove(mem, (char*)pa, PGSIZE);
    // if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
    //   kfree(mem);
    //   goto err;
    // }
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```
检查COW物理页并进行COW处理函数，注意这里va不能等于MAXVA，这是个大坑，usertests一直MAXVA样例不通过就是没有加=这个原因。
```C
int
uvmcheckcow(uint64 va) {
  // 虚拟地址vm对应的页面是否是COW页
  if (va >= MAXVA) return 0;
  pte_t *pte;
  struct proc *p = myproc();
  if((pte = walk(p->pagetable, va, 0)) == 0)
    return 0;
  return va < p->sz && (*pte & PTE_V) && (*pte & PTE_COW);
}

int
uvmcopycow(uint64 va) {
  if (va >= MAXVA) return -1;
  pte_t *pte;
  uint flags;
  uint64 pa;
  uint64 mem;
  struct proc *p = myproc();
  if((pte = walk(p->pagetable, va, 0)) == 0)
    return -1;
  // 分配一个新的物理页面 并把原物理页面引用计数减1
  pa = PTE2PA(*pte);
  if ((mem = (uint64)cowalloc((void *)pa)) == 0)
    return -1;
  // 清除COW 设置为可写
  flags = PTE_FLAGS(*pte);
  if (*pte & PTE_COW) {
    flags = flags & ~PTE_COW;
    flags = flags | PTE_W;
  }
  // 解除原页面映射 会把PTE设为0
  uvmunmap(p->pagetable, PGROUNDDOWN(va), 1, 0); // 这里不做dofree
  // 建立新页面映射
  if(mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE, (uint64)mem, flags) != 0){
    return -1;
  }
  return 0;
}
```
usertrap增加对应的page fault处理：
```C
if(r_scause() == 8){
  // system call

  if(killed(p))
    exit(-1);

  // sepc points to the ecall instruction,
  // but we want to return to the next instruction.
  p->trapframe->epc += 4;

  // an interrupt will change sepc, scause, and sstatus,
  // so enable only now that we're done with those registers.
  intr_on();

  syscall();
} else if((which_dev = devintr()) != 0){
  // ok
} else if ((r_scause() == 13 || r_scause() == 15) && uvmcheckcow(r_stval())) {
  // 发生页面错误 且是COW页
  // 进行写时复制
  if (uvmcopycow(r_stval()) == -1)
    setkilled(p); // 没有可用的物理内存 杀死进程
}
else {
  printf("usertrap(): unexpected scause 0x%lx pid=%d\n", r_scause(), p->pid);
  printf("            sepc=0x%lx stval=0x%lx\n", r_sepc(), r_stval());
  setkilled(p);
}
```
引用计数需要维护的数组及锁：
```C
struct spinlock refcntlock;
int refcnt[(PHYSTOP - KERNBASE) / PGSIZE];

#define GETPAINDEX(p) (p - KERNBASE) / PGSIZE // 放在defs.h 不要放在kalloc.c

void
kinit()
{
  initlock(&kmem.lock, "kmem");
  initlock(&refcntlock, "refcnt");
  freerange(end, (void*)PHYSTOP);
}
```
kalloc和kfree增加引用计数判断：
```C
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // 页面引用计数 <= 0 时释放
  acquire(&refcntlock);
  if (--refcnt[GETPAINDEX((uint64)pa)] <= 0) {
    // Fill with junk to catch dangling refs.
    memset(pa, 1, PGSIZE);

    r = (struct run*)pa;

    acquire(&kmem.lock);
    r->next = kmem.freelist;
    kmem.freelist = r;
    release(&kmem.lock);
  }
  release(&refcntlock);
}

void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r) {
    memset((char*)r, 5, PGSIZE); // fill with junk
    refcnt[GETPAINDEX((uint64)r)] = 1; // 新分配的物理页面引用计数设为1
  }

  return (void*)r;
}
```
新增的cowalloc函数，这里没有直接kalloc的原因是，需要考虑到这一点，COW触发后只会将当前进程的COW清除，分配一个新的物理页，另一个进程依旧是COW页的。以父子进程为例，子进程COW处理后，父进程依旧是COW页，但此时其物理页的引用计数由2减为了1，当对其进行写入时，就不要再申请一个新的物理页了，复用原物理页即可。
```C
void *
  cowalloc(void* pa) {
  acquire(&refcntlock);

  // 当前物理页引用计数为1 说明是COW之后未做更改的进程的物理页
  // 直接使用原物理页 此函数之后清除其PTE_COW
  if (refcnt[GETPAINDEX((uint64)pa)] <= 1) {
    release(&refcntlock);
    return pa;
  }

  // 当前物理页引用计数为2 最初的COW
  // kalloc一片新的物理页 复制过去
  uint64 mem;
  if((mem = (uint64)kalloc()) == 0) {
    release(&refcntlock);
    return 0; // 内存不够
  }
  memmove((void*)mem, (void*)pa, PGSIZE);
  // 旧页引用计数减1
  refcnt[GETPAINDEX((uint64)pa)]--;

  release(&refcntlock);
  return (void *)mem;
}
```
最后就是copyout()，注意if条件的判断，当PTE不可写时，PTE可能是COW，所以原if条件直接将不可写PTE return是不合适的，应该增加COW判断。
```C
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;
  pte_t *pte;

  // 内核态虚拟内存拷贝到用户态
  // 其实是在像用户态内存进行写
  // 此时不会触发缺页异常 所以要在此函数实现
  // 判断用户页是否是COW页 如果是进行COW操作
  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    if(va0 >= MAXVA)
      return -1;
    pte = walk(pagetable, va0, 0);
    if(pte == 0 || (*pte & PTE_V) == 0 || (*pte & PTE_U) == 0 ||
       ((*pte & PTE_W) == 0 && (*pte & PTE_COW) == 0))
      return -1;
    if (*pte & PTE_COW) {
      if (uvmcopycow(dstva) == -1) return -1;
      // struct proc *p = myproc();
      // pte = walk(p->pagetable, va0, 0);
    }
    pa0 = PTE2PA(*pte);
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
```

---

> 作者: [UAreFree](https://github.com/UAreFree)  
> URL: http://localhost:1313/posts/mit-lab-cow/  

