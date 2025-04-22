---
title: Linux虚拟机内核编译
date: 2025-04-02 09:44:41
tags:
    - Linux
    - qemu
categories: Study
---

# 说明

记录一下在实验室host上的qemu虚拟机(Ubuntu)内编译不止一种linux内核的过程.

## 宿主机编译内核

根据hd师兄所说:

> 可以在host上编译出bzImage文件，然后-kernel参数
> <https://nickdesaulniers.github.io/blog/2018/10/24/booting-a-custom-linux-kernel-in-qemu-and-debugging-it-with-gdb/>
> -initrd后面的文件一般直接从vm里面复制一份大部分时候都能用
> 如果不行，就把虚拟机的核和内存分多一些，然后在虚拟机编译一次安装一次，生成新的initrd，再复制到host

我希望先试试在宿主机编译内核的方式.
在虚拟机编译有各种缺点, 比如占用空间, 我编译时剩余10G空间都不足编译. 而且如果有多个内核想测试, 在宿主机编译也更方便.

### bzImage 和 initrd 概念

需要先搞清楚这两个概念.

1. **bzImage**：
   - **定位**：压缩的 Linux 内核镜像文件（Big ZImage）
   - **组成**：
     - 引导头（Boot Header）：用于 BIOS/UEFI 识别可引导镜像
     - 解压程序：负责自解压内核到内存
     - 核心内核代码：包含调度、内存管理、驱动框架等核心模块
   - **生成路径**：`arch/x86_64/boot/bzImage`（x86架构）
   - 在linux内, /boot文件夹下的vmlinuz其实就是在内核编译的`make install`一步中把bzImage拷贝过去得到的

2. **initrd（Initial RAM Disk）**：
   - **作用**：临时根文件系统（RAM-based rootfs）
   - **生命周期**：
     - 内核启动时加载到内存
     - 提供必要的驱动模块和工具（如磁盘驱动、LVM工具等）
     - 挂载真实根文件系统后自动卸载
   - **典型内容**：
  
     ```bash
     /bin/busybox      # 基础工具集
     /lib/modules      # 内核模块
     /scripts/init     # 初始化脚本
     ```

我们如果在不改变虚拟机内部数据的情况下想替换内核, 需要在外部宿主机准备bzImage和initrd.

#### Q：为何有时需要重新生成initrd？

当出现以下情况时需要更新initrd：

1. 内核模块变更（如新增存储驱动）
2. 根文件系统位置变化（如从/dev/sda1改为/dev/vda1）
3. 加密磁盘等需要早期加载的工具

**生成命令示例**：

```bash
# 在虚拟机内操作
sudo mkinitramfs -o /boot/initrd.img-$(uname -r)
```

### 具体流程

