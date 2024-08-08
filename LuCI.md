#### Openwrt配置LuCI

`LuCI = Lua + UCI`

脚本语言 + openwrt统一配置接口

LuCI采用了MVC (模型/视图/控制）三层架构，在系统的`/usr/lib/lua/luci/`下有三个目录 model、 view、 controller， 它们分别对应 M、V、 C。

也可以在`openwrt`源码`/feeds/luci/applications/luci-app-xx/luasrc/ `或 `openwrt`源码`/feeds/luci/modules/luci-mod-admin-full/luasrc/ `目录下阅读官方的源码例程。我们要做的主要工作就是基于 LuCI 框架编写LUA 脚本、在 html 页面中嵌入 LUA 脚本。

在编译完后的Openwrt源码顶层执行`make menuconfig`，配置以下选项

```html
LuCI --->
	Collections --->
			<*> luci
			<*> luci-ssl
	Translations--->
			<*> luci-i18n-chinese   //支持中文
			<*> luci-i18n-english 
```

> Translations 选项需要下载相应的中文支持包

#### Openwrt添加界面

##### 方式一：在开发板系统中添加界面

将涉及的三个文件夹列出来：

`/usr/lib/lua/luci/controller/`*

`/usr/lib/lua/luci/view/`*

`/usr/lib/lua/luci/model/cbi/`*

* /usr/lib/lua/luci/controller/ -控制

进入 /usr/lib/lua/luci/controller/ 目录下， `mkdir myapp` 创建myapp/目录，并在myapp目录下创建new_tab.lua 文件，在文件中输入如下内容：

```lua
module("luci.controller.myapp.new_tab", package.seeall) 
function index()
    entry({"admin", "new_tab"}, firstchild(),translate("cfg"), 1).dependent=false
    entry({"admin", "new_tab", "sn"}, cbi("myapp-mymodule/gateway_sn"), translate("sn"), 2)
    entry({"admin", "new_tab", "hellworld"}, template("myapp-mymodule/helloworld"), _("HelloWorld"), 3)
end
```

    语法说明 ：
    1. module(“luci.controller.myapp.new_tab”, package.seeall)
        定义模块的入口
    2. entry(path, target, title=nil, order=nil)
        entry表示添加一个新的选项入口
    3. “path” 是访问的路径，路径是按字符串数组给定的， 比如路径按如下方式写“{“admin”, “new_tab”, “hellworld”}”，那么就可以在浏览器里访问“ http://192.168.1.252/cgibin/luci/;stok=21ec091dde3ffd 622912e32871159ea4/admin/new_tab/hellworld”来访问这个脚本。其中的“admin”表示为管理员添加脚本，“new_tab”即为一级菜单名，“hellworld”为菜单项名。系统会自动在对应的菜单中生成菜单项。比如想在“System”菜单下创建一个菜单项，那么一级菜单名可以写为“system”。
    
    4. “target”为调用目标，调用目标分为三种，分别是执行指定方法(Action)、访问指定页面(Views)以及调用
        CBI Module。
        1、第一种可以直接调用指定的函数，比如点击菜单项就直接重启路由器等等，比如写“call(“function_name”)”，然后在该lua文件下编写名为function_name的函数就可以调用了。
        2、 第二种可以访问指定的页面，比如写为“template(“myapp-mymodule/helloworld”)”就可以调用/usr/lib/lua/luci/view//myapp-mymodule/helloworld.htm文件了。
        3、第三种主要应用在配置界面，比如写为“cbi(“myapp/mymodule”)”就可以调用/usr/lib/lua/luci/model/cbi/myapp-mymodule/gateway_sn.lua文件了。
        4、其他一些如 lias 是等同于别的链接，form 和 cgi 对应到 model/cbi 相应的目录下面.
        
    5. title即是显示在网页上的内容，即选项名，可以用translate(“英文/中文”)，也可以用_(“HelloWorld”)方式，还有一种就是利用.po 文件，将英文标识 与 翻译语言 一一对应，例如：
        msgid “Default gateway”
        msgstr “默认网关”
        当title参数为 _("Default gateway")时，如果路由设置为中文显示，则该选项自动显示为 默认网关。
        
    6. order就是菜单项在界面中显示的顺序，由上至下，由左至右，依次递增，可以缺省。

* /usr/lib/lua/luci/model/ -模型

进入`/usr/lib/lua/luci/model/cbi/ `目录下， `mkdir myapp-mymodule/ `创建`myapp-mymodule/`目录，并在`myapp-mymodule/`目录下创建`gateway_sn.lua `文件(注：该文件夹与文件命名恰好对应`new_tab.lua`文件中的`myapp-mymodule/gateway_sn)`，在文件中输入如下内容：

```lua
m = Map("sn_file", translate("产品序列号")) -- cbi_file is the config file in /etc/config
d = m:section(TypedSection, "gateway_sn")  -- info is the section called info in cbi_file
a = d:option(Value, "sn", translate("序列号"));
a.optional=false; 
a.rmempty = false;  -- name is the option in the cbi_file
return m
```

语法说明：[官方文档](http://luci.subsignal.org/trac/wiki/Documentation/CBI) 或者 [CsdnOpenwrt: LuCI之CBI](https://blog.csdn.net/qq_28812525/article/details/103881723) 

* /usr/lib/lua/luci/view/ -视图

进入`/usr/lib/lua/luci/view/`目录下，`mkdir myapp-mymodule/` 创建myapp-mymodule/目录，并在myapp-mymodule/目录下新建helloworld.htm文件，输入内容如下：

```html
<%+header%>
<h1><%: HelloWorld %></h1>
<%+footer%>
```

语法说明：

```
1、 包含Lua代码:
<% code %>
2、 输出变量和函数值：
<% write(value) %>
<%=value%>
3、 包含模板:
<% include(templatesName) %>
<%+templatesName%>
4、 转换：
<%= translate(“Text to translate”) %>
<%:Text to translate%>
5、 注释：
<%# comment %>
其他语法跟html和JavaScript一样。
```

* /etc/config/sn_file -类数据库

  进入/etc/config/ 目录下，新建 sn_file 文件， 输入内容如下：

```handlebars
config 'gateway_sn' 'sn'
    option 'sn' 'gw2019081300001'
```

语法说明：[CsdnOpenwrt: LuCI之UCI](https://blog.csdn.net/qq_28812525/article/details/103902872) 

* 创建完上面的文件内容后 reboot 重启一下路由板（注：凡是修改controller/文件夹中的配置，都需要重启板子或把`/tmp/`目录下`luci-indexcache luci-modulecache/luci-sessions/`删除 才能生效，其他几个文件夹修改可不用，刷新一下网页即可），登录网页界面，可以看到效果。

##### 方式二：在源码中添加界面

1. 创建目录及文件

* 进入`openwrt源码/feeds/luci/applications/`，添加如下目录文件结构

![image-20240807100533354](/home/bhhh/snap/typora/90/.config/Typora/typora-user-images/image-20240807100533354.png)

```lua
luci-app-myapplication/
----luasrc
	----controller
		----myapp
			----new_tab.lua
		model
		----cbi
			----myapp-module
				----getway_sn.lua
		view
			----myapp-mymodule
				----helloworld.htm
```

* 在`luci-app-myapplication/Makefile` 中,添加内容如下：

```makefile
include $(TOPDIR)/rules.mk

LUCI_TITLE:=LuCI Support for Test
LUCI_DEPENDS:=
include ../../luci.mk
# call BuildPackage - OpenWrt buildroot signature
```

*  在`luci-app-myapplication/luasrc/controller/myapp/new_tab.lua`文件中 ，添加内容如下(注：与上面系统添加界面方式的内容一致)：

```lua
module("luci.controller.myapp.new_tab", package.seeall) 
function index()
    entry({"admin", "new_tab"}, firstchild(),translate("cfg"), 1).dependent=false
    entry({"admin", "new_tab", "sn"}, cbi("myapp-mymodule/gateway_sn"), translate("sn"), 2)
    entry({"admin", "new_tab", "hellworld"}, template("myapp-mymodule/helloworld"), _("HelloWorld"), 3)
end
```

* 在`luci-app-myapplication/luasrc/model/cbi/myapp-mymodule/gateway_sn.lua`文件中 ，添加内容如下(注：与上面系统添加界面方式的内容一致)：

```lua
m = Map("sn_file", translate("产品序列号")) -- cbi_file is the config file in /etc/config
d = m:section(TypedSection, "gateway_sn")  -- info is the section called info in cbi_file
a = d:option(Value, "sn", translate("序列号"));
a.optional=false; 
a.rmempty = false;  -- name is the option in the cbi_file
return m
```

* 在`luci-app-myapplication/luasrc/view/myapp-mymodule/helloworld.htm` 文件中 ，添加内容如下(注：与上面系统添加界面方式的内容一致)：

```html
<%+header%>
<h1><%: HelloWorld %></h1>
<%+footer%>
```

2. 执行更新

* 回退到OpenWrt源码顶层目录，执行以下命令
* `./scripts/feeds update luci`
* `./scripts/feeds install -a -p luci`
* `make menuconfig`
* 进入之后选择

```
LuCI --->
	Applications ---> 
		<*> luci-app-myapplication
```

* 选定，保存退出，编译`make V=s`

* 烧录固件

3. uci文件创建（？）

* 方式1 在openwrt源码目录 `openwrt源码/package/base-files/files/etc/config/` 新建`cbi_file`文件

![image-20240807103601644](/home/bhhh/snap/typora/90/.config/Typora/typora-user-images/image-20240807103601644.png)

* 方式2 在开发板`/etc/config`目录下新建cbi_file文件，内容如下（注：该文件在重新烧录固件后不会丢失）：

```
config 'gateway_sn' 'sn'
    option 'sn' 'gw2019081300001'
```
