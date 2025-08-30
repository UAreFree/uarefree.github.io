# MIT6.1810 Lab:Mmap


MIT6.1810 lab xv6 2024 mmap

<!--more--> 

先读一下从内核世界透视{{<link href="https://mp.weixin.qq.com/s/AUsgFOaePwVsPozC3F6Wjw" content="mmap内存映射的本质（原理篇）">}}这篇文章。
```C
#include <sys/mman.h>
void* mmap(void* addr, size_t length, int prot, int flags, int fd, off_t offset);
```
系统调用mmap映射的是虚拟内存中的文件映射与匿名映射区，在这段虚拟内存区域中，包含了一段一段的虚拟映射区，每调用一次mmap就会在文件映射与匿名映射区划分一段作为申请的虚拟内存。

addr：指定映射的虚拟内存起始地址，一般设为NULL，交由内核决定起始地址。

length：申请的内存有多大，决定匿名映射的物理内存有多大或文件映射的文件区域有多大。addr和length必须按照PAGE_SIZE对齐。

如果是文件映射，就要通过参数fd指定要映射文件的描述符，通过offset指定文件映射区在文件的偏移。Linux中内存页和磁盘块大小一般情况都是4KB，这里的offset也必须按照4KB对齐。

mmap映射的虚拟内存在内核中用struct vm_area_struct结构表示，进程中的VMA有两种组织形式，双向链表和红黑树。mmap系统调用本质是先要在虚拟内存空间中划分出一段VMA出来，这段VMA区域的大小由vm_start，vm_end表示，而它们由mmap参数addr、length决定；随后内核会对这段VMA进行相关的映射，如果是文件映射，内核会将映射的文件以及要映射文件区域在文件中的offset，与VMA中的vm_file、vm_pgoff关联映射起来，他们由mmap参数fd、offset决定；mmap映射的这段VMA中的相关权限和标志位，是由mmap参数的prot、flags决定的，最终会映射到VMA中的vm_page_prot、vm_flags中，指定进程对这块VMA的访问权限和相关标志位。PS：进程所依赖的动态链接库.so文件也是通过mmap文件映射的方式将代码段、数据段映射到文件映射与匿名映射区中。

mmap中参数prot指定VMA的访问权限，取值有四种：

```C
#define PROT_READ 0x1  /* page can be read */
#define PROT_WRITE 0x2  /* page can be written */
#define PROT_EXEC 0x4  /* page can be executed */
#define PROT_NONE 0x0  /* page can not be accessed */
```
mmap中参数flags指定VMA映射方式，OS对物理页的管理类型有两种：一种是匿名页、一种是文件页，对应的映射也就分为两种，匿名映射、文件映射，mmap所映射的物理内存能否在多进程间共享又分为共享映射、私有映射两种映射方式，共享映射是多进程共享的，一个进程修改了共享映射的内存其他进程是可以看到的，用于多进程间的通信；私有映射是进程私有的，其他进程看不到，多进程修改同一映射文件将不会回写到磁盘文件上。

按照匿名、文件与共享、私有进行组合，就有四种映射方式。

1. 私有匿名映射：MAP_PRIVATE | MAP_ANONYMOUS，glib库中封装的malloc函数申请内存大于128KB时使用mmap私有匿名映射方式来申请堆内存。这里mmap只是先申请一段VMA，还没有真正分配物理内存，PTE是空的，只有读写此区域时，才会触发缺页异常分配物理内存。私有匿名映射除了用于申请虚拟内存之外，还会用于execve系统调用中，内核需要删除释放旧的虚拟内存空间并情况页表，然后打开可执行文件，解析文件头，判断可执行文件的格式，对应函数进行加载。这个过程就需要私有匿名映射创建新的虚拟内存空间中的BSS段、堆和栈。
{{< image src="/xv6/私有匿名映射.png" caption="私有匿名映射" >}}

1. 私有文件映射：MAP_PRIVATE，mmap内存文件映射的本质是vm_area_struct结构体中传入参数fd所对应的file结构体指针，指向file结构体，file结构体中存有inode结构体成员，即该文件对应的inode索引，从而找到对应的磁盘块block。和私有匿名映射一样，它不会立即分配物理页面，而是利用缺页异常，进程1在建立PTE之后，再次访问这段文件内存映射时就相当于直接访问文件的page cache，这个过程是在用户态的，没有切态。进程2同样在访问这段文件时也触发缺页异常，建立PTE，但会直接指向进程1中的page cache。所有进程的PTE都设为只读，当任一进程试图写入时，又会触发缺页中断，会申请一个内存页，然后将page cache的内容拷贝到新内存页中，并更新PTE。当进程都有各自专属的物理内存页时，就和page cache脱离关系了，各自的修改在进程之间时互不可见的，且均不会回写到磁盘文件中。可执行文件的.text、.data就使用私有文件映射到了进程虚拟内存空间中的代码段和数据段中。
{{< image src="/xv6/私有文件映射多进程.png" caption="私有文件映射多进程" >}}
{{< image src="/xv6/私有文件映射进程空间.png" caption="私有文件映射进程空间" >}}

