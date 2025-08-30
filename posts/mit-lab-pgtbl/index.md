# MIT6.1810 Lab:Page Tables


MIT6.1810 lab xv6 2024 pgtbl

<!--more--> 

## Speed up system calls
加速系统调用：在内核空间和用户空间共享只读区域的数据，这样不需要跨内核，减少了用户态和内核态切换的开销。这里就是指频繁的系统调用会带来内核态与用户态的切换开销，为了减少这个开销，做法有：
1. 可以减少系统调用的次数（如设置用户buffer和内核buffer缓存系统调用请求数据，到达一定量时再一次性执行系统调用，典型的就是read()、write()）
2. 使用协程，用户态的线程，不会跨核
3. 本实验做法，内核空间与用户空间虚拟映射到同一个只读的物理页面，避免内核态和用户态切换（Linux称为VDSO）

本实验通过fork子进程，实现用户态程序ugetpid()和系统调用getpid()一样的效果。ugetpid()实现已经给出来，是读虚拟地址USYSCALL处的内容。所以需要在fork创建子进程时（内核态），向USYSCALL地址（用户态虚拟地址空间）写入struct usyscall，对应分配了一个物理页，那么之后用户态ugetpid就能直接在USYSCALL处读数据，而不跨核。
```C
int
ugetpid(void)
{
  struct usyscall *u = (struct usyscall *)USYSCALL;
  return u->pid;
}
```
代码实现参照trapframe生命周期代码，在进程分配trapframe内存、建立trapframe映射、释放trapframe内存这三部分，实现usyscall，存入当前进程的pid。
```C
// kernel/proc.c
// static struct proc* allocproc(void)
// Allocate a trapframe page.
if((p->trapframe = (struct trapframe *)kalloc()) == 0){
  freeproc(p);
  release(&p->lock);
  return 0;
}

// Allocate a usyscall page
if ((p->usyscall = (struct usyscall *)kalloc()) == 0) {
  freeproc(p);
  release(&p->lock);
  return 0;
}
p->usyscall->pid = p->pid;
```
```C
// kernel/proc.c
// pagetable_t proc_pagetable(struct proc *p)

// map the trapframe page just below the trampoline page, for
// trampoline.S.
if(mappages(pagetable, TRAPFRAME, PGSIZE,
            (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmfree(pagetable, 0);
  return 0;
}

if(mappages(pagetable, USYSCALL, PGSIZE,
          (uint64)(p->usyscall), PTE_R | PTE_U) < 0){
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmfree(pagetable, 0);
  return 0;
}
```
```C
// kernel/proc.c
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->usyscall)
    kfree((void*)p->usyscall);
  p->usyscall = 0;
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
```
总结：
1. 这个实验深刻认识到，内核空间是可以操作用户空间的，不要有固有思维内核空间只操作内核，用户空间只操作用户程序。是内核的权限高，可以操作任何代码，这里就可以拿到p->trapframe，你会说这不是进程用户空间的吗？实际上kernel代码是可以操作这页的。所以这个实验开始困惑的是用户空间和内核共享只读区域是什么，这里的答案就是用户空间内核都可以读写，是将内核操作的结果放到了用户可以读的用户地址空间，仅此而已，不是什么用户虚拟地址、内核虚拟地址映射到一个物理内存，这里没有涉及内核虚拟地址。
2. 回答最后可以speed up的syscall，很明显就是只涉及读取内核态的结果的系统调用，不对内核态进行修改，如getpid()、uptime()、getuid()、sbrk(0)等。再次强调，这里的加速是指不进行跨核。

## Print a page table
编写一个函数打印页表的内容。xv6是三级页表，打印的形式是..2级页表PTE，.. ..2级PTE下的所有1级PTE，.. .. ..1级PTE下的所有0级PTE。根据提示，可以参考freewalk递归遍历所有PTE。

