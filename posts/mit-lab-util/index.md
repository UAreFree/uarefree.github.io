# MIT6.1810 Lab:Xv6 and Unix Utilities


MIT6.1810 lab xv6 2024 util

<!--more--> 

## Boot xv6
首先是搭建环境，这里使用CLion + WSL2的环境进行开发，git克隆代码仓库：
```Shell
git clone git://g.csail.mit.edu/xv6-labs-2024
```
将项目放到了github中进行版本管理，remove掉了原远程仓库，ssh连接到自己新建的仓库，push上去就可以。
```Shell
git remote remove origin
```
安装 lab tools ，我这里使用的是Ubuntu-22.04，安装的QEMU是6.0系列，这里用不了。提示说需要QEMU 7.2+、GDB 8.3+，对应的Ubuntu-24之后，汗流浃背了。搜了一下WSL2还能同时安装不同版本的Ubuntu，于是下了Ubuntu-24.04，make qemu后正常运行，使用ctrl-a x退出终端。

此外，CLion在make后会提示没有all这个目标的构建错误，使用WSL为默认的编译工具链，并修改makefile设置，将target设为qemu即可，解析完后此时是能实现代码跳转的。
{{< image src="/xv6/CLion配置.png" caption="CLion配置" >}}

配置CLion gdb debug参考这个{{< link href="https://juejin.cn/post/7448512834663661579" content=链接 title="CLion xv6" >}}的方法。

注意请勿随意修改.git目录和原origin指向的链接，所以上述remove原origin仓库是不合适的，应该是添加github仓库，然后分支push，这样就能将原仓库代码保存到自己github上进行版本管理。

```Shell
git remote add github git@github.com:UAreFree/xv6-labs-2024.git

git push github util:util
```

## Sleep
题意：实现用户级别的程序sleep，即暂停指定的时间，这里的时间刻度是定时器中断时间间隔。

提示：第一反应是这个定时器中断时间间隔怎么实现，这里提示使用sleep系统调用，那就不需要考虑了，整体应该是考察如何实现user中的shell程序。

在Makefile中添加用户级程序，$U 是用户程序的目录，每个程序对应一个.c或.S文件，经过编译后生成对应的.o文件。
```Makefile
UPROGS=\
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
    $U/_sleep\
```
新建sleep.c文件，代码实现：
```C
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
    // 用户忘记传参数 打印错误信息
    if(argc <= 1){
        fprintf(2, "usage: sleep [int]\n");
        exit(1);
    }
    // 使用sleep系统调用
    sleep(atoi(argv[1]));
    exit(0);
}
```
运行`./grade-lab-util sleep`进行代码测试。

## Pingpong
依旧用户程序，通过一对管道在两个进程间“乒乓”发送一个字节，父进程向子进程发送一个字节，子进程打印"<pid>: received ping"，并将该字节通过管道写入父进程，然后退出；父进程从子进程读取该字节，打印"<pid>: received pong"。

pipe管道p[0]是读，p[1]是写，在读管道时，关闭写端；在写管道时，关闭读端。这里用了两个管道（一对），对每个管道按照上述原则进行读写就可以了，以及注意exit和wait在父子进程中的使用。

新建pingpong.c文件，代码实现：
```C
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
    int pipe1[2], pipe2[2];
    pipe(pipe1);
    pipe(pipe2);
    char c = 'p';
    // 父进程向子进程发送一个字节 通过管道
    int pid = fork();
    if (pid < 0) {
        fprintf(2, "fork failed\n");
        exit(1);
    }
    if (pid == 0) {
        // 从pipe1里读
        close(pipe1[1]);
        read(pipe1[0], &c, 1);
        close(pipe1[0]);
        fprintf(1,"%d: received ping\n", getpid());
        // 向pipe2里写
        close(pipe2[0]);
        write(pipe2[1], &c, 1);
        close(pipe2[1]);
        exit(0);
    }else {
        // 向pipe1里写p
        close(pipe1[0]);
        write(pipe1[1], &c, 1);
        close(pipe1[1]);
        // 等子进程退出
        wait(0);
        // 从pipe2里读p
        close(pipe2[1]);
        read(pipe2[0], &c, 1);
        close(pipe2[0]);
        fprintf(1,"%d: received pong\n", getpid());
    }
    // 子进程向父进程发送上述同一个字节 通过另一个管道
    exit(0);
}
```
## Find
查找目录树中所有具有指定名称的文件，向子目录递归查询。

示例是输入了两个参数，第一个是“.”，是根目录；第二个是要查找的文件名，也就是在所有目录中查找。下边是阅读ls实现的代码注释，按照这个思路，find就是ls目录树所有文件，找到与文件名相同的文件路径，所以就是当ls遇到目录时继续ls进行递归实现，再加上文件名判断打印即可。