3. 共享文件映射：MAP_SHARED，和私有文件映射过程一样，唯一不同的点是，多进程中的虚拟内存映射区通过缺页中断会映射到同一page cache中，且是可读可写，不会再次触发缺页中断去各自分配新的物理页。多进程对共享映射区的任何修改都会通过内核回写线程pdflush刷新到磁盘文件中。根据 mmap 共享文件映射多进程之间读写共享（不会发生写时复制）的特点，常用于多进程之间共享内存（page cache），多进程之间的通讯。
{{< image src="/xv6/共享文件映射.png" caption="共享文件映射" >}}

4. 共享匿名映射：MAP_SHARED | MAP_ANONYMOUS，将fd设为-1来实现共享匿名映射，这种映射方式常用于父子进程之间共享内存，父子进程之间的通讯。这个思路如果按照上述的话，就是进程1缺页中断分配一个物理页，进程2同样缺页中断要分配进程1所指的物理页，但是进程2怎么找到这个物理页？找不到的。但这对于共享文件映射很容易，因为有文件的page cache存在，进程2可以根据offset从page cache中查找是否已经有其他进程把映射的文件内容加载到文件页中。如果文件页已经存在 page cache 中了，进程 2 直接映射这个文件页就可以了。共享匿名映射在内核中是通过一个叫做 tmpfs 的虚拟文件系统来实现的，tmpfs 不是传统意义上的文件系统，它是基于内存实现的，挂载在 dev/zero 目录下。当多个进程通过 mmap 进行共享匿名映射的时候，内核会在 tmpfs 文件系统中创建一个匿名文件，这个匿名文件并不是真实存在于磁盘上的，它是内核为了共享匿名映射而模拟出来的，匿名文件也有自己的 inode 结构以及 page cache。在 mmap 进行共享匿名映射的时候，内核会把这个匿名文件关联到进程的虚拟映射区 VMA 中。这样一来，当进程虚拟映射区域与 tmpfs 文件系统中的这个匿名文件映射起来之后，后面的流程就和共享文件映射一模一样了。由于是基于内存实现的虚拟文件系统，在其他进程是不可见的，而子进程是可见的，适用于父子进程间的共享匿名映射。

总结：
1. 私有匿名映射，其主要用于进程申请虚拟内存，以及初始化进程虚拟内存空间中的 BSS 段，堆，栈这些虚拟内存区域。
2. 私有文件映射，其核心特点是背后映射的文件页在多进程之间是读共享的，多个进程对各自虚拟内存区的修改只能反应到各自对应的文件页上，而且各自的修改在进程之间是互不可见的，最重要的一点是这些修改均不会回写到磁盘文件中。我们可以利用这些特点来加载二进制可执行文件的 .text , .data section 到进程虚拟内存空间中的代码段和数据段中。
3. 共享文件映射，多进程之间读写共享（不会发生写时复制），常用于多进程之间共享内存（page cache），多进程之间的通讯。
4. 共享匿名映射，用于父子进程之间共享内存，父子进程之间的通讯。父子进程之间需要依赖 tmpfs 中的匿名文件来实现共享内存。是一种特殊的共享文件映射。

xv6的mmap函数声明：
```C
void* mmap(void *addr, size_t len, int prot, int flags, int fd, off_t offset);
```
可以看出和上述Linux的mmap函数声明是一致的，本实验就是要实现类似于Unix的上述简化实现。只实现文件映射部分，没有匿名映射。实验说明：
1. 内核自己决定文件映射的虚拟地址，mmap返回该地址，失败返回0xffffffffffffffff。
2. len是映射的字节数。
3. prot有可读、可写、可执行三种。
4. flags是MAP_PRIVATE或MAP_SHARED，即私有文件映射或共享文件映射，就是上述所提的2、3两种情况。
5. 在usertrap()中page fault进行处理分配物理内存。
6. MAP_SHARED可以不分配物理内存。
munmap函数声明：
```C
int munmap(void *addr, size_t len);
```
1. 移除指定地址范围内的mmap映射。
2. MAP_SHARED要将修改写入文件，进程退出时也要对MAP_SHARED的任何修改写入文件。
   
