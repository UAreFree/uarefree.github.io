# MIT6.1810 Lab:File System


MIT6.1810 lab xv6 2024 fs

<!--more--> 

## Large files
增加xv6文件大小，一个文件的大小是由inode决定的，inode记录文件的元信息，其中的address数组记录磁盘的块号。我们知道，文件就是一堆磁盘块，磁盘块的数量越多文件越大，所以理论上address数组越大，文件越大。但inode也是占用一个块大小（xv6是1024字节）的，那么address的大小会被限制在二百多，很明显是不够的。这里inode限制address数组大小为13，前12个元素是直接块，即address索引（逻辑块号）与其值（磁盘块号）是一一对应的，第13个元素是间接块，即指向了一个磁盘块，那个磁盘块里又划分了（BSIZE / sizeof(uint)）大小的数组，每个元素指向一个磁盘块，这样就由1个磁盘块扩充为了256个磁盘块。
{{< image src="/xv6/间接块扩充文件.png" caption="间接块扩充文件" >}}
按照类似的思路，我们可以将上述间接块指向的磁盘块改为间接块，这样建立二级间接块，在二级间接块上指向实际的磁盘块，从而将1个磁盘块扩充为了256*256个磁盘块（这里其实牺牲了部分磁盘块作为间接块而不是文件块）。具体实现代码：
```C
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  if(bn < NDIRECT){ // 小于直接块数目
    if((addr = ip->addrs[bn]) == 0){
      addr = balloc(ip->dev); // 分配一个磁盘块给文件
      if(addr == 0)
        return 0;
      ip->addrs[bn] = addr;
    }
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){ // 小于间接块数目
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0){
      addr = balloc(ip->dev); // 分配一个磁盘块作为间接块
      if(addr == 0)
        return 0;
      ip->addrs[NDIRECT] = addr;
    }
    bp = bread(ip->dev, addr); // 从间接块里读
    a = (uint*)bp->data; // 这里转为了uint* 一个指针指向其分配的磁盘块
    if((addr = a[bn]) == 0){ // bp的size就是NINDIRECT BSIZE/sizeof(uint)
      addr = balloc(ip->dev); // 分配一个磁盘块给文件
      if(addr){
        a[bn] = addr;
        log_write(bp);
      }
    }
    brelse(bp);
    return addr;
  }
  bn -= NINDIRECT;

  // 0-256*256
  // 0-255
  if (bn < NDBINDIRECT) { // 小于二级间接块数目
    if((addr = ip->addrs[NDIRECT + 1]) == 0){
      addr = balloc(ip->dev); // 分配一个磁盘块作为一级间接块
      if(addr == 0)
        return 0;
      ip->addrs[NDIRECT + 1] = addr;
    }

    uint cn = bn / NINDIRECT; // 在一级间接块的index
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[cn]) == 0) {
      addr = balloc(ip->dev); // 再分配一个磁盘块作为二级间接块
      if(addr){
        a[cn] = addr;
        log_write(bp);
      }
    }
    brelse(bp);

    uint index = bn % NINDIRECT; // 在二级间接块的index
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[index]) == 0) {
      addr = balloc(ip->dev); // 再分配一个磁盘块给文件
      if(addr) {
        a[index] = addr;
        log_write(bp);
      }
    }
    brelse(bp);
    return addr;
  }

  panic("bmap: out of range");
}
```
注意更改fs.h中的对应宏定义。
```C
#define NDIRECT 11
#define NINDIRECT (BSIZE / sizeof(uint))
#define NDBINDIRECT (NINDIRECT * NINDIRECT)
#define MAXFILE (NDIRECT + NINDIRECT + NDBINDIRECT)

struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+2];   // Data block addresses
};
```
以及file.h中inode的address数组的大小，这里要和dinode的address数组的大小相对应，不然fs.img构建会出错。
```C
// in-memory copy of an inode
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+2];
};
```
还有根据提示要确保itrunc释放文件的所有块，类似的进行更改，至此能通过usertests。
```C
void
itrunc(struct inode *ip)
{
  int i, j;
  struct buf *bp;
  uint *a;

  for(i = 0; i < NDIRECT; i++){ // 释放所有的直接块
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  if(ip->addrs[NDIRECT]){ // 如果一级间接块存在
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){ // 释放所有的文件块
      if(a[j])
        bfree(ip->dev, a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]); // 释放一级间接块
    ip->addrs[NDIRECT] = 0;
  }

  if (ip->addrs[NDIRECT + 1]){ // 如果二级间接块存在
    bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++) {
      if(a[j]) {
        struct buf *bp2 = bread(ip->dev, a[j]);
        uint* c = (uint*)bp2->data;
        for (int k = 0; k < NINDIRECT; k++) {
          if(c[k])
            bfree(ip->dev, c[k]);
        }
        brelse(bp2);
        bfree(ip->dev, a[j]);
      }
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT + 1]); // 释放一级间接块
    ip->addrs[NDIRECT + 1] = 0;
  }

  ip->size = 0;
  iupdate(ip);
}
```
## Symbolic links
这个实验如果明白了符号链接的原理过程就能迎刃而解，本质上就是在path的路径创建了一个新文件（代码以inode表达），在文件里写入target路径，同时标记此文件为T_SYMLINK类型，遇到这种文件要读文件内容再一次以文件内容为path去找对应的inode，从而得到所指向文件的内容。代码具体实现：