freewalk这段代码递归实现了所有页表的释放，是理解递归实现的很好示例，它的出口就是到了叶子页（L0级页表）不再递归，然后交由上层递归kfree本级页表。
```C
void
freewalk(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    // PTE有效 && PTE不是叶子页（没有RWX位）
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
      pagetable[i] = 0;
    } else if(pte & PTE_V){ // freewalk之前要uvmunmap叶子页
      panic("freewalk: leaf");
    }
  }
  kfree((void*)pagetable); // kfree的位置很重要
}
```
参考freewalk实现的，就是用递归深度控制遍历层级，这里遍历的depth从0到2，对应着2、1、0三级页表（freewalk的depth相当于是从0到1），然后去打印对应的二进制就行。注意使用void *转换，能输出对应%p格式，以及这里的虚拟地址，是依赖上层pte的基地址的，所以也需要传递上层pte的虚拟地址到下层递归中。
```C
void
vmprintsub(pagetable_t pagetable, int depth, uint64 base) {
  if (depth > 2) return;
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if(pte & PTE_V){
      uint64 child = PTE2PA(pte);
      printf(".."); // 2 ..
      int count = depth;
      while (count--) { // 1 .. ..
        printf(" ..");  // 0 .. .. ..
      }
      uint64 va = base + (i << PXSHIFT(2-depth));
      printf("%p: pte %p pa %p\n", (void *)va, (void *)pte, (void *)PTE2PA(pte));
      vmprintsub((pagetable_t)child, depth + 1, va);
    }
  }
}
void
vmprint(pagetable_t pagetable) {
  // your code here
  printf("page table %p\n", pagetable);
  // 打印页表项
  // ..va: pte pa
  // .. ..va: pte pa
  // .. .. ..va: pte pa
  vmprintsub(pagetable, 0, 0);
}
```
## Use superpages
使用超级页，RISC-V分页硬件支持2M的页面，比普通4KB更大的页面称为超级页，2M的页面称为兆页面。OS设置L1级的PTE中的PTE_V和PTE_R位，并设置物理页号为指向2MB物理内存的区域起始位置。使用超级页可以减少页表使用的物理内存量，并可以减少TLB缓冲中未命中的次数，从而大幅提升性能。

题目中提示了，通过设置L1级的PTE直接映射到物理地址，因为2M的页面对应的就是2^(9+12)。所以代码就是在所有有关分配超级页内存时页表相关的操作。

