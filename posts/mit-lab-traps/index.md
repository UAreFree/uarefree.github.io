# MIT6.1810 Lab:Traps


MIT6.1810 lab xv6 2024 trap
<!--more--> 

## RISC-V assembly
1. 哪些寄存器包含函数的参数？比如，哪个寄存器保存printf函数中的13？
   
    a0、a1、a2寄存器保存函数参数。a2寄存器保存printf函数中的13。
2. f函数调用在哪里？g函数调用在哪里？
   
    没有具体的调用，实际都做了函数内联优化，直接执行x+3。
3. 函数printf位于哪个地址？
   
    位于0x6bc地址。
```C
    30: 68c000ef           jal    6bc <printf>
```
4. 在printf的jalr之后，ra寄存器的值是多少？
   
    0x34，执行main中下一条指令。
5. 代码输出是什么？如果RISC-V是大端序，你会将i设为多少才能得到相同的输出？你会将57616改为其他不同的值吗？
```C
        unsigned int i = 0x00646c72;
        printf("H%x Wo%s", 57616, (char *) &i);
```
输出：HE110 World。强转为char*是指从变量i的地址处按字节读取变量i，如果不取i的地址，那就是在0x00646c72地址处按字节读取变量i，这是错的。当前是小端序，低字节低地址，所以按0x72、0x6c、0x64、0x00的顺序读，转换为rld。若是大端序，则将i设为0x726c6400才能得到相同的输出。57616不需要变，因为对应的十六进制不会改变。

6. 代码“y=”之后会打印什么？
```C
printf("x=%d y=%d", 3);
```
打印的是不确定的值，printf会错误读取栈上某个随机的值，是未定义行为。

## Backtrace
error发生后定位其函数调用链进行回溯。编译器生成了包含当前调用链每个函数栈帧的机器码，每个栈帧包含返回地址和指向调用它的栈帧指针。s0寄存器包含一个指向当前栈帧的指针。讲义中的这张图清晰的展示了栈帧结构，从s0寄存器得到栈帧指针fp，然后在fp-8的位置是返回地址，在fp-16的位置是保存的上一个调用它的fp。
{{< image src="/xv6/栈帧结构.png" caption="栈帧结构" >}}
思路就是循环遍历fp的值，从而读取对应栈帧的返回地址并打印。什么时候循环停止，提示说所有栈帧都在同一页上，所以超出该页范围即停止，实现代码：
```C
void
backtrace(void) {
  uint64 fp = r_fp();
  uint64 fpmin = PGROUNDDOWN(fp);
  uint64 fpmax = PGROUNDDOWN(fp) + PGSIZE;
  while (fp < fpmax && fp >= fpmin) {
    printf("%p\n", (uint64 *)*(uint64 *)(fp-8));
    fp = *(uint64*)(fp-16);
  }
}
```
## Alarm
添加一个新的sigalarm(interval, handler)系统调用，应用程序调用sigalarm(n, fn)时，则在程序消耗CPU时间的每n个ticks时调用应用程序函数fn。这里alarmtest划分了四个test，一个一个来做。
首先就是添加sigalarm和sigreturn两个系统调用，参照syscall实验，主要是得到用户参数，传递到proc结构中，新建的计数变量ticks和中断处理程序handler函数指针。sys_sigreturn在这个test先直接return 0;即可。
```C
uint64
sys_sigalarm(void) {
  int ticks;
  uint64 handler_addr;
  argint(0, &ticks);
  argaddr(1, &handler_addr);
  myproc()->ticks = ticks;
  myproc()->handler = (void (*)())handler_addr;
  return 0;
}
```
因为每次定时器中断都会在usertrap中进行处理，所以在处理函数中增加统计tick数量，若达到ticks即执行用户程序handler，这里需要将sepc寄存器的值设为handler的地址，从而在trap返回时指向handler程序的地址。这样就完成了test0。
```C
if(which_dev == 2) {
  p->passticks ++;
  if (p->ticks > 0 && p->passticks == p->ticks) {
    p->passticks = 0;
    // 返回用户程序handler地址处
    p->trapframe->epc = (uint64)p->handler;
  }
  yield();
}
```
test1要求在从handler返回时，要恢复中断之前的寄存器的值，类比于trapframe保存寄存器的值，这里新增另一个atrapframe来保存中断之前的寄存器，在sigreturn时再进行恢复。
```C
if(which_dev == 2) {
  p->passticks ++;
  if (p->ticks > 0 && p->passticks == p->ticks) {
    // 把定时器中断前的寄存器保存到atrpframe中
    memmove(p->atrapframe,p->trapframe,sizeof(struct trapframe));
    p->passticks = 0;
    // 返回用户程序handler地址处
    p->trapframe->epc = (uint64)p->handler;
  }
  yield();
}
```
```C
uint64
sys_sigreturn(void) {
  struct proc* p = myproc();
  // 恢复定时器中断前的寄存器 如果不恢复此时寄存器值是用户handler程序的寄存器值 原返回程序的寄存器值被覆盖所以寄
  memmove(p->trapframe,p->atrapframe,sizeof(struct trapframe));
  return 0;
}
```
test2要求不能重复执行中断处理程序，即在进程结构体中增加标志位表示当前是否在执行handler，初始为0，在调用handler时设为1，sigreturn时设为0。
```C
if(which_dev == 2) {
  p->passticks ++;
  if (p->ticks > 0 && p->passticks == p->ticks && p->alarmflag == 0) {
    // 把定时器中断前的寄存器保存到atrpframe中
    memmove(p->atrapframe,p->trapframe,sizeof(struct trapframe));
    p->passticks = 0;
    // 返回用户程序handler地址处
    p->trapframe->epc = (uint64)p->handler;
    // 此进程在执行handler
    p->alarmflag = 1;
  }
  yield();
}
```
```C
uint64
sys_sigreturn(void) {
  struct proc* p = myproc();
  // 恢复定时器中断前的寄存器 如果不恢复此时寄存器值是用户handler程序的寄存器值 原返回程序的寄存器值被覆盖所以寄
  memmove(p->trapframe,p->atrapframe,sizeof(struct trapframe));
  p->alarmflag = 0;
  return 0;
}
```
test3要求sigreturn返回时a0寄存器的值要是中断之前的a0寄存器的值，sigreturn返回0会将a0设为0，所以这里直接返回保存的trapframe中的a0的值。
```C
uint64
sys_sigreturn(void) {
  struct proc* p = myproc();
  // 恢复定时器中断前的寄存器 如果不恢复此时寄存器值是用户handler程序的寄存器值 原返回程序的寄存器值被覆盖所以寄
  memmove(p->trapframe,p->atrapframe,sizeof(struct trapframe));
  p->alarmflag = 0;
  return p->trapframe->a0;
}
```
以上所有在proc.h中新增的结构体成员，其生命周期需要在proc.c中添加对应代码，主要体现在allocproc.c函数及freeproc.c函数中，这里没有给出。至此，alarmtest所有test都通过。




---

> 作者: [UAreFree](https://github.com/UAreFree)  
> URL: http://localhost:1313/posts/mit-lab-traps/  

