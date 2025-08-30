# MIT6.1810 Lab:Locks


MIT6.1810 lab xv6 2024 locks

<!--more--> 

## Buffer cache
如果多个进程密集使用文件系统，它们可能会争用bcache.lock，这是一个大锁，保护bcache中的buffers。修改bcache、bget、brelse使不同块中的并发查找和释放不太可能发生锁冲突。不能增加buffer的数量，也就是依旧30个buffer。不需要实现LRU缓存替换策略，但必须当缓存未命中时，能使用任何refcnt为零的buffer。这个refcnt表示当前有多少进程在使用这个buffer，每次bget()得到一个buffer后，会对这个buffer的refcnt加一，只有当refcnt变为0时，才会被回收。
```Bash
$ bcachetest
start test0
test0 results:
--- lock kmem/bcache stats
lock: kmem: #test-and-set 0 #acquire() 33030
lock: kmem: #test-and-set 0 #acquire() 28
lock: kmem: #test-and-set 0 #acquire() 73
lock: bcache: #test-and-set 0 #acquire() 96
lock: bcache.bucket: #test-and-set 0 #acquire() 6229
lock: bcache.bucket: #test-and-set 0 #acquire() 6204
lock: bcache.bucket: #test-and-set 0 #acquire() 4298
lock: bcache.bucket: #test-and-set 0 #acquire() 4286
lock: bcache.bucket: #test-and-set 0 #acquire() 2302
lock: bcache.bucket: #test-and-set 0 #acquire() 4272
lock: bcache.bucket: #test-and-set 0 #acquire() 2695
lock: bcache.bucket: #test-and-set 0 #acquire() 4709
lock: bcache.bucket: #test-and-set 0 #acquire() 6512
lock: bcache.bucket: #test-and-set 0 #acquire() 6197
lock: bcache.bucket: #test-and-set 0 #acquire() 6196
lock: bcache.bucket: #test-and-set 0 #acquire() 6201
lock: bcache.bucket: #test-and-set 0 #acquire() 6201
--- top 5 contended locks:
lock: virtio_disk: #test-and-set 1483888 #acquire() 1221
lock: proc: #test-and-set 38718 #acquire() 76050
lock: proc: #test-and-set 34460 #acquire() 76039
lock: proc: #test-and-set 31663 #acquire() 75963
lock: wait_lock: #test-and-set 11794 #acquire() 16
tot= 0
test0: OK
```
实验所期望的输出可以看出，lock: bcache的锁争用次数变为了0，且多了13个lock: bcache.bucket，每个bucket的锁争用次数也为0。

需要以“bcache”开头为每个锁调用initlock方法。bcache是多个进程共享的，不能像kalloc一样将所有的buffer切割为每个进程所独享，很明显buffer是固定30个大小，多进程需要都能够访问的。建议使用哈希表来查找缓存中的块号，每个哈希桶持有一把锁。哈希表就是key-value对，那我的key就是hash(块号)，value就是存储的buffer，每个桶还要持有一把锁。

这些锁冲突情况是可以接受的：
1. 两个进程去使用相同的块号。
2. 两个进程查bcache中的buffer（遍历找对应的块号），发现没有，需要查找未使用的块（refcnt=0）进行替换。
3. 两个进程同时使用哈希到一个bucket的块号。

hints：
1. 哈希表大小设为13，固定值，印证了上述结果打印可以通过test。
2. 在哈希表中维护buffer操作必须是原子性的。
3. 删除双向链表形式的bcache且不实现LRU，在bget()中可以选择任何refcnt==0的buffer，在brelse()中也无需获取bcache.lock。
4. 遍历哈希表以及遍历bucket的元素去查找未使用的buffer是可以的。
5. 在驱逐一个buffer时需要持有bcache锁和每个bucket的锁。
6. 正常来说，驱逐一个buffer会将其搬到另一个槽位上（块号hash后得到不同的key），如果在同一个槽位上需要避免死锁。
7. 调试时需要保留全局bcache.lock的acquie/relse在bget()的开头和结尾，一旦代码正确remove这个全局锁。
8. 使用xv6的竞争检测器来找潜在的竞争。

实验思路：

