# make qemu这个命令做了什么
查看makefile文件
```
QEMU = qemu-system-riscv64

QEMUOPTS = -machine virt -bios none -kernel $K/kernel -m 128M -smp $(CPUS) -nographic
QEMUOPTS += -global virtio-mmio.force-legacy=false
QEMUOPTS += -drive file=fs.img,if=none,format=raw,id=x0
QEMUOPTS += -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

qemu: $K/kernel fs.img
	$(QEMU) $(QEMUOPTS)
```
make qemu做的事其实就是把编译的成果物kernel和fs.img文件根据启动参数QEMUOPTS用qemu模拟器跑起来，这里解释一下QEMUOPTS中各个参数的意义加深对项目的理解
```
-machine virt： 选择 QEMU 的 “virt” machine 型号（一个通用的虚拟化平台，常用于 ARM / RISC-V 等的裸机/OS 测试）
-bios none： 不加载 BIOS/固件。对某些架构（包括 riscv）是常见选项，配合 -kernel 直接把内核映射进来
-kernel $K/kernel： 让 QEMU 直接加载并执行路径为 $K/kernel 的内核镜像
-m 128M：给虚拟机分配 128MB 内存
-smp $(CPUS)： 设置虚拟 CPU 个数为 $(CPU)
-nographic: 不创建图形窗口
-global virtio-mmio.force-legacy=false: global是 QEMU 的“全局对象属性”选项，设置一个设备/总线的属性。virtio-mmio.force-legacy=false 表示禁用（或不强制使用）virtio-mmio 的 legacy 模式，从而尝试使用现代的、非 legacy 的 virtio-mmio 行为（与QEMU 版本、guest 驱动实现有关）。作用是让 virtio-mmio 设备在支持现代规范的 guest 驱动下工作。(这个暂时没理解是什么意思)
-drive file=fs.img,if=none,format=raw,id=x0: 声明一个未直接连接到总线的块设备源，文件是 fs.img，格式是原始raw，并赋予该源 id 为 x0。if=none 表明这个 drive 描述本身不直接指定接口类型（稍后用 -device ... drive=x0 挂接）
-device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0: 创建一个 virtio-blk-device（virtio 块设备，虚拟磁盘），将其与上面 id 为x0的 drive 绑定，并放到virtio-mmio-bus.0总线上。这样 guest 内核会看到一个 virtio 块设备(通常在 Linux 中会出现/dev/vda)
```
# fs.img
fs.img可以理解为虚拟机中Linux系统所用的磁盘文件，里面有根文件系统，这个文件由mkfs函数生成。
```
UEXTRA=
ifeq ($(LAB),util)
	UEXTRA += user/xargstest.sh
endif

UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_forktest\
	...

fs.img: mkfs/mkfs README $(UEXTRA) $(UPROGS)
	mkfs/mkfs fs.img README $(UEXTRA) $(UPROGS)
```
UEXTRA和UPROGS是用户态程序，仔细分析下mkfs.c中的main函数，可以知道这个函数的作用就是将README,\$(UEXTRA)和\$(UPROGS)打包成一个基于inode的文件镜像。
阅读mkfs.c源码，结合陈海波教授写的《操作系统-原理与实践》，可以加深对基于inode的文件系统的理解。
## 基于inode的文件系统
inode是"index node"的简写，即"索引节点"，用于记录一个文件的数据所在的存储块号。
### 文件系统存储布局
![[fs_img_layout.png]]
### super block
```
// 超级块结构体
struct superblock {
  uint magic;        // Must be FSMAGIC
  uint size;         // Size of file system image (blocks)
  uint nblocks;      // Number of data blocks
  uint ninodes;      // Number of inodes.
  uint nlog;         // Number of log blocks
  uint logstart;     // Block number of first log block
  uint inodestart;   // Block number of first inode block
  uint bmapstart;    // Block number of first free map block
};
```
根据这个结构体可知，在基于inode的文件系统中，这个超级块用于存放对这个文件系统的布局的描述信息。比如inode数量，inode表的起始块号和bitmap的起始块号。
在main函数中,这部分代码做的就是完成对super block中各个结构体成员属性值的计算，然后通过`wscet()`函数将超级块的内容写入文件中对应的位置。
```
  nmeta = 2 + nlog + ninodeblocks + nbitmap;
  nblocks = FSSIZE - nmeta;

  sb.magic = FSMAGIC;
  sb.size = xint(FSSIZE);
  sb.nblocks = xint(nblocks);
  sb.ninodes = xint(NINODES);
  sb.nlog = xint(nlog);
  sb.logstart = xint(2);
  sb.inodestart = xint(2+nlog);
  sb.bmapstart = xint(2+nlog+ninodeblocks);

  printf("nmeta %d (boot, super, log blocks %u inode blocks %u, bitmap blocks %u) blocks %d total %d\n",
         nmeta, nlog, ninodeblocks, nbitmap, nblocks, FSSIZE);

  freeblock = nmeta;     // the first free block that we can allocate

  for(i = 0; i < FSSIZE; i++)
    wsect(i, zeroes);

  memset(buf, 0, sizeof(buf));
  memmove(buf, &sb, sizeof(sb));
  wsect(1, buf);    // put super block into 1th block in fs.img
```
### 文件与目录
```
#define NDIRECT 12

// inode结构体
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses
};
```
在基于inode的文件系统中，type字段同于表示当前inode指向的data block块中存放的是目录还是文件。**正如书上所说，将目录看作是一种特殊类型的文件，复用inode机制来实现目录，唯一的区别是目录的格式是由文件系统规定的，数据的读写由文件系统控制，而不是像普通程序那样由应用程序控制。**
以创建根目录为例：
```
  // 目录项
  struct dirent {
    ushort inum;
    char name[DIRSIZ];
  };

  // 创建根目录"/"
  rootino = ialloc(T_DIR);
  assert(rootino == ROOTINO);
  
  // 在根目录中添加目录项"."
  bzero(&de, sizeof(de));
  de.inum = xshort(rootino);
  strcpy(de.name, ".");
  iappend(rootino, &de, sizeof(de));
  
  // 在根目录中添加目录项".."
  bzero(&de, sizeof(de));
  de.inum = xshort(rootino);
  strcpy(de.name, "..");
  iappend(rootino, &de, sizeof(de));
```
上述函数完成后，在qemu中输入`ls -al`,就能看到如下输出了：
```
$ ls  
.              1 1 1024  
..             1 1 1024
```
以向根目录中添加文件为例：
```
// 向文件系统申请一个类型为T_FILE的inode
inum = ialloc(T_FILE);

// 将该目录目录项添加到根文件的inode所指向的data block中
bzero(&de, sizeof(de));
de.inum = xshort(inum);
strncpy(de.name, shortname, DIRSIZ);
iappend(rootino, &de, sizeof(de));

// 将用户态的程序，比如ls这些二进制可执行文件写入inum这个inode所指向的data block
while((cc = read(fd, buf, sizeof(buf))) > 0)
  iappend(inum, buf, cc);
```
`向根目录中添加文件`其实是一个对于操作系统使用者视角的描述。在底层文件系统的视角看的话，所添加文件的数据并不是放在根目录所指向的data block的，它有自己的独立的一块data block用于存放文件的数据，该data block和目录的data block两者之间是孤立的。只是有一个对于这个新增加的data block的描述符(又称为目录项)，做为两者之间联系的纽带，被append到了目录的data block中。
执行完上述函数之后，在qemu中输入`ls -al`,就能看到如下输出了：
```
$ ls  
.              1 1 1024  
..             1 1 1024  
README         2 2 2403  
xargstest.sh   2 3 93  
cat            2 4 35440  
echo           2 5 34296  
.....
```
看到这里，对这个Makefile的构建规则到底做了什么
```
fs.img: mkfs/mkfs README $(UEXTRA) $(UPROGS)
	mkfs/mkfs fs.img README $(UEXTRA) $(UPROGS)
```
和这句hint为什么要这么说
```
- Add your sleep program to UPROGS in Makefile; once you've done that, make qemu will compile your program and you'll be able to run it from the xv6 shell.
```
应该有很深刻的理解了。
### 多级inode
读完书上的基本概念之后，对于一个支持二级索引的inode是这么一个基本认知：
```
struct dinode {
  short type;           // File type
  ...
  uint dir_idx[DIR_IDX_NUM];   // 直接索引，直接指向文件数据
  uint pri_idx[PRI_IDX_NUM];   // 一级索引，指向一个存放一级索引块
  uint sec_idx[SEC_IDX_NUM];   // 二级索引，指向一个存放二级索引块
};
```
而实际上，xv6的inode只支持一级索引，且该索引机制不是通过在inode中添加对应的结构体成员来实现的，而是将直接索引和一级索引都存放在`addrs`数组中，根据代码可知，该inode最大支持管理`256 + 12 = 268KB`的文件。对于文件中指定位置的数据是位于直接索引中还是位于一级索引所指向的数据块中的判定，可以查看`iappend()`函数中的做法。
```
#define NDIRECT 12
#define NINDIRECT (BSIZE / sizeof(uint))
#define MAXFILE (NDIRECT + NINDIRECT)

struct dinode {
  short type;              // File type
  ...
  uint addrs[NDIRECT+1];   // Data block addresses
};
```