目前我有qemu虚拟机, 名称假设为femu. 我在宿主机上也有要编译的linux内核代码.(我这里出于测试目的用的是CCZNS的, <https://github.com/yingjia-wang/CCZNS>)

下面用G表示目标, D表示要做的事情, N表示补充说明, ND表示对补充说明部分要做的事情. GOK表示目标完成, NOK表示补充说明完成.

G: 生成bzImage

D: 进入linux内核代码文件夹(假设叫linux), 通过`make menuconfig`配置内核. 这里假设不需要进行额外设定. 会当场得到`.config`文件.

D: 运行`make -j$(nproc) bzImage` 编译内核镜像. 可能会生成到`arch/x86_64/boot/bzImage`. 我的大小约11MB.

GOK.

G: 获取initrd

D: 虚拟机内可能已经有initrd, 在`/boot`文件夹.

D: 假设虚拟机和宿主机有共享文件夹`/host-shared`, 可使用`sudo cp /boot/initrd.img-$(uname -r) /mnt/hostshare/initrd.img`复制过去. 我的大小约100MB.

如果没有或者不确定是否可用, 也可以直接在虚拟机内重新生成:

```bash
sudo mkinitramfs -o /mnt/hostshare/initrd.img
```

不过我这样在旧内核生成的initrd似乎不能用于新内核. 所以我还是选择了一种麻烦但更简单的方式, 就是给qemu扩容后在内部编译一次安装一次, 再用`mkinitramfs`生成新的initrd.

N: qemu扩容大致流程(假设`lsblk`看到的要使用的硬盘是`/dev/sda3`, `df -h`看到名称是`/dev/mapper/ubuntu--vg-ubuntu--lv`(我也不懂这个名称是什么意思)):

```bash
sudo apt install cloud-guest-utils  # 如果未安装growpart
sudo growpart /dev/sda 3           # 调整分区到最大可用空间
sudo pvresize /dev/sda3            # 让LVM识别物理卷的新大小
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv  # 使用全部剩余空间
# 对于ext2/3/4文件系统：
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
df -h /  # 确认根分区容量已增加
```

NOK.

内部编译获取initrd和bzImage:

```bash
# 在虚拟机的新内核源码目录执行
make clean && make mrproper # 如果已有预设定的.config则不需要make mrproper, 直接跳到下面make menconfig的下一句
make menuconfig      # 按需调整. 不清楚就不调整

make -j$(nproc) > build.log 2>&1   # 编译内核（根据CPU核心数并行编译）并把输出记录到log文件(当然也可以不输出到log文件)

ls -lh arch/x86/boot/bzImage       # 检查 bzImage 是否存在（x86 示例）, 通常应在 5~15 MB 范围内
file vmlinux                     # 检查 vmlinux 是否存在, 应为非压缩的 ELF 可执行文件
sudo make modules_install       # 安装内核模块到系统目录
sudo make install               # 安装内核到/boot并自动生成新内核的vmlinuz(即bzImage)和initrd, 旧的会保留
```

编译如果很快多半遇到问题了, 仔细看看输出然后自行搜索或问ai.

关于certs.pem的一个报错的解决方法:<https://blog.csdn.net/qq_36393978/article/details/118157426>
关于missing symbol table的解决方法:<https://blog.csdn.net/qq_40552624/article/details/131401326>
关于`pahole (pahole) is not available`: pahole需要安装, 并且这里需要安装较低版本(1.23以下)否则与内核会不匹配, 大致流程:

```bash
git clone https://git.kernel.org/pub/scm/devel/pahole/pahole.git
cd pahole
git checkout v1.23  # 切换到 1.23 版本标签

mkdir build
cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr/local -D__LIB=lib ..
make -j$(nproc)
sudo make install
ldconfig

pahole --version
```

每次编译失败之后记得`make clean`一下. 如果感觉config有问题就再`make mrproper`但之后需要重新生成和设定`.config`.

后续, initrd和bzImage传到宿主机后, 可以回linux文件夹做make clean, 就可以腾出很多空间.
缩减qemu虚拟机可参考<https://www.cnblogs.com/milton/p/15940809.html>. 最好是提前做好备份, 我是把虚拟机搞坏了.

编译新内核后, 如果linux版本比旧内核高, 可能会导致默认启动内核变为新内核.
如果想把默认启动内核改为旧内核, 可以参考<https://blog.csdn.net/bby1987/article/details/104264285>

对于完全没有编译过新内核的qemu虚拟机, 我测试发现也是可以通过使用之前编译出的bzImage和initrd来启动新内核的.
不过**启动虚拟机时设定的内存大小(-m xG)似乎需要和编译initrd时的设定相同**. 我编译时是设定为12G, 后面如果用4G内存启动就会提示内存不足.

GOK.

N: 关于共享文件夹, 简单说一下怎么设置:

ND: 在qemu启动参数添加:

```bash
-fsdev local,security_model=passthrough,id=fsdev0,path=/home/yourname/host_share \
-device virtio-9p-pci,id=fs0,fsdev=fsdev0,mount_tag=hostshare
```

在虚拟机内执行以下步骤:

```bash
# 安装 9P 文件系统支持
sudo apt update
sudo apt install linux-modules-extra-$(uname -r)  # 确保内核模块已加载
sudo modprobe 9pnet_virtio  # 加载内核模块
# 手动挂载共享文件夹
sudo mkdir -p /mnt/hostshare
sudo mount -t 9p -o trans=virtio,version=9p2000.L hostshare /mnt/hostshare
# 验证挂载
echo "Hello from VM" | sudo tee /mnt/hostshare/test.txt # 创建测试文件
cat /home/yourname/host_share/test.txt # 在宿主机检查文件
```

这里的手动挂载意味着每次开启虚拟机都得做. 有办法自动挂载, 编辑`/etc/fstab`添加:
`hostshare /mnt/hostshare 9p trans=virtio,version=9p2000.L,rw,_netdev 0 0`

我按这个流程是一切顺利了.

NOK.

G: 运行带自定义内核的虚拟机

D: 将生成的bzImage和initrd.img共同存放到某个文件夹. 比如就放到存放qemu虚拟机的文件夹下.

在脚本中, 添加常量:

```bash
KERNEL_OPTIONS=" \
   -kernel $KERDIR/bzImage \
   -initrd $KERDIR/initrd.img
"
```

在qemu启动参数添加 `${KERNEL_OPTIONS}\`以及`-append "root=$ROOT console=ttyS0 nokaslr" \`. 后续为了测试方便可自行修改脚本.

这里的`$ROOT`自行修改, 可以在虚拟机内运行`df -h`和`sudo blkid`确认一下root设备的UUID比如`UUID=132343af-024a-4184-a257-a3e9c8f493be`并输入. 输入设备名称应该也行.

N: 这里append参数我本来是加在OPTION里面, 但是其双引号一直导致语义错误, 索性最后就直接加在启动命令里了.

(TODO)




### 一、核心概念解析

#### （一）`bzImage` 和 `initrd` 的本质


#### （二）宿主机编译 vs 虚拟机内编译
| 特性                  | 宿主机编译                     | 虚拟机内编译                 |
|-----------------------|------------------------------|----------------------------|
| **资源占用**           | 占用宿主机资源                | 占用虚拟机资源（可能受限）   |
| **编译速度**           | 多核并行，速度快              | 虚拟机资源限制，速度较慢     |
| **内核安装方式**       | 不污染虚拟机存储（仅传bzImage）| 需安装到虚拟机/boot目录     |
| **调试灵活性**         | 可直接用QEMU+gdb调试          | 依赖虚拟机内部调试工具       |
| **适用场景**           | 快速测试多版本内核            | 需要完整安装到系统的场景     |

---

### 二、QEMU启动流程的深层原理

#### （一）默认启动流程（未指定-kernel时）
1. **BIOS/UEFI阶段**：
   - QEMU模拟固件加载虚拟磁盘的第一个扇区（MBR/GPT）
   - 执行引导加载程序（如GRUB）

2. **引导加载阶段**：
   - GRUB从虚拟磁盘的 `/boot` 目录读取配置文件
   - 加载 `/boot/vmlinuz-xxx` 内核文件和 `/boot/initrd.img-xxx`

3. **内核启动阶段**：
   - 内核解压并初始化硬件
   - 挂载initrd，执行 `/init` 脚本
   - 切换到真实根文件系统（如 `/dev/vda1`）

#### （二）显式指定内核时的流程
```bash
qemu-system-x86_64 -kernel bzImage -initrd initrd.img ...
```
1. **跳过固件阶段**：
   - 直接加载指定的 `bzImage` 到内存
   - 绕过虚拟磁盘的引导加载程序

2. **内核参数控制**：
   - `-append` 参数直接传递内核命令行参数
   - 可覆盖虚拟磁盘中的GRUB配置

---

### 三、原QEMU脚本解析（为何不指定kernel也能启动）

#### （一）关键配置项分析
```bash
-drive file=$OSIMGF,if=none,aio=native,cache=none,format=qcow2,id=hd0
```
- **作用**：加载已安装OS的qcow2镜像
- **隐含行为**：
  - QEMU自动从镜像的引导扇区启动
  - 依赖镜像内预置的 `/boot/vmlinuz` 和 `/boot/initrd.img`

#### （二）验证方法
1. **进入虚拟机查看启动配置**：
   ```bash
   # 查看当前内核版本
   uname -r
   # 查看GRUB配置
   cat /boot/grub/grub.cfg
   # 查看已安装内核文件
   ls /boot/vmlinuz* /boot/initrd*
   ```

2. **QEMU日志分析**：
   ```bash
   dmesg | grep "Kernel command line"
   # 观察实际使用的内核参数
   ```

---

### 四、混合启动方案设计

#### （一）保留原脚本优势
```bash
# 原FEMU设备配置保留
FEMU_OPTIONS="-device femu,devsz_mb=8192,...,femu_mode=3"
```

#### （二）新增内核测试能力
```bash
# 在原有脚本中添加参数
KERNEL_OPTIONS="-kernel ./bzImage -initrd ./initrd.img"
APPEND_OPTIONS="-append 'root=/dev/vda1 console=ttyS0 nokaslr'"

sudo ./qemu-system-x86_64 \
    ...原有参数... \
    ${KERNEL_OPTIONS} \
    ${APPEND_OPTIONS}
```

#### （三）动态切换逻辑
```bash
#!/bin/bash
# femu-start.sh

if [[ $1 == "custom-kernel" ]]; then
    EXTRA_OPTIONS="-kernel ./bzImage -initrd ./initrd.img"
fi

./qemu-system-x86_64 ... ${EXTRA_OPTIONS} ...
```

---

### 五、原理示意图
```
+-------------------------------+
|          QEMU 实例            |
|  +-------------------------+  |
|  |        Guest OS         |  |
|  |  +-------------------+  |  |
|  |  | 内核空间          |  |  |
|  |  | - bzImage         |  |  |
|  |  | - initrd          |  |  |
|  |  +-------------------+  |  |
|  +-------------------------+  |
|                               |
| 启动方式选择:                  |
| 1. 默认启动: 从虚拟磁盘引导     |
|    (依赖/boot内容)             |
| 2. 指定内核: 直载内存镜像       |
|    (覆盖默认配置)              |
+-------------------------------+
```

---

### 六、关键问题解答



#### Q：nokaslr参数的作用？
- **KASLR（Kernel Address Space Layout Randomization）**：
  - 安全特性：随机化内核内存地址
  - 调试影响：导致gdb断点失效
- **nokaslr**：禁用此特性以保证调试时地址稳定

---

### 七、扩展学习建议

1. **内核调试技巧**：
   ```bash
   # 在QEMU启动参数中添加
   -s -S  # 启动gdbserver并暂停CPU
   # 在gdb中连接
   target remote :1234
   hbreak start_kernel
   ```

2. **性能优化方向**：
   - 使用ccache加速重复编译：`export CCACHE_DIR=/path/to/ccache`
   - 启用KVM虚拟化：`-enable-kvm -cpu host`
   - 分配更多CPU核心：`-smp 8`

3. **高级存储配置**：
   ```bash
   # 创建持久化命名空间
   -device nvme,id=nvme0,serial=deadbeef
   -drive file=zns.img,if=none,id=zns1
   -device femu,drive=zns1,nsid=1
   ```

通过这样的原理梳理，你可以更自如地在不同场景间切换测试策略，同时保持系统的高效运行。