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
fs.img可以理解为虚拟机中Linux系统所用的磁盘文件，里面有根文件系统，这个文件是怎么来的？这个需要看一下makefile
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
UEXTRA和UPROGS是用户态程序，通过mkfs目录下的mkfs.c打包生成fs.img镜像文件，这里仔细分析下mkfs.c中的main函数

