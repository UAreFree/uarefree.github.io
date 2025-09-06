# MIT6.1810 Lab:System Calls


MIT6.1810 lab xv6 2024 syscall

<!--more--> 

## System call tracing
实现系统调用trace，trace接受一个参数（整数掩码），如果系统调用号在掩码中设置，需要修改xv6内核，使其在每个系统调用即将返回时打印一行（进程ID、系统调用名称、返回值）。

用户程序需要调用system call，那么所提供的system call的函数声明在user.h，而实际的定义实现却是在kernel中，要知道用户态的代码和内核态的代码是隔离的，那么用户程序是如何调用内核的代码实现的呢？答案是user.pl，user.pl是在用户空间的脚本，作为桥梁，在makefile时生成user.S，这个汇编代码include了syscall.h，syscall.h声明了系统调用号（对应kernel system call 函数），把这个编号放到寄存器a7中，然后ecall进内核，在ecall时会将a7寄存器的值存到trapframe中，从而传到内核中，这样内核从trapframe中加载a7寄存器的值，从而调用对应的system call。

本实验是要实现system call，但是需要shell去测试验证，所以依旧需要先实现一个用户程序trace，在这个程序中调用system call，这里源代码已经提供了trace用户程序。需要增加的是上述关于调用内核system call的操作。

ecall会调用trampoline.S代码，这段代码会执行内核的usertrap函数，当r_scause() == 8时会调用syscall()，syscall()会取a7寄存器的值，然后调用对应syscall函数数组索引的系统调用函数，返回值存在trapframe的a0寄存器里。所以就在syscall()里判断是否是要跟踪的系统调用号，如果是，打印进程ID、系统调用名称和返回值。

整体实现过程就是：1. 添加新增的与syscall有关的函数存根、声明等。2. 在syscall()中拿到用户参数掩码，与当前系统调用号进行按位与运算，如果是1则打印。3. 注意根据提示，增加系统调用名称数组以及在fork时将父进程的mask也copy到子进程中。

主要的代码实现：

```C
// kernel/syscall.c
// ...
// Prototypes for the functions that handle system calls.
extern uint64 sys_fork(void);
extern uint64 sys_exit(void);
extern uint64 sys_wait(void);
extern uint64 sys_pipe(void);
extern uint64 sys_read(void);
extern uint64 sys_kill(void);
extern uint64 sys_exec(void);
extern uint64 sys_fstat(void);
extern uint64 sys_chdir(void);
extern uint64 sys_dup(void);
extern uint64 sys_getpid(void);
extern uint64 sys_sbrk(void);
extern uint64 sys_sleep(void);
extern uint64 sys_uptime(void);
extern uint64 sys_open(void);
extern uint64 sys_write(void);
extern uint64 sys_mknod(void);
extern uint64 sys_unlink(void);
extern uint64 sys_link(void);
extern uint64 sys_mkdir(void);
extern uint64 sys_close(void);
extern uint64 sys_trace(void); // 在外部.c文件中的sys_trace

// An array mapping syscall numbers from syscall.h
// to the function that handles the system call.
static uint64 (*syscalls[])(void) = { // 函数指针数组
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
  [SYS_trace] sys_trace,
};

char* sysnames[] = {
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
    [SYS_trace] "trace",
};

void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    // Use num to lookup the system call function for num, call it,
    // and store its return value in p->trapframe->a0
    p->trapframe->a0 = syscalls[num]();
    if ((p->mask & (1 << num)) == (1 << num)) {
      // 代码中对uint64会赋值-1 导致打印a0时出现问题 make grade Test trace children过不了
      if (p->trapframe->a0 == -1) { // 这里单独增加了判断
        printf("%d: syscall %s -> %d\n", p->pid, sysnames[num], -1);
      }else {
        printf("%d: syscall %s -> %lu\n", p->pid, sysnames[num], p->trapframe->a0);
      }
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```
```C
// kernel/sysproc.c
// ...
// 和进程相关的系统调用实现放在此文件里
uint64
sys_trace(void) { // 注意这里的传入参数全是void
  // 因为使用argint内核函数来获取放到trampframe中的寄存器的值
  // 用户程序trace有传入一个参数 int
  int n;
  argint(0, &n); // 提取第一个用户参数
  struct proc *p = myproc();
  p->mask = n;
  return 0;
}
```
```C
// kernel/proc.c
// ...
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

  // copy saved user registers.
  *(np->trapframe) = *(p->trapframe);

  // Cause fork to return 0 in the child.
  np->trapframe->a0 = 0;

  np->mask = p->mask; // copy trace mask

  // increment reference counts on open file descriptors.
  for(i = 0; i < NOFILE; i++)
    if(p->ofile[i])
      np->ofile[i] = filedup(p->ofile[i]);
  np->cwd = idup(p->cwd);

  safestrcpy(np->name, p->name, sizeof(p->name));

  pid = np->pid;

  release(&np->lock);

  acquire(&wait_lock);
  np->parent = p;
  release(&wait_lock);

  acquire(&np->lock);
  np->state = RUNNABLE;
  release(&np->lock);

  return pid;
}
```
## Attack xv6
用户程序与内核程序是隔离的，两者交互只能通过系统调用实现。而如果系统调用的实现存在bug，攻击者就可以通过这个bug突破隔离边界。
本实验时的bug是注释掉memset(mem, 0, sz)，新分配的内存保留了先前使用的内容。

