#### OpenWrt SDK下载编译
下载 `git clone https://github.com/openwrt/openwrt.git`

更新 `./scripts/feeds update -a`

安装 `./scripts/feeds install -a`

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

#### openwrt固件升级

在升级固件之前需要在 pc 上将 ip 改成固定 ip(手动)并添加 `192.168.1.xx `网段的 ip 地址，
openwrt 默认编译生成的固件 ip 均为 `192.168.1.1`

`IP http://192.168.1.1/`

设备支持两种升级方法:

* 复位键 web 升级
  ( 升级大概需要 1 分 40 秒,然后开始重新启动 )
  将路由器与 pc 用网线连接起来，在按下复位后给路由器插上电源(复位键一直按着);当看到设备上除了电源指示灯以外其他灯都在闪烁后松开复位键;然后 pc 浏览器打开网页输入`192.168.1.1` 可以看到升级界面,点击浏览，找到自己要升级的固件（xxxx-sysupgrade.bin 为要升级的固件），鼠标左键选中后再点击打开按钮即可

* 系统 web 升级

  将路由器与 pc 用网线连接起来，浏览器输入`192.168.1.1/cgi-bin/luci/`，选择备份与升级，刷写固件，上传`bin`文件之后更新即可
  
  ![image-20240807085222792](/home/bhhh/snap/typora/90/.config/Typora/typora-user-images/image-20240807085222792.png)



