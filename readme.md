# 编译实时内核
为了保证`ethercat主站`实时性，我们需要重新编译实时内核。按照igh的官网说明，如果没有使用官方实现的驱动的网卡，可以不使用实时内核。在这里我们使用`realtek 8168b`网卡（8168b网卡在开源驱动型号变为了8169）。然后使用`xenomai`作为实时内核的实现。
## linux
我们使用linux作为`ethercat`主站的实现。在这里我们使用32位的`ubuntu 14.04`（内核版本为 3.13.* ）。然后编译的内核版本为`3.14.44`。[下载地址](https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.14.44.tar.xz)
## xenomai
`xenomai`作为一个linux的实时固件，它的原理是重写了linux系统的中断处理函数，从而自己变为了一个简单的实时系统内核，而linux本身变为了它的一个优先级较低的task，从而保证其他实时task的处理，在`ethercat主站`中我们使用`xenomai2`，目前它最新版本为2.6.5。[下载地址](http://xenomai.org/downloads/xenomai/stable/xenomai-2.6.5.tar.bz2)

## 安装编译内核时的依赖
```
sudo apt-get install kernel-package libqt4-dev pkg-config vim
```
## patch linux 内核文件
将 linux 源码 和 xenomai 源码解压到同一文件夹下。然后执行
```
cd linux-3.14.44
../xenomai-2.6.5/scripts/prepare-kernel.sh
```
几个选项都选默认

## 配置内核
```
make xconfig
```
推荐配置如下
```
* General setup
  --> Local version - append to kernel release: -xenomai-2.6.5
  --> Timers subsystem
      --> High Resolution Timer Support (Verify you have HRT ON)
* Real-time sub-system
  --> Xenomai (Enable)
  --> Nucleus (Enable)
* Power management and ACPI options
  --> Run-time PM core functionality (Disable)
  --> ACPI (Advanced Configuration and Power Interface) Support
      --> Processor (Disable)
  --> CPU Frequency scaling
      --> CPU Frequency scaling (Disable)
  --> CPU idle
      --> CPU idle PM support (Disable)
* Pocessor type and features
  --> Processor family
      --> Core 2/newer Xeon (if \"cat /proc/cpuinfo | grep family\" returns 6, set as Generic otherwise)
      --> SMT (Hyperthreading) scheduler support (Disable)
      --> Preemption Model
          --> Voluntary Kernel Preemption (Desktop)
 * Device Driver
  -->GPIO Support
      --> Intel EG20T PCH/LAPIS Semiconductor***(Disable)
  -->USB Support
      --> USB Gadget Support (Disable)
  -->stagging drivers
      --> Data Aquisation Support(comedi) (Disable)
```

## 编译内核
```
CONCURRENCY_LEVEL=$(nproc) make-kpkg --rootcmd fakeroot --initrd kernel_image kernel_headers
```
编译内核时间取决你的cpu。

## 安装内核
编译完成后，需要将编译好的内核的2个deb文件安装。
```
cd ..
sudo dpkg -i linux-*.deb
```

## 配置GRUB
```
sudo vim /etc/default/grub
```
然后修改如下：
```
GRUB_DEFAULT=saved
GRUB_SAVEDEFAULT=true
GRUB_HIDDEN_TIMEOUT_QUIET=true
GRUB_TIMEOUT=5
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash i915.i915_enable_rc6=0 i915.powersave=0 noapic xeno_nucleus.xenomai_gid=1234 xenomai.allowed_group=1234"
GRUB_CMDLINE_LINUX=""
```

## 更新grub重启
```
sudo update-grub
sudo reboot
```
## 安装xenomai
```
cd xenomai-2.6.5/
./configure
make -j$(nproc)
sudo make install
```

# 编译igh
## 安装依赖
在安装`build-essential`后，安装下面编译该程序的其他依赖。

`sudo apt-get install autoconf autogen libtool`


## 配置
然后进入`etherlabmaster-code`文件夹，执行`./bootstrap`，生成配置文件。

然后执行下面命令进行配置：
```
./configure --with-linux-dir=/usr/src/linux-headers-3.14.44-xenomai-2.6.5 --with-module-dir=/lib/modules/3.14.44-xenomai-2.6.5 --enable-generic --enable-rtdm --with-xenomai-dir=/usr/xenomai --enable-cycles --enable-hrtimer --enable-8139too=no --enable-r8169=yes  --with-r8169-kernel=3.14 --prefix=/usr/local/etherlab
```
~~其中`/usr/src/linux-headers-4.4.0-28-generic`为你想把ethercat驱动编译在那个内核版本，你当前系统运行的内核版本可以通过`uname -a`命令检测到。~~

## 编译安装
```
make #编译用户态的库
make modules #编译ethercat驱动
sudo su #登录root用户
make install #安装库文件
make modules_install #安装驱动
```

## 配置项
```
cd /etc
sudo mkdir sysconfig
sudo cp /usr/local/etherlab/etc/sysconfig/ethercat /etc/sysconfig
sudo cp /usr/local/etherlab/etc/init.d/ethercat /etc/init.d
sudo cp /usr/local/etherlab/etc/ethercat.conf /etc
```
然后使用`ifconfig`命令获取到网卡的mac地址，然后配置2个文件。
```
/etc/ethercat.conf
/etc/sysconfig/ethercat
```
你需要在上面2个文件中的`MASTER0_DEVICE=`后添加你的网卡的mac地址形如`MASTER0_DEVICE="b8:ae:ed:7e:6e:66"`, 然后把`DEVICE_MODULES=`添加`"generic"`或`r8169`形如`DEVICE_MODULES="generic"`

然后执行`sudo depmod`命令。

## 运行
通过`sudo /etc/init.d/ethercat start`启动ethercat

通过`sudo /etc/init.d/ethercat stop`停止ethercat

通过`sudo /etc/init.d/ethercat restart`重新启动ethercat


## 配置用户态库和依赖
### 修改ethercat设备权限
```
cd /etc/udev/rules.d #进入udev rules文件夹
sudo vim 99-ethercat.rules #新建一个ethercat的rule文件
```
在`99-ethercat.rules`文件中添加下面内容
```
KERNEL=="EtherCAT[0-9]", MODE="0777"
```
保存后退出，然后执行`sudo udevadm control --reload-rules` 然后重启电脑。

### 配置库
将`/usr/local/etherlab/include`下的2个头文件放入`/usr/local/include`

将`/usr/local/etherlab/lib`下的`libethercat.so.1.0.0`放入`/usr/local/lib`

将`/usr/local/etherlab/bin`下的`ethercat`改名为`ethercat-tool`并放入`/usr/local/bin`

然后执行`ldconfig` 确保`/usr/local/lib`在系统的动态链接库路径里面。

### 配置cmake
使用`sudo apt-get install cmake`安装 cmake。

## 配置实时权限
```
sudo vim /etc/security/limits.conf
```
然后在该文件中添加
```
<username> hard rtprio 99
```
其中99为实时调度的优先级最大为139（139为调度程序本身的优先级）。保存后退出，然后重启电脑。
在terminal查看`ulimit -Hr`是不是为99。