建立大小为13的哈希表，要访问的buffer块哈希到桶上，每个桶一把锁，这样就将原来的一把大锁化为了13把小锁。当缓存命中时就直接拿取对应的锁进行更新操作，很容易处理，就是拿到自己桶对应的小锁即可；但当缓存未命中时，就要遍历buffer去找最久未使用的buffer进行驱逐替换处理。这里分为两种情况，如果要驱逐的buffer和新加入的buffer哈希到一个桶上，以及未哈希到一个桶上。不在同一个桶上就涉及链表断开、重分配的操作，在同一个桶上就不需要链表操作。

实验要处理的上述代码逻辑主要体现在bget函数上，在bget中就是先获取桶A的锁，去遍历所有桶找最久未使用的buffer，然后拿到对应桶B的锁进行链表断开驱逐buffer操作，释放桶B的锁，将buffer挂在桶A上，再释放桶A的锁。这是我最开始的实现方式，运行xv6后会发现系统卡住不动了，分析是发生了死锁。这里由于是遍历所有桶，就有可能出现这种情况，即CPU1拿到桶A的锁，遍历发现要驱逐的buffer在桶B，于是申请桶B的锁，而同时CPU2拿到桶B的锁，同样遍历发现要驱逐的buffer在桶A，去申请桶A的锁。这样就是CPU1拿着桶A的锁去申请桶B的锁，CPU2拿着桶B的锁去申请桶A的锁，造成死锁。死锁的发生条件：互斥、请求保持、不可剥夺、循环等待。破坏上述4个条件之一就可以避免死锁，这里可以破坏请求保持条件，即在遍历查找要驱逐的buffer时释放自己的桶锁，后边重分配操作时再获取。

上述处理方式虽然解决了死锁的问题，但又会带来一个问题，就是在释放桶A的锁之后，拿到桶B的锁之前，这个间隙可能会同时有一个线程也在对同一个要添加的块进行bget，那么就会同样判断缓存未命中，又去重复进行驱逐替换操作，导致同一区块有多份缓存的情况。那么怎么解决这个问题呢？这里采取的办法是使用一把驱逐锁保护这个间隙，让桶A释放自己的锁之后，拿到桶B的锁之前这个过程串行化。并且在获取这个驱逐锁后再次判断是否缓存命中，这样就保证只有第一个进入的线程对该块进行了缓存操作，以下是具体的代码实现。

