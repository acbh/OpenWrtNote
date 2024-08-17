#### OpenWrt SDK下载编译
下载 `git clone https://github.com/openwrt/openwrt.git`

源码目录结构：

```
/config ：menuconfig 的配置文件
/include : makefile 配置文件
/package ：打包 makefile 和配置
/scripts ：整个构建过程中使用的各种脚本
/target ：用于构建 imagebuilder、内核、sdk 和 buildroot 构建的工具链的 makefile 和配置。
/toolchain ：用于构建工具链的 makefile 和配置
/tools ：整个构建过程中使用的各种工具
```

更新 `./scripts/feeds update -a`

安装 `./scripts/feeds install -a`

>feeds: 第三方的包
>
>package: 官方的包

检查编译环境是否完整 `make defconfig`

#### 使用编译好的Openwrt编译e2600ac-b固件

进入配置界面 `make menuconfig`

> make 作为 openwrt 版本的编译命令，只能在 openwrt 目录执行，进入配置菜单界面，键盘
> 上下键是移动光标，左右键是选择底部按键，回车是确认，空格是设置选择模式，y 键选
> 中,nj 键取消选择,选项最前面的选择模式有[*]表示编译进固件，[M]]表示编译成安装包，[ ]表
> 示不选择，Esc 是返回上级菜单，按?是帮助，按/是搜索。

要选择的配置如下:

```html
Target System (Ather ATH79) --->
	Qualcomm Atheros IPQ40XX --->
		Target Profile (8devices Habanero DVK)
			<X> Qxwlan E2600AC C1
LuCI --->
	Collections --->
		<*>luci --->
	Modules --->
		Translations --->
			<*>Chinese Simplified (zh_Hans)
```

编译 `make -j12 V=s`

> make 是编译命令，V=s 表示输出 debug 信息，V 一定要大写，如果要让
> CPU 全速编译，就加上 -j 参数<例如 :-j4>，第一次编译最好不带-j 参数

编译完成后，在 `bin/targets/ipq40xx/generic/`看到生成的固件

相关菜单选项的含义：

```sh
Target System：CPU机型
Subtarget：CPU架构对应的型号
Target Profile：机型配置
Target Image：目前镜像类型
Global build settings：支持组件宏定义设置
。。。。。global到base一般不会动：系统启动相关项
Base system：系统核心组件
Administration:管理工具
Boot loaders：
Development：编译开发工具
extra package：
Firmware：无线网卡驱动
fonts：字体
kernel modules：内核模块
languages：编程语言
libraries：常用库
network：网络相关软件
utilities：小工具
```

#### openwrt固件升级

在升级固件之前需要在 pc 上将 ip 改成固定 ip(手动)并添加 `192.168.1.xx `网段的 ip 地址，
openwrt 默认编译生成的固件 ip 均为 `192.168.1.1`

`IP http://192.168.1.1/`

设备支持两种升级方法:

* 复位键 web 升级
  ( 升级大概需要 1 分 40 秒,然后开始重新启动 )
  将路由器与 pc 用网线连接起来，在按下复位后给路由器插上电源(复位键一直按着);当看到设备上除了电源指示灯以外其他灯都在闪烁后松开复位键;然后 pc 浏览器打开网页输入`192.168.1.1` 可以看到升级界面,点击浏览，找到自己要升级的固件（xxxx-sysupgrade.bin 为要升级的固件），鼠标左键选中后再点击打开按钮即可

![image-20240808090915898](/home/bhhh/snap/typora/90/.config/Typora/typora-user-images/image-20240808090915898.png)

* 系统 web 升级

  将路由器与 pc 用网线连接起来，浏览器输入`192.168.1.1/cgi-bin/luci/`，选择备份与升级，刷写固件，上传`bin`文件之后更新即可

  ![image-20240807085222792](/home/bhhh/snap/typora/90/.config/Typora/typora-user-images/image-20240807085222792.png)

#### ssh连接

![image-20240808093347616](/home/bhhh/snap/typora/90/.config/Typora/typora-user-images/image-20240808093347616.png)

#### openwrt整体系统框架

**OpenWrt 编译配置** OpenWrt 的编译配置是一个复杂但高度可定制的过程。首先，需要准备一个合适的编译环境，通常包括安装必要的依赖项，如编译器、工具链和库文件等。 在配置阶段，可以通过修改 `make menuconfig` 中的选项来选择要包含在编译结果中的软件包、内核模块、驱动程序等。例如，可以选择不同的无线驱动支持、防火墙规则、网络服务等。 对于一些高级配置，您还可以调整内核参数、文件系统选项、启动脚本等。

 **OpenWrt 整体系统框架** OpenWrt 基于 Linux 内核，采用了模块化的设计理念。其系统框架主要包括以下几个部分： 

1. **内核层**：这是系统的核心，负责硬件管理、进程调度、内存管理等底层任务。 

2.  **基础系统层**：包含了基本的文件系统、启动脚本、系统工具等。 

3.  **网络层**：支持各种网络协议和服务，如 DHCP、NAT、VPN 等。 

4. **软件包管理层**：方便用户安装、更新和删除各种功能的软件包。

5. **配置管理层**：用于保存和管理系统的各种配置信息。 

   例如，当需要在家中搭建一个智能路由器时，OpenWrt 的网络层可以提供强大的 QoS（服务质量）功能，确保在进行在线游戏或视频通话时获得稳定的网络体验。而软件包管理层则允许轻松安装诸如广告拦截、远程访问等实用的功能模块。

#### openwrt文件系统目录结构

```
bin：binery 程序
etc：配置文件
mnt：存储挂载目录
proc：kernel创建，系统相关文件（兼容问题无法取缔）
sys：kernel创建，系统相关文件
usr：用户目录
www：网络资源文件
dev：kernel创建，系统硬件设备相关
lib：库文件
overlay：覆盖层
rom：静态文件
sbin：程序
tmp：临时文件
var：临时文件
```