首先在kalloc.c文件中参考原kalloc相关操作，增加超级页相关结构体、分配超级页物理内存、释放超级页物理内存代码。
```C
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;

struct {
  struct spinlock lock;
  struct run *freelist;
} superkmem; // 超级页链表

void
kinit()
{
  initlock(&kmem.lock, "kmem");
  initlock(&kmem.lock, "superkmem");
  freerange(end, (void*)PHYSTOP);
}

void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end - 50 * SUPERPGSIZE; p += PGSIZE)
    kfree(p);

  // 留出最后50个超级页的物理内存空间
  p = (char*)SUPERPGROUNDUP((uint64)p);
  for(; p + SUPERPGSIZE <= (char*)pa_end; p += SUPERPGSIZE)
    superkfree(p);
}

void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}

void
superkfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % SUPERPGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("superkfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, SUPERPGSIZE);

  r = (struct run*)pa;

  acquire(&superkmem.lock);
  r->next = superkmem.freelist;
  superkmem.freelist = r;
  release(&superkmem.lock);
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

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}

void *
superkalloc(void)
{
  struct run *r;

  acquire(&superkmem.lock);
  r = superkmem.freelist;
  if(r)
    superkmem.freelist = r->next;
  release(&superkmem.lock);

  if(r)
    memset((char*)r, 5, SUPERPGSIZE); // fill with junk
  return (void*)r;
}
```
实验题目提到通过sbrk()系统调用申请新分配的超级页，其所涉及到的调用路径是sys_sbrk()->growproc(n)->uvmalloc(p->pagetable, sz, sz + n, PTE_W)，在uvmalloc函数中判断size的大小，若申请的空间size大于等于2MB，说明是超级页，对应的就分配超级页物理内存（调用上述superkalloc），建立超级页映射（新建mapsuperpages）。
```C
uint64
uvmalloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz, int xperm)
{
  char *mem;
  uint64 a;
  int sz;

  if(newsz < oldsz)
    return oldsz;

  oldsz = PGROUNDUP(oldsz);
  for(a = oldsz; a < newsz; a += sz){
    if (newsz - oldsz >= SUPERPGSIZE && a % SUPERPGSIZE == 0) {
      sz = SUPERPGSIZE;
      mem = superkalloc();
    }else {
      sz = PGSIZE;
      mem = kalloc();
    }
    if(mem == 0){
      uvmdealloc(pagetable, a, oldsz);
      return 0;
    }
#ifndef LAB_SYSCALL
    memset(mem, 0, sz);
#endif
    if (sz == PGSIZE) {
      if(mappages(pagetable, a, sz, (uint64)mem, PTE_R|PTE_U|xperm) != 0){
        kfree(mem);
        uvmdealloc(pagetable, a, oldsz);
        return 0;
      }
    }else {
      if(mapsuperpages(pagetable, a, sz, (uint64)mem, PTE_R|PTE_U|xperm) != 0){
        superkfree(mem);
        uvmdealloc(pagetable, a, oldsz);
        return 0;
      }
    }
  }
  return newsz;
}
```
建立超级页的页表映射，使用了新建的PTE_S标志位（#define PTE_S (1L << 8) // super page），标志此页面是超级页面，是建立在L1级页表上的。以及之后walk的时候根据PTE_S标志位判断，若是超级页，也就不用遍历到L0级页表，L1级PTE所对应的PFN就是虚拟地址所对应的物理地址。
```C
int
mapsuperpages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
{
  uint64 a, last;
  pte_t *pte;

  if((va % SUPERPGSIZE) != 0)
    panic("mapsuperpages: va not aligned");

  if((size % SUPERPGSIZE) != 0)
    panic("mapsuperpages: size not aligned");

  if(size == 0)
    panic("mapsuperpages: size");

  a = va;
  last = va + size - SUPERPGSIZE;
  for(;;){
    if((pte = walksuper(pagetable, a, 1)) == 0)
      return -1;
    if(*pte & PTE_V)
      panic("mapsuperpages: remap");
    *pte = PA2PTE(pa) | perm | PTE_V |PTE_U | PTE_S;
    if(a == last)
      break;
    a += SUPERPGSIZE;
    pa += SUPERPGSIZE;
  }
  return 0;
}
```
```C
pte_t *
walksuper(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA)
    panic("walk");

  pte_t *pte = &pagetable[PX(2, va)]; // 提取2级、1级对应的pte
  if(*pte & PTE_V) {
    pagetable = (pagetable_t)PTE2PA(*pte); // pte对应的物理地址 2级对应1级页表起始地址
  } else {
    if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
      return 0;
    memset(pagetable, 0, PGSIZE);
    *pte = PA2PTE(pagetable) | PTE_V;
  }
  return &pagetable[PX(1, va)]; // 1级页表的pte
}
```
```C
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA)
    panic("walk");

  for(int level = 2; level > 0; level--) {
    pte_t *pte = &pagetable[PX(level, va)]; // 提取2级、1级对应的pte
    if(*pte & PTE_V) {
      pagetable = (pagetable_t)PTE2PA(*pte); // pte对应的物理地址 2级对应1级页表起始地址
      if (*pte & PTE_S) { // 如果是超级页PTE 直接返回
        return pte; // L1级页表的pte
      }
#ifdef LAB_PGTBL
      if(PTE_LEAF(*pte)) {
        return pte;
      }
#endif
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  return &pagetable[PX(0, va)]; // 0级页表的pte
}
```
根据提示，在fork子进程时，会调用uvmcopy函数copy父进程的页表，所以依旧判断PTE_S，增加超级页的copy操作。在退出时调用uvmunmap函数也是依旧判断PTE_S，进行对应的superkfree操作。
```C
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  char *mem;
  int szinc;

  for(i = 0; i < sz; i += szinc){
    szinc = PGSIZE;
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if (*pte & PTE_S) {
      szinc = SUPERPGSIZE;
      if((mem = superkalloc()) == 0)
        goto err;
      memmove(mem, (char*)pa, SUPERPGSIZE);
      if(mapsuperpages(new, i, SUPERPGSIZE, (uint64)mem, flags) != 0){
        superkfree(mem);
        goto err;
      }
    }else {
      if((mem = kalloc()) == 0)
        goto err;
      memmove(mem, (char*)pa, PGSIZE);
      if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
        kfree(mem);
        goto err;
      }
    }

  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```
```C
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;
  int sz;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += sz){
    sz = PGSIZE;
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0) {
      printf("va=%ld pte=%ld\n", a, *pte);
      panic("uvmunmap: not mapped");
    }
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      if ((*pte & PTE_S) == 0) { // 普通页面
        uint64 pa = PTE2PA(*pte);
        kfree((void*)pa);
      }else if (*pte & PTE_S) { // 超级页
        uint64 pa = PTE2PA(*pte);
        superkfree((void*)pa);
        sz = SUPERPGSIZE;
      }else {
        panic("uvmunmap: not a leaf\n");
      }
    }
    *pte = 0;
  }
}
```



---

> 作者: [UAreFree](https://github.com/UAreFree)  
> URL: http://localhost:1313/posts/mit-lab-pgtbl/  

