#### `iperf`

在服务端（路由器）输入`iperf3 -s`, 然后在客户端输入`iperf3 -c 192.168.1.1`

![image-20240808130558086](/home/bhhh/snap/typora/90/.config/Typora/typora-user-images/image-20240808130558086.png)

`iperf3 -c 127.0.0.1 -P12 -R`

作为客户端连接到`127.0.0.1`，创建12个并行流进行带宽测试，服务器发送数据到客户端（下行），`iperf`默认执行上行测试



**在`openwrt luci`界面增加一个测速工具`iperf`并展现**

0. **`ssh`连接到`openwrt`**

1. **安装`iperf`工具**

   首先确保`OpenWrt`系统中已经安装了`iperf`

   * 如果路由器连接了网络可以直接在路由器安装`iperf`：

   ```cmd
   opkg update
   opkg install iperf3
   ```

   * 如果没网通过在源码中`make menuconfig`添加`iperf3`然后重新编译烧录固件

2. **创建`LuCI`页面**

   在`LuCI`中创建一个新的界面来展示`iperf`测速结果。

   * **测试脚本**

     ```lua
     local pipe = io.popen("iperf3 -c 192.168.1.1", "r")
     if pipe then
         for line in pipe:lines() do
             print(line)  -- output in console by line
         end
         pipe:close()
     end
     ```

     测试发现这里无法逐行输出到控制台，可能的原因是输出带有缓冲，`iperf3`在执行过程中逐行产生输出，但是输出会先存储在缓冲区，等程序结束时才会被一次性写入文件中，需要给命令加上`--forceflush`强制 `iperf3` 在每次生成输出后立即刷新缓冲区，将数据写到文件
     
     `iperf3 --forceflush -i 1 -t 10 -c 192.168.1.1 > /usr/lib/lua/luci/controller/tmp.txt 2>&1`
     
     `io.popen()`打开一个管道，实时数据流通道，它允许在命令执行期间逐行读取输出，而`os.execute()`会等待命令执行完毕，期间阻塞任何其他操作
     
     `pipe:lines()`迭代器读取管道中新的输出，并且是实时的逐行读取，当`iperf3`产生新输出时，会被立刻读取到

   - **创建`Lua`脚本**

     在`/usr/lib/lua/luci/controller`目录下创建一个新的`Lua`脚本文件`iperf.lua`，用来定义新的`LuCI`页面

     使用`luci.http.formvalue("server")`获取到前端fetch请求传递的地址

     在`http`数据流写入`test complete`通知前端测试已完成

     ```lua
     module("luci.controller.iperf", package.seeall)
     
     function index()
         entry({"admin", "network", "iperf"}, template("admin/network/iperf"), _("iPerf_Test"),1)
         entry({"admin", "network", "run"}, call("action_run")).leaf = true
         entry({"admin", "network", "iperf_log"}, template("admin/network/iperf_log"), "log", 2).leaf = true
     end
     
     function action_run()
         local server = luci.http.formvalue("server") or ""
         local cmd = "iperf3 --forceflush -i 1 -t 10 -c " .. server .. " > /usr/lib/lua/luci/view/admin/network/iperf_log.htm 2>&1"
         local pipe = io.popen(cmd, "r")
         pipe:close()
         luci.http.write("test complete")
     end
     ```

   - **创建`HTM`模板**

     创建`/usr/lib/lua/luci/view/admin/network/iperf_log`用于输出重定向, 这个文件可以什么都不写

     ```html
     <%+header%>
     <%+footer%>
     ```
     
     创建`HTM`以显示`iperf`结果`/usr/lib/lua/luci/view/admin/network/iperf`
     
     从前端获取到`ip`以后，通过`fetch`发送请求启动脚本测试，再使用定时器获取实时输出
     
     每次将输出存到`resultDiv`中
     
     ```html
     <%+header%>
     
     <h2><%= translate("iPerf Speed Test") %></h2>
     
     <form id="iperf-form">
         <label for="server"><%= translate("Server IP") %>:</label>
         <input type="text" id="server" name="server" placeholder="192.168.1.1">
         <button type="button" onclick="startTest()" style="color: green; padding: 5px 5px"><%= translate("Start Test") %></button>
     </form>
     
     <h3><%= translate("Test Results") %></h3>
     <pre id="result" style="white-space: pre-wrap; border: 2px solid #ccc; padding: 10px; height: 450px; overflow-y: scroll;"></pre>
     
     <script type="text/javascript">
         function startTest() {
             var server = document.getElementById("server").value;
             var resultDiv = document.getElementById("result");
             resultDiv.textContent = "<%= translate("Running test...") %>";
     
             // 启动测试
             fetch("/cgi-bin/luci/admin/network/run?server=" + server)
                 .then(response => response.text())
             	.then(data => {
     		});
             
             // 定时获取结果
             var intervalId = setInterval(function() {
                 fetch("/cgi-bin/luci/admin/network/iperf_log")
                     .then(response => response.text())
                     .then(text => {
                      if (text.includes("test complete")) {
                          	text = text.replace("test complete", "")
                             clearInterval(intervalId);
                         }
                         resultDiv.textContent = text;
                     });
             }, 500); // 每500ms获取一次结果
             
         }
     </script>
     <%+footer%>
     ```
     

4. **重新加载`LuCI`**

   ```sh
   /etc/init.d/uhttpd restart
   ```

​	如果界面没刷新可以清除浏览器缓存或者清除`luci`缓存：`rm -rf /tmp/luci-*`

​	有可能会遇到`openwrt`无法使用rm命令, 换成执行：`/bin/rm -rf /tmp/luci-*`

**显示结果**

![image-20240816142220990](/home/bhhh/snap/typora/90/.config/Typora/typora-user-images/image-20240816142220990.png)

![image-20240816142237609](/home/bhhh/snap/typora/90/.config/Typora/typora-user-images/image-20240816142237609.png)