attacktest的流程是：随机生成8字节的字符串秘钥，通过secret.c写入内存，再发起attack.c窃取秘钥并将其写入文件描述符2中。要实现的是attack.c，即怎样利用清除新分配页面时保留先前使用的页面内容的漏洞来实现。

基本思路是将secret.c新分配的内存释放掉，然后attack.c请求新的内存，从而得到上次未清除页面内容的新页面，其中就含有秘钥。因为secret是作为子进程运行后exit了，所以其资源包括内存会被释放掉，但是在物理内存上写的内容并没有被擦除。核心是找到子进程secret所写的那个物理页，然后在attack中去读这个物理页的内容。

如何找到秘钥所在的物理页？首先在secret子进程结束后执行exit，父进程会wait回收子进程的资源，在wait中调用了freeproc(pp);在freeproc(pp)中调用到了proc_freepagetable(p->pagetable, p->sz);

```C
void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable, TRAMPOLINE, 1, 0); // 解除从TRAMPOLINE开始的1页映射 不释放该页物理内存
  uvmunmap(pagetable, TRAPFRAME, 1, 0); // 解除从TRAPFRAME开始的1页映射 不释放该页物理内存
  uvmfree(pagetable, sz); // 解除从0到虚拟内存最大地址下的所有页映射 释放对应的物理内存 递归释放页表内存页
}
```

这段代码进行了内存释放相关的操作，在uvmfree(pagetable, sz);中调用了uvmunmap(pagetable, 0, PGROUNDUP(sz)/PGSIZE, 1);而uvmunmap中会调用kfree((void*)pa);将页表中虚拟页对应的物理页释放掉。具体的释放操作是把物理页还到freelist上，freelist是内核维护的空闲物理内存页链表，kfree会把还的物理页挂到freelist最前边。

```C
void
kfree(void *pa) // 释放pa地址对应的物理内存页
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");


#ifndef LAB_SYSCALL
  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);
#endif
  
  r = (struct run*)pa; // run链表 当前物理页

  acquire(&kmem.lock);
  r->next = kmem.freelist; // 把当前物理页挂到freelist最前边
  kmem.freelist = r;
  release(&kmem.lock);
}
```
具体的分配、释放freelist的物理页较为复杂，最后秘钥页是在第17页，具体可参考这个{{<link href="https://www.youtube.com/watch?v=8wq1BcXhjp4" content=PPT讲解 >}}。对应代码实现：

```C
int
main(int argc, char *argv[])
{
  // your code here.  you should write the secret to fd 2 using write
  // (e.g., write(2, secret, 8)
  char *end = sbrk(17 * PGSIZE);
  end = end + 16 * PGSIZE;
  write(2, end + 32, 8);
  exit(1);
}
```

---

> 作者: [UAreFree](https://github.com/UAreFree)  
> URL: http://localhost:1313/posts/mit-lab-syscall/  