在buf结构体中新增lastuse成员变量，维护一个时间戳，方便查找最久未使用的buffer。
```C
struct buf {
  int valid;   // has data been read from disk?
  int disk;    // does disk "own" buf?
  uint dev;
  uint blockno;
  struct sleeplock lock;
  uint refcnt;
  uint lastuse;
  //struct buf *prev; // LRU cache list
  struct buf *next;
  uchar data[BSIZE];
};
```
bio.c代码：
```C
#include "types.h"
#include "param.h"
#include "spinlock.h"
#include "sleeplock.h"
#include "riscv.h"
#include "defs.h"
#include "fs.h"
#include "buf.h"

#define NBUCKET 13
#define HASH(dev, blockno) (((dev) << 27 | (blockno)) % NBUCKET)

struct {
  struct buf buf[NBUF];

  struct buf bufmap[NBUCKET]; // 哈希桶
  struct spinlock bucketlocks[NBUCKET]; // 每个桶一把锁
  struct spinlock evictionlocks[NBUCKET]; // 每个桶一把驱逐锁
} bcache;

void
binit(void)
{
  for (int i = 0; i < NBUCKET; i++) {
    initlock(&bcache.bucketlocks[i], "bcache_bucket_lock");
    initlock(&bcache.evictionlocks[i], "bcache_eviction_lock");
    bcache.bufmap[i].next = 0;
  }

  for (int i = 0; i < NBUF; i++) {
    struct buf *b = &bcache.buf[i];
    initsleeplock(&b->lock, "buffer");
    b->lastuse = 0;
    b->refcnt = 0;
    // b接到0号bucket上
    b->next = bcache.bufmap[0].next;
    // 挂上新的b
    bcache.bufmap[0].next = b;
  }
}

// Look through buffer cache for block on device dev.
// If not found, allocate a buffer.
// In either case, return locked buffer.
static struct buf*
bget(uint dev, uint blockno)
{

  struct buf *b;

  uint key = HASH(dev, blockno);

  // 拿到key对应的桶锁
  acquire(&bcache.bucketlocks[key]);

  // 缓存命中
  for (b = bcache.bufmap[key].next; b; b = b->next) {
    if (b->dev == dev && b->blockno == blockno) {
      b->refcnt++;
      // 释放key桶锁
      release(&bcache.bucketlocks[key]);
      acquiresleep(&b->lock);
      return b;
    }
  }
  // 缓存未命中

  // 先释放key桶锁 避免死锁
  release(&bcache.bucketlocks[key]);

  // 拿到桶的驱逐锁串行化 避免对同一个块进行多次缓存驱逐及重分配
  acquire(&bcache.evictionlocks[key]);
  // 再次检查 是避免第二个进程拿到锁后进行缓存驱逐及重分配
  for (b = bcache.bufmap[key].next; b; b = b->next) {
    if (b->dev == dev && b->blockno == blockno) {
      acquire(&bcache.bucketlocks[key]);
      b->refcnt++;
      release(&bcache.bucketlocks[key]);
      release(&bcache.evictionlocks[key]);
      acquiresleep(&b->lock);
      return b;
    }
  }
  // 没有缓存命中执行的操作
  struct buf *beforeleast = 0;
  uint holdingbucket = -1;
  for (int i = 0; i < NBUCKET; i++) {
    // 拿到当前桶锁
    acquire(&bcache.bucketlocks[i]);
    int newfound = 0;
    for (b = &bcache.bufmap[i]; b->next; b = b->next) {
      // 遍历当前桶的buffer
      // 没有文件引用的空闲块 找lastuse最小的
      if (b->next->refcnt == 0 && (!beforeleast || b->next->lastuse < beforeleast->lastuse)) {
        beforeleast = b;
        newfound = 1;
      }
    }
    if (!newfound) {
      // 没找到 释放当前桶锁
      release(&bcache.bucketlocks[i]);
    }else {
      // 新桶找到了就更新 释放旧桶的锁
      if (holdingbucket != -1) release(&bcache.bucketlocks[holdingbucket]);
      holdingbucket = i;
    }
  }
  // 遍历所有桶都没有找到
  if (!beforeleast) {
    panic("bget: no buffers");
  }
  // 找的对应buffer
  b = beforeleast->next;
  // 驱逐的buffer和新加的buffer不是一个桶
  if (holdingbucket != key) {
    // 移除驱逐桶的buffer
    beforeleast->next = b->next;
    release(&bcache.bucketlocks[holdingbucket]);
    // 重新拿到key桶的锁
    acquire(&bcache.bucketlocks[key]);
    // 挂到key桶上
    b->next = bcache.bufmap[key].next;
    bcache.bufmap[key].next = b;
  }
  // 是一个桶就不需要链表操作
  b->dev = dev;
  b->blockno = blockno;
  b->refcnt = 1;
  b->valid = 0;
  release(&bcache.bucketlocks[key]);
  release(&bcache.evictionlocks[key]);
  acquiresleep(&b->lock);
  return b;

}

// Return a locked buf with the contents of the indicated block.
struct buf*
bread(uint dev, uint blockno)
{
  struct buf *b;

  b = bget(dev, blockno);
  if(!b->valid) {
    virtio_disk_rw(b, 0);
    b->valid = 1;
  }
  return b;
}

// Write b's contents to disk.  Must be locked.
void
bwrite(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("bwrite");
  virtio_disk_rw(b, 1);
}

// Release a locked buffer.
// Move to the head of the most-recently-used list.
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  uint key = HASH(b->dev, b->blockno);
  acquire(&bcache.bucketlocks[key]);
  b->refcnt--;
  if (b->refcnt == 0) {
    // 更新时间戳
    b->lastuse = ticks;
  }
  release(&bcache.bucketlocks[key]);
}

void
bpin(struct buf *b) {
  uint key = HASH(b->dev, b->blockno);
  acquire(&bcache.bucketlocks[key]);
  b->refcnt++;
  release(&bcache.bucketlocks[key]);
}

void
bunpin(struct buf *b) {
  uint key = HASH(b->dev, b->blockno);
  acquire(&bcache.bucketlocks[key]);
  b->refcnt--;
  release(&bcache.bucketlocks[key]);
}
```
test结果：
{{< image src="/xv6/bcachetest.png" caption="bcachetest" >}}


---

> 作者: [UAreFree](https://github.com/UAreFree)  
> URL: http://localhost:1313/posts/mit-lab-lock/  