hints：
1. 按照之前的老方法添加mmap和munmap系统调用。
2. 定义VMA结构体，记录映射的虚拟地址空间的起始地址、长度、权限、文件等信息。
3. 声明一个大小为16的VMA数组。
4. 把VMA数组对应到一个未使用的区域。
5. VMA结构体中应包含一个指向file结构体的指针。
6. mmap应增加文件的引用计数。
7. VMA区域引发的page fault进行处理，分配一页物理内存，使用readi读入文件的4KB并映射到用户地址空间，设置正确的页面权限。
8. 使用uvmunmap取消对应地址范围的映射，并减少相应file的引用计数，如果页面MAP_SHARED，需要将其写回文件，参考filewrite。
9. 修改exit()取消进程的映射区域，就像调用munmap一样。
10. 修改fork()确保父子进程有相同的映射区域，增加VMA的引用计数，子进程的page fault可以分配一个新的物理页，不与父进程共享。

注意通过mmap对映射文件的写操作不能超过文件大小。mmap参数length大小是2页但是文件只有1.5页怎么办？

代码实现：

老办法添加sys_mmap和sys_munmap两个系统调用的函数声明，这里不再赘述。

添加VMA结构体，并声明一个大小为16的VMA结构体数组。参照mmap的函数参数，结构体成员变量需要有起始地址、结束地址、映射长度、权限、类型、文件描述符、偏移量、对应的映射文件这些，此外加入valid变量表示这块vma区域是否映射到了对应的文件。
```C
void* mmap(void *addr, size_t len, int prot, int flags, int fd, off_t offset);
```
```C
// proc.h
struct vma {
  int valid;  // 是否被映射
  uint64 start;  // 起始地址
  uint64 end;    // 结束地址
  uint64 length; // 映射长度
  int prot;   // 权限
  int flags;  // 类型
  int fd;     // 文件
  int offset; // 偏移量

  struct file *file; // 指向对应的映射文件
};

#define NVMA 16

// Per-process state
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
  struct vma vmas[NVMA];       // mmap virtual memory area
};
```
实现sys_mmap函数，读入传入的参数，注意使用argfd同时读入fd对应的file结构体，遍历vma数组找一块未被映射的vma，然后通过vmaend()去为它分配一段空闲的虚拟内存区域。这里我们将vma放到堆区之上，TRAPFRAME之下，从高地址向低地址扩展。
```C
// sysfile.c
uint64
sys_mmap(void) {
  uint64 addr;
  int length; // 长度
  int prot;   // 权限
  int flags;  // 类型
  int fd;     // 文件
  int offset; // 偏移量

  struct file* file;
  argaddr(0, &addr);
  argint(1, &length);
  argint(2, &prot);
  argint(3, &flags);
  argfd(4, &fd, &file);
  argint(5, &offset);
  if (addr < 0 || length < 0 || prot < 0 || flags < 0 || fd < 0 || offset < 0) {
    return -1;
  }

  // 如果是 MAP_SHARED 文件必须可写
  if (flags & MAP_SHARED && prot & PROT_WRITE && !file->writable) {
    return -1;
  }

  // 找一个未被映射的VMA
  struct vma *v = 0;
  struct proc *p = myproc();
  for (int i = 0; i < NVMA; i++) {
    if (p->vmas[i].valid == 0) {
      // 未被映射的VMA
      v = &p->vmas[i];
      break;
    }
  }
  if (v == 0) return -1;

  // 找一片区域给vma
  uint64 end = vmaend();
  // 初始化vma
  v->valid = 1;
  v->start = end - length;
  v->end = end;
  v->length = length;
  v->prot = prot;
  v->flags = flags;
  v->fd = fd;
  v->file = file;
  v->offset = offset;

  // 增加文件的引用计数
  filedup(file);

  return v->start;
}
```
所以我们仍需遍历所有vma，找位于最低区域的vma，将新的vma放到它下面。具体就是比较v->start地址，找最低地址作为新vma的最高地址v->end。
```C
// sysfile.c
uint64
vmaend() {
  // 从TRAPFRAME向下分配一片一片的vma
  // 遍历所有已分配的vma 找最低地址作为新分配的end地址
  struct proc *p = myproc();
  uint64 minstart = TRAPFRAME;
  struct vma *v = 0;
  for (int i = 0; i < NVMA; i++) {
    if (p->vmas[i].valid && p->vmas[i].start <= minstart) {
      minstart = p->vmas[i].start;
      v = &p->vmas[i];
    }
  }

  if (v == 0) return minstart;
  return PGROUNDDOWN(v->start);
}
```
此时我们建立了vma虚拟内存到文件结构体file的映射关联，但是并没有分配物理页，而是依赖延时分配的机制在page fault时才去分配物理页，读入file对应的文件块。所以需要在usertrap中实现以上page fault的处理。
```C
// trap.c
// ...
 else if((which_dev = devintr()) != 0){
  // ok
} else if (r_scause() == 13 || r_scause() == 15) {
  // page fault
  if (mmapfault(r_stval()) != 0) {
    printf("usertrap(): mmapfault scause 0x%lx pid=%d\n", r_scause(), p->pid);
    printf("            sepc=0x%lx stval=0x%lx\n", r_sepc(), r_stval());
    setkilled(p);
  }
}
```
mmapfault这里需要注意一点，如果是写错误（r_scause() == 15），要判断文件权限、映射类型是否是只读的。
```C
// trap.c
int
mmapfault(uint64 addr) {

  struct proc *p = myproc();
  struct vma *v = 0;
  // 找addr对应的vma
  for (int i = 0; i < NVMA; i++) {
    if (p->vmas[i].valid && p->vmas[i].start <= addr && addr < p->vmas[i].end) {
      v = &p->vmas[i];
      break;
    }
  }
  if (v == 0) {
    return -1;
  }

  // 不能对只读文件进行写操作
  // r_scause() == 15 是写内存出错 r_scause() == 13 是读内存出错
  if (r_scause() == 15 && v->prot & PROT_READ && !(v->prot & PROT_WRITE) && v->flags & MAP_SHARED) {
    return -1;
  }

  // 分配物理页
  char *pa = kalloc();
  if (!pa) {
    return -1;
  }
  memset(pa, 0, PGSIZE);

  // 读取文件内容到物理内存中
  uint offset = v->offset + PGROUNDDOWN(addr) - v->start;
  ilock(v->file->ip);
  if (readi(v->file->ip, 0, (uint64)pa, offset, PGSIZE) == 0) {
    iunlock(v->file->ip);
    return -1;
  }
  iunlock(v->file->ip);

  // 设置页面权限
  uint perm;
  perm = PTE_V | PTE_U;
  if (v->prot & PROT_READ) perm |= PTE_R;
  if (v->prot & PROT_WRITE) perm |= PTE_W;
  if (v->prot & PROT_EXEC) perm |= PTE_X;

  // 设置映射
  mappages(p->pagetable, PGROUNDDOWN(addr), PGSIZE, (uint64)pa, perm);

  return 0;
}
```
至此完了mmap的部分，接下来实现munmap。munmap除了释放映射，还要判断MAP_SHARED进行写回。先读入传入的参数，这里使用函数munmap来封装具体操作，因为后边所提到的exit中也是同样的操作，可复用。
```C
// sysfile.c
uint64
sys_munmap(void) {
  uint64 addr;
  int length;

  argaddr(0, &addr);
  argint(1, &length);
  if (addr < 0 || length < 0) {
    return -1;
  }

  return munmap(addr, length);
}
```
munmap找到对应的vma区域并调用writeback函数释放映射并写回文件，提示中说到，munmap的范围有三种类型，从start开始释放、释放到end、start到end区域全释放。所以需要更新对应的vma，尤其是从start释放部分映射时，对应的文件偏移量offset也要相应增加，不然mmaptest中的部分测试用例是有问题的，比如释放了前一页，再释放后半页，最后读取文件时的首个字符却是后半页的首字符的情况，这是因为没有更新对应文件的offset，而把后半页写回到了文件的首页中。
```C
// sysfile.c
uint64
munmap(uint64 addr, int length) {

  struct proc *p = myproc();
  struct vma *v = 0;
  // 找对应的vma区域
  for (int i = 0; i < NVMA; i++) {
    if (p->vmas[i].valid && p->vmas[i].start <= addr && addr < p->vmas[i].end) {
      v = &p->vmas[i];
      break;
    }
  }
  if (v == 0) {
    return -1;
  }

  // unmap并写回文件
  if (writeback(p->pagetable, addr, length, v) != 0) {
    return -1;
  }

  if (addr == v->start) { // 从start释放
    v->start += length;
    v->offset += length; // 注意 文件也要对应偏移 否则部分释放有问题
  }else if (addr == v->end - length) { // 释放到end
    v->end = addr;
  }
  v->length -= length;

  // 减少file引用计数
  if (v->length <= 0) {
    fileclose(v->file);
    v->valid = 0;
  }
  return 0;
}
```
writeback函数参照filewrite，调用writei进行文件写入。这里也要注意写入文件不能超过文件原本的大小，mmap映射的文件大小已经固定，不能扩充文件。
```C
// sysfile.c
int
writeback(pagetable_t pgtbl, uint64 va, int length, struct vma* v) {

  pte_t* pte;
  uint64 addr;
  // 遍历vma区域页面
  for (addr = PGROUNDDOWN(va); addr < PGROUNDDOWN(va+length); addr += PGSIZE) {
    // 获取对应PTE
    if ((pte = walk(pgtbl, addr, 0)) == 0) {
      return -1;
    }
    if (va - v->start > v->file->ip->size) break;
    // 未分配物理页
    if (!(*pte & PTE_V)) continue;
    // 如果是MAP_SHARED 且可写 要写回文件
    // hint说明可以不考虑PTE_D
    if (v->flags & MAP_SHARED && v->prot & PROT_WRITE) {
      begin_op();
      ilock(v->file->ip);
      uint offset = v->offset + addr - v->start;
      // 写入不能超过原文件大小
      uint size = v->file->ip->size - offset;
      if (size > PGSIZE) size = PGSIZE;
      writei(v->file->ip, 1, addr, offset, size);
      iunlock(v->file->ip);
      end_op();
    }
    // 释放物理页及映射
    kfree((void*)PTE2PA(*pte));
    *pte = 0;
  }
  return 0;
}
```
最后实现fork时的子进程也要copy父进程的vma区域，以及exit子进程退出时执行munmap操作。
```C
// proc.c
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

  for (int i = 0; i < NVMA; i++) {
    if (p->vmas[i].valid) {
      np->vmas[i] = p->vmas[i];
      filedup(p->vmas[i].file);
    }
  }
  
  // ...
}
```
```C
// proc.c
void
exit(int status)
{
  struct proc *p = myproc();

  if(p == initproc)
    panic("init exiting");

  // 释放vma映射
  for (int i = 0; i < NVMA; i++) {
    if (p->vmas[i].valid) {
      if (munmap(p->vmas[i].start, p->vmas[i].length) != 0) {
        panic("exit munmap");
      }
    }
  }
  
  // ...
}
```
至此，mmaptest全部通过。
{{< image src="/xv6/mmaptest.png" caption="mmaptest" >}}