增加symlink系统调用，在path处创建一个T_SYMLINK类型的文件，writei写入target路径为文件内容。

```C
uint64
sys_symlink(void) {
  char target[MAXPATH];
  char path[MAXPATH];
  struct inode *ip;

  if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
    return -1;

  //printf("link path: %s\n", path);
  begin_op();
  // 在path上创建一个文件
  ip = create(path, T_SYMLINK, 0, 0);
  if(ip == 0){
    end_op();
    return -1;
  }

  // 文件内容为target
  if(writei(ip, 0, (uint64)target, 0, strlen(target)) < 0) {
    end_op();
    return -1;
  }
  iunlockput(ip);
  end_op();

  return 0; // 成功
}
```
修改open系统调用，循环读取path，如果是符号链接文件，文件内容target读到path里，下个循环再去读指向target的内容。这里调试了个小bug，开始没有在readi之前memset(path, 0, MAXPATH);导致symlinktest过不了，报错：FAILURE: open() failed in many test。然后打印path发现：
```Shell
open path: /testsymlink/aa
while path: /testsymlink/aa
while path: /testsymlink/4a
FAILURE: open() failed in many test
```
分析在建立符号链接时是没有/testsymlink/4a这个path的，看symlinktest.c也知道是在测试从/testsymlink/aa到/testsymlink/az，这些符号链接指向的/testsymlink/4。很容易看出来是path在readi时只覆盖了/testsymlink/aa的/testsymlink/a这个区域为/testsymlink/4，而/testsymlink/aa最后一个a还残留在path里，所以增加memset进行了数据的清理，解决了这个问题。
```C
uint64
sys_open(void)
{
  char path[MAXPATH];
  int fd, omode;
  struct file *f;
  struct inode *ip;
  int n;

  argint(1, &omode);
  if((n = argstr(0, path, MAXPATH)) < 0)
    return -1;

  //printf("open path: %s\n", path);

  begin_op();

  if(omode & O_CREATE){ // 创建文件
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    int depth = 0;
    while (1) {
      //printf("while path: %s\n", path);
      if((ip = namei(path)) == 0){
        end_op();
        return -1;
      }

      ilock(ip);

      if (ip->type == T_SYMLINK && (omode & O_NOFOLLOW) == 0) {
        if (++depth > 10) { // 限制循环链接深度
          iunlockput(ip);
          end_op();
          return -1;
        }

        memset(path, 0, MAXPATH); // 没有此句会出错 path可能会有之前的残留数据未被readi覆盖
        if (readi(ip, 0, (uint64)path, 0, MAXPATH) < 0) { // 读文件内容 target路径
          iunlockput(ip);
          end_op();
          return -1;
        }
        iunlockput(ip); // 要释放ip
      }else
        break;
    }

    if(ip->type == T_DIR && omode != O_RDONLY){
      iunlockput(ip);
      end_op();
      return -1;
    }
  }

  if(ip->type == T_DEVICE && (ip->major < 0 || ip->major >= NDEV)){
    iunlockput(ip);
    end_op();
    return -1;
  }

  if((f = filealloc()) == 0 || (fd = fdalloc(f)) < 0){
    if(f)
      fileclose(f);
    iunlockput(ip);
    end_op();
    return -1;
  }

  if(ip->type == T_DEVICE){
    f->type = FD_DEVICE;
    f->major = ip->major;
  } else {
    f->type = FD_INODE;
    f->off = 0;
  }
  f->ip = ip;
  f->readable = !(omode & O_WRONLY);
  f->writable = (omode & O_WRONLY) || (omode & O_RDWR);

  if((omode & O_TRUNC) && ip->type == T_FILE){
    itrunc(ip);
  }

  iunlock(ip);
  end_op();

  return fd;
}
```



---

> 作者: [UAreFree](https://github.com/UAreFree)  
> URL: http://localhost:1313/posts/mit-lab-fs/  

