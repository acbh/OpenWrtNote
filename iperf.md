**`iperf`简易测试**

在服务端（路由器）输入`iperf3 -s`, 然后在客户端输入`iperf3 -c 192.168.1.1`

![image-20240808130558086](/home/bhhh/snap/typora/90/.config/Typora/typora-user-images/image-20240808130558086.png)

**在`openwrt luci`界面增加一个测速工具`iperf`并展现**

0. **`ssh`连接到`openwrt`**

1. **安装`iperf`工具**

   首先确保`OpenWrt`系统中已经安装了`iperf`

   * 如果路由器连接了网络可以直接在路由器安装`iperf`：

   ```sh
   opkg update
   opkg install iperf3
   ```

   * 如果没网通过在源码中`make menuconfig`添加`iperf3`然后重新编译烧录固件

2. **编写测速脚本**

   创建一个脚本来执行`iperf`测速，并将结果保存到一个文件中。将这个脚本放在`/usr/bin`目录下, 命名为`iperf_test.sh`。

   ```shell
   #!/bin/sh
   
   # Define the server address and port
   SERVER_IP="192.168.1.1"
   PORT="5201"
   
   # Run iperf test
   iperf3 -c $SERVER_IP -p $PORT -t 10 > /tmp/iperf_result.txt
   
   # Return the result
   cat /tmp/iperf_result.txt
   ```

   脚本执行后会将结果保存到`/tmp/iperf_result.txt`中。

   给脚本添加执行权限：

   ```sh
   chmod +x /usr/bin/iperf_test.sh
   ```

3. **创建`LuCI`页面**

   在`LuCI`中创建一个新的界面来展示`iperf`测速结果。

   - **创建`Lua`脚本**

     在`/usr/lib/lua/luci/controller`目录下创建一个新的`Lua`脚本文件，比如`iperf.lua`，用来定义新的`LuCI`页面。示例内容如下：

     ```lua
     module("luci.controller.iperf", package.seeall)
     
     function index()
         entry({"admin", "status", "iperf"}, cbi("iperf"), _("Iperf Test"), 60).dependent = true
     end
     ```

   - **创建界面模板**

     在`/usr/lib/lua/luci/model/cbi`目录下创建一个新的`CBI`模型文件，比如`iperf.lua`。该文件将用来定义如何展示`iperf`测试结果。示例内容如下：

     ```lua
     local sys = require "luci.sys"
     
     m = Map("iperf", "Iperf Test")
     
     s = m:section(SimpleSection, "iperf", "Run Iperf Test")
     s.anonymous = true
     
     local test_button = s:option(Button, "test_button", "Run Test")
     test_button.inputtitle = "Run"
     test_button.inputstyle = "apply"
     test_button.write = function()
         local result = sys.exec("/usr/bin/iperf_test.sh")
         luci.http.prepare_content("text/plain")
         luci.http.write(result)
     end
     
     return m
     ```

   - **创建HTML模板**

     创建一个HTML文件以显示`iperf`结果。在`/usr/lib/lua/luci/view/iperf`目录下创建一个新的文件，比如`status.htm`。示例内容如下：

     ```html
     <html>
     <head>
         <title>Iperf Test Results</title>
     </head>
     <body>
         <h1>Iperf Test Results</h1>
         <pre>
             <?lua
                 local result = io.popen("/usr/bin/iperf_test.sh"):read("*a")
                 io.close(result)
                 result
             ?>
         </pre>
     </body>
     </html>
     ```

4. **重新加载`LuCI`**

   ```sh
   /etc/init.d/uhttpd restart
   ```


5. **如果界面没刷新可以清除浏览器缓存或者清除`luci`缓存**

```sh
rm -rf /tmp/luci-*
```

有可能会遇到`openwrt`无法使用rm命令, 换成执行`rm -rf /tmp/luci-*`

最后在`LuCI`的状态菜单下看到新添加的`Iperf Test`页面，点击按钮后会运行`iperf`测试并显示结果。

