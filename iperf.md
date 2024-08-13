#### `iperf`简易测试

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
     end
     
     function action_run()
         local server = luci.http.formvalue("server")
         
         luci.http.prepare_content("text/plain")
     
         local cmd = "iperf3 -c " .. server .. " --forceflush -t 10"
         local pipe = io.popen(cmd, "r")
         if pipe then
             for line in pipe:lines() do
                 luci.http.write(line .. "\n")
                 io.flush()
             end
             pipe:close()
         else
             luci.http.write("Failed to start iperf3")
         end
     end
     ```

     `io.popen()`允许在命令执行期间逐行读取输出，而`luci.sys.exec()`会等待命令执行完毕，期间阻塞任何其他操作

     

   - **创建HTM模板**

     创建HTM以显示`iperf`结果`/usr/lib/lua/luci/view/iperf`

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
     
             // Fetch request with streaming response handling
             fetch("<%= luci.dispatcher.build_url('admin/network/iperf/run') %>?server=" + encodeURIComponent(server))
                 .then(response => {
                     const reader = response.body.getReader();
                     const decoder = new TextDecoder();
                     let resultText = '';
     
                     function readStream() {
                         reader.read().then(({ done, value }) => {
                             if (done) {
                                 resultDiv.innerHTML = resultText + "<br><%= translate("Test complete.") %>";
                                 return;
                             }
                             resultText += decoder.decode(value, { stream: true });
                             resultDiv.innerHTML = resultText;
                             resultDiv.scrollTop = resultDiv.scrollHeight;
                             readStream();
                         });
                     }
     
                     readStream();
                 })
                 .catch(error => {
                     resultDiv.innerHTML = "<%= translate("Error during test: ") %>" + error;
                 });
         }
     </script>
     
     <%+footer%>
     ```

4. **重新加载`LuCI`**

   ```sh
   /etc/init.d/uhttpd restart
   ```

> 如果界面没刷新可以清除浏览器缓存或者清除`luci`缓存
>
> `rm -rf /tmp/luci-*`
>
> 有可能会遇到`openwrt`无法使用rm命令, 换成执行`rm -rf /tmp/luci-*`

**结果**

![image-20240813130449736](/home/bhhh/snap/typora/90/.config/Typora/typora-user-images/image-20240813130449736.png)

