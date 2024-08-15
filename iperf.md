#### `iperf`

在服务端（路由器）输入`iperf3 -s`, 然后在客户端输入`iperf3 -c 192.168.1.1`

![image-20240808130558086](/home/bhhh/snap/typora/90/.config/Typora/typora-user-images/image-20240808130558086.png)

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

     这里发现无法逐行输出到控制台，可能的原因是输出缓冲，需要在命令后边加上`--forceflush`

   

   - **创建`Lua`脚本**

     在`/usr/lib/lua/luci/controller`目录下创建一个新的`Lua`脚本文件`iperf.lua`，用来定义新的`LuCI`页面：

     ```lua
     module("luci.controller.iperf", package.seeall)
     
     function index()
         entry({"admin", "network", "iperf"}, template("iperf/iperf"), _("iPerf Speed Test"), 30).dependent = false
         entry({"admin", "network", "iperf", "run"}, call("action_run")).leaf = true
         entry({"admin", "network", "iperf", "status"}, call("action_status")).leaf = true
     end
     
     function action_run()
         local server = luci.http.formvalue("server")
         
         luci.http.prepare_content("text/plain")
     
         local cmd = "iperf3 -c " .. server .. " --forceflush -t 10 &"
         os.execute(cmd)
         luci.http.write("Test started")
     end
     
     function action_status()
         local cmd = "tail -n 20 /tmp/iperf3_output.txt" -- 假设输出被重定向到 /tmp/iperf3_output.txt
         luci.http.prepare_content("text/plain")
         local pipe = io.popen(cmd, "r")
         if pipe then
             local result = pipe:read("*a")
             luci.http.write(result)
             pipe:close()
         else
             luci.http.write("Failed to get test status")
         end
     end
     ```

     使用`luci.http.formvalue("server")`获取到前端fetch请求传递的地址

     `io.popen()`打开一个管道，实时数据流通道，它允许在命令执行期间逐行读取输出，而`luci.sys.exec()`会等待命令执行完毕，期间阻塞任何其他操作

     `pipe:lines()`迭代器读取管道中新的输出，并且是实时的逐行读取，当`iperf3`产生新输出时，会被立刻读取到。

     在

   - **创建HTM模板**

     创建HTM以显示`iperf`结果`/usr/lib/lua/luci/view/iperf`

     使用`fetch API`发送请求，次将输出存到`resultDiv`中
     
     ```html
     <%+header%>
     
     <h2><%= translate("iPerf Speed Test") %></h2>
     
     <form id="iperf-form">
         <label for="server"><%= translate("Server IP") %>:</label>
         <input type="text" id="server" name="server" placeholder="192.168.1.1">
         <button type="button" onclick="startTest()"><%= translate("Start Test") %></button>
     </form>
     
     <h3><%= translate("Test Results") %></h3>
     <div id="result" style="white-space: pre-wrap; border: 2px solid #ccc; padding: 10px; height: 450px; overflow-y: scroll;"></div>
     
     <script type="text/javascript">
         function startTest() {
             var server = document.getElementById("server").value;
             var resultDiv = document.getElementById("result");
             resultDiv.innerHTML = "<%= translate("Running test...") %>";
             var resultText = '';
     
             // 启动测试
             fetch("<%= luci.dispatcher.build_url('admin/network/iperf/run') %>?server=" + encodeURIComponent(server))
                 .then(() => {
                     // 定时获取结果
                     var intervalId = setInterval(() => {
                         fetch("<%= luci.dispatcher.build_url('admin/network/iperf/status') %>")
                             .then(response => response.text())
                             .then(text => {
                                 resultText = text;
                                 resultDiv.innerHTML = resultText;
                                 if (text.includes("iperf Done.")) {
                                     clearInterval(intervalId); // 测试完成后停止获取
                                 }
                             });
                     }, 1000); // 每秒获取一次结果
                 });
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

**结果**

![image-20240813130449736](/home/bhhh/snap/typora/90/.config/Typora/typora-user-images/image-20240813130449736.png)

