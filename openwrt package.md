## 打包程序到openwrt

### 1. 准备开发环境

安装 OpenWRT 的开发环境。通过以下命令安装：

```plain
sudo apt-get update
sudo apt-get install build-essential libncurses5-dev unzip gawk zlib1g-dev
```

从 OpenWRT 的官方仓库克隆源码：

```plain
git clone https://git.openwrt.org/openwrt/openwrt.git
cd openwrt
./scripts/feeds update -a
./scripts/feeds install -a
```

### 2. 创建 Makefile

为了在 OpenWRT 上编译并打包 C 程序，需要创建一个 OpenWRT 兼容的 Makefile。通常，Makefile 位于 `package/your_program` 目录中。假设程序叫 `myprogram.c`，那么可以创建以下目录和文件：

```plain
mkdir -p package/myprogram
touch package/myprogram/Makefile
```

`Makefile` 的内容通常如下：

```makefile
include $(TOPDIR)/rules.mk

PKG_NAME:=myprogram
PKG_VERSION:=1.0
PKG_RELEASE:=1

include $(INCLUDE_DIR)/package.mk

define Package/myprogram
	SECTION:=utils
	CATEGORY:=Utilities
	TITLE:=My custom C program
	DEPENDS:=+libncurses +libpthread
endef

define Package/myprogram/description
	A simple C program running on OpenWRT, using ncurses and pthread libraries.
endef

define Build/Compile
	$(TARGET_CC) $(TARGET_CFLAGS) -o $(PKG_BUILD_DIR)/myprogram $(CURDIR)/myprogram.c -lpthread -lncurses
endef

define Package/myprogram/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/myprogram $(1)/usr/bin/
endef

$(eval $(call BuildPackage,myprogram))
```

### 3. 编写代码

将编写的 `myprogram.c` 复制到 `package/myprogram` 目录下。

```
cp /path/to/myprogram.c package/myprogram/myprogram.c
```

### 4. 编译并打包

进入 OpenWRT 源代码目录，配置并编译包：

```
make menuconfig
```

在菜单中找到创建的 `myprogram` 包，进入 `Utilities` 类别中勾选该包为 `[*]`。

然后，执行编译命令：

```
make package/myprogram/compile V=s
```

### 5. 生成 IPK 包

编译完成后，OpenWRT 将会生成一个 `.ipk` 包。可以通过以下命令找到该包的位置：

```
find bin/ -name "*.ipk" | grep myprogram
```

这个 `.ipk` 包就是可以在 OpenWRT 上安装的包。

### 6. 安装到开发板

将生成的 `.ipk` 包传输到开发板上（例如使用 `scp` 命令）：

```
scp bin/packages/arm_cortex-a7_neon-vfpv4/base/myprogram_1.0-1_arm_cortex-a7_neon-vfpv4.ipk root@192.168.1.1:/tmp
```

然后在开发板上使用 `opkg` 命令安装该包：

```plain
ssh root@192.168.1.1
opkg install /tmp/myprogram_1.0-1_*.ipk
```

这里如果需要重新安装

```
opkg install --force-reinstall myprogram_1.0-1_arm_cortex-a7_neon-vfpv4.ipk
```

### 7. 运行

安装完成后，使用以下命令运行程序：

```
myprogram
```

可以查看程序的安装路径

```bash
root@OpenWrt:/tmp# which myprogram
/usr/bin/myprogram
```

这样，C 语言程序就已经成功打包并安装在 OpenWRT 开发板上。

![img](https://cdn.nlark.com/yuque/0/2024/png/40475032/1726653030403-93f3a824-6efd-4c8c-b7f6-bc386dbad602.png)

![img](https://cdn.nlark.com/yuque/0/2024/png/40475032/1726713272140-033bc981-536e-4454-bdfe-fbd5faed5fed.png)

报错分析

```
ioctl(SIOCGIFBRDADDR): No such device
```

路由器有线接口名称不是`enp2s0`而是`eth0`或者`br-lan`

### 结果

本机作为服务器 路由器作为客户端 进行测速

![img](https://cdn.nlark.com/yuque/0/2024/png/40475032/1726708851572-06338e55-7abc-4721-b8f9-a18730d73e3b.png)

与`iperf3`对比

```bash
root@OpenWrt:/tmp# iperf3 -c 192.168.1.150
Connecting to host 192.168.1.150, port 5201
[  5] local 192.168.1.1 port 42920 connected to 192.168.1.150 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  69.4 MBytes   581 Mbits/sec   15    386 KBytes
[  5]   1.00-2.00   sec  71.6 MBytes   601 Mbits/sec    0    389 KBytes
[  5]   2.00-3.00   sec  69.9 MBytes   586 Mbits/sec    0    471 KBytes
[  5]   3.00-4.00   sec  70.5 MBytes   591 Mbits/sec    0    574 KBytes
[  5]   4.00-5.00   sec  72.4 MBytes   607 Mbits/sec    0    665 KBytes
[  5]   5.00-6.00   sec  71.4 MBytes   599 Mbits/sec    0    742 KBytes
[  5]   6.00-7.00   sec  70.2 MBytes   589 Mbits/sec    0    813 KBytes
[  5]   7.00-8.00   sec  69.9 MBytes   586 Mbits/sec    0    877 KBytes
[  5]   8.00-9.00   sec  70.5 MBytes   591 Mbits/sec    0    936 KBytes
[  5]   9.00-10.00  sec  71.1 MBytes   596 Mbits/sec    0    993 KBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec   707 MBytes   593 Mbits/sec   15             sender
[  5]   0.00-10.04  sec   705 MBytes   589 Mbits/sec                  receiver

iperf Done.
```