首先阅读ls代码，使用open()得到文件描述符fd，再使用fstat()将fd的文件信息存入st结构体中。根据st中所示文件的类型，进行对应处理，如是文件直接打印，如是目录还需打印目录项所有条目。
```C
void
ls(char *path)
{
  char buf[512], *p;
  int fd;
  struct dirent de;
  struct stat st;

  if((fd = open(path, O_RDONLY)) < 0){
    fprintf(2, "ls: cannot open %s\n", path);
    return;
  }

  if(fstat(fd, &st) < 0){ // fstat系统调用 把fd文件的信息存入st结构体中
    fprintf(2, "ls: cannot stat %s\n", path);
    close(fd);
    return;
  }

  switch(st.type){
  case T_DEVICE:
  case T_FILE:
    printf("%s %d %d %d\n", fmtname(path), st.type, st.ino, (int) st.size); // 文件直接打印 路径名 文件类型 inode号 文件大小
    break;

  case T_DIR:
    if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
      printf("ls: path too long\n");
      break;
    }
    strcpy(buf, path); // 路径名copy到buf中
    p = buf+strlen(buf); // 指向buf末尾
    *p++ = '/'; // 当前目录末尾+/ 继续指向末尾
    while(read(fd, &de, sizeof(de)) == sizeof(de)){ // 目录也是文件 可以read 目录里包含一系列dirent结构体数据（存有inode及对应的路径名）
      if(de.inum == 0)
        continue;
      memmove(p, de.name, DIRSIZ); // 复制目录项的路径名到p中
      p[DIRSIZ] = 0;
      if(stat(buf, &st) < 0){
        printf("ls: cannot stat %s\n", buf);
        continue;
      }
      printf("%s %d %d %d\n", fmtname(buf), st.type, st.ino, (int) st.size); // 打印该文件信息
    }
    break;
  }
  close(fd);
}
```
新建find.c文件，代码实现：
```C
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"
#include "kernel/fcntl.h"

void
lsa(char *path, char* filename)
{
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if((fd = open(path, O_RDONLY)) < 0){
        fprintf(2, "ls: cannot open %s\n", path);
        return;
    }

    if(fstat(fd, &st) < 0){ // fstat系统调用 把fd文件的信息存入st结构体中
        fprintf(2, "ls: cannot stat %s\n", path);
        close(fd);
        return;
    }

    switch(st.type){
        case T_DEVICE:
        case T_FILE:
            break;

        case T_DIR:
            if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
                printf("ls: path too long\n");
                break;
            }
            strcpy(buf, path); // 路径名copy到buf中
            p = buf+strlen(buf); // 指向buf末尾
            *p++ = '/'; // 当前目录末尾+/ 继续指向末尾
            while(read(fd, &de, sizeof(de)) == sizeof(de)){ // 目录也是文件 可以read 目录里包含一系列dirent结构体数据（存有inode及对应的路径名）
                if(de.inum == 0)
                    continue;
                memmove(p, de.name, DIRSIZ); // 复制目录项的路径名到p中
                p[DIRSIZ] = 0;
                if(stat(buf, &st) < 0){
                    printf("ls: cannot stat %s\n", buf);
                    continue;
                }
                if (st.type == T_DIR) { // 该目录下这个目录项对应的依旧是目录 递归遍历
                    if (strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0)
                        continue;
                    lsa(buf, filename);
                }else { // 这个目录项对应的是文件 进行判断输出
                    if (strcmp(de.name, filename) == 0) {
                        printf("%s\n", buf);
                    }
                }
            }
            break;
    }
    close(fd);
}

void
main(int argc, char *argv[])
{
    if (argc <= 1) {
        fprintf(2,"usage: find <.> <filename>\n");
        return;
    }
    char *path = argv[1];
    char *filename = argv[2];
    lsa(path, filename);
}
```

## Xargs
参数描述为一个要运行的命令，然后从标准输入中读取行，并将该行附加到命令的参数中。

有个坑是exec要求传入的字符串参数数组必须以0结尾，表示参数列表的结束。我说调了好久为什么echo一直没有输出。

初步代码实现：
```C

#include "kernel/param.h"
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

// 读取每一行 出现\n进行分割
int readline(char* buf) {
    char c;
    char* p = buf;
    while (read(0, &c, 1) > 0) {
        if (c == '\n')
            return 1;
        *p++ = c;
    }
    return 0;
}

int
main(int argc, char *argv[])
{
    char buf[512];
    // 对标准输入进行读取
    while (readline(buf)) {
        char* argvexec[MAXARG];
        argvexec[0] = argv[1];
        argvexec[1] = argv[2];
        argvexec[2] = buf;
        argvexec[3] = 0;
        // 对每一行执行exec
        if (fork() == 0) {
            exec(argv[1], argvexec);
            exit(0);
        }else {
            wait(0);
        }
    }
}
```
运行`sh < xargstest.sh`时出现了错误：
```Shell
$ sh < xargstest.sh
$ $ $ $ $ $ hello
hello
grep: cannot open ./b/b
$ $ 
```
查看测试脚本，是在根目录下创建了./a/b、./c/b、./b三个文件，为什么会出现./b/b文件？打印输入参数，发现是buf存的./b/b，说明是在readline()时出的问题，显然是buf没有清空，读到了之前的旧的数据（./c/b后边的/b），故加上memset()进行清零。
```Shell
init: starting sh
$ sh < xargstest.sh
$ $ $ $ $ $ grep
hello
./a/b
(null)
hello
grep
hello
./c/b
(null)
hello
grep
hello
./b/b
(null)
grep: cannot open ./b/b
$ $ 
```
新建xargs.c文件，最终代码实现：
```C

#include "kernel/param.h"
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

// 读取每一行 出现\n进行分割
int readline(char* buf) {
    memset(buf, 0, sizeof(buf));
    char c;
    char* p = buf;
    while (read(0, &c, 1) > 0) {
        if (c == '\n')
            return 1;
        *p++ = c;
    }
    return 0;
}

int
main(int argc, char *argv[])
{
    char buf[512];
    // 对标准输入进行读取
    while (readline(buf)) {
        char* argvexec[MAXARG];
        argvexec[0] = argv[1];
        argvexec[1] = argv[2];
        argvexec[2] = buf;
        argvexec[3] = 0;
        // 对每一行执行exec
        if (fork() == 0) {
            exec(argv[1], argvexec);
            exit(0);
        }else {
            wait(0);
        }
    }
}
```





---

> 作者: [UAreFree](https://github.com/UAreFree)  
> URL: http://localhost:1313/posts/mit-lab-util/  