总结：mmap作为xv6的最后一个实验，综合了内存管理、进程管理、文件系统这三大块的内容，是一个综合性极佳的实验。首先认识到了为什么需要虚拟内存这一问题的答案，除了实现进程隔离防止多进程访问地址冲突、给进程提供一个看起来更大的地址空间以外，他还可以在此基础之上利用page fault陷入trap处理程序，从而实现延时分配（先提供虚拟空间，在访问时再去分配物理空间，比如sbrk扩充了虚拟地址空间大小，在访问扩充后的虚拟地址时才会实际分配物理页，这里是根本没有建立PTE映射，也就是没有PTE的page fault）、写时复制（fork时子进程copy父进程的页表，指向同一块虚拟地址空间，但将PTE设为不可写，当其中一个进程试图写入时，触发page fault再去分配物理页，copy原来的物理页，这里是建立了PTE，但权限为不可写）机制。以及去映射文件避免读写文件多次地从用户态到内核态的上下文切换（也就是本实验实现的，本质上是使用该系统调用将文件内容读到内存中，从而再次读写文件时不用频繁调用read、write系统调用）。

PS：强烈推荐文中开头提到的文章，mmap这个概念其实一直困扰了我很久，就像文中说到的：
{{< image src="/xv6/文件映射的疑问.png" caption="文件映射的疑问" >}}

我脑子中就全是这些疑惑，看到这段话时，顿时感到终于有人懂我了。这篇文章真是跪着读完的，相比于网上查阅的大多数资料、gpt答案，简直是好太多，醍醐灌顶的感觉。

以及网上参考的实验实现可能在xv6 2024版本的实验中有些已经过不了了，推测是这些老头在不断更新测试用例，查看test文件的代码很有帮助，比如mmaptest中过不了的测试可以回查源码，看它是在测试哪方面，对应哪里出了问题，能快速定位解决。

参考：
1. {{< link href="https://mp.weixin.qq.com/s/AUsgFOaePwVsPozC3F6Wjw" content="从内核世界透视 mmap 内存映射的本质（原理篇）">}}
2. {{< link href="https://kerolt.work/posts/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/mit6.s081lab10-mmap/" content="kerolt的实现">}}


---

> 作者: [UAreFree](https://github.com/UAreFree)  
> URL: http://localhost:1313/posts/mit-lab-mmap/  

