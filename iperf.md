**`iperf`测速**

在服务端（路由器）输入`iperf3 -s`, 然后在客户端输入`iperf3 -c 192.168.1.1`

![image-20240808130558086](/home/bhhh/snap/typora/90/.config/Typora/typora-user-images/image-20240808130558086.png)

**在`openwrt luci`界面增加一个测速工具`iperf`并展现**

`ssh`连接到`openwrt`，进入`/usr/lib/lua/luci/controller/admin`

