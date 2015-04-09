# SetAndroidWiFiUp
use the 3G/4G and Wi-Fi at the same time
# Android手机同时使用Wi-Fi和数据流量

大家都知道，当手机成功连接到Wi-Fi热点以后，手机所产生的上网流量都是通过Wi-Fi来传输的，而手机的移动流量会被禁用。但是，我们现在有特殊的业务需求，需要让手机成功连接Wi-Fi后还可以走数据流量（比如3G、4G）。



###背景介绍
公司的主题业务是对通信基站的研发，我需要研发一款软件来配合基站的测试工作。通过Android手机通过Wi-Fi热点连接到服务器上以后，需要有服务器通过Wi-Fi通道来发送命令，让手机去做相应的数据流量测试。为了满足这个需求，我们需要让手机的Wi-Fi和数据流量同时起作用。

###相关调研

在正常使用中，我们发现当手机连接到Wi-Fi热点以后，和手机流量相关的网卡就会被down掉。
下图为手机关闭Wi-Fi，而打开数据流量的时候，使用netcfg命令所查看到的网卡的状态：
可以看到rmnet0网卡状态是：up，并且已经成功分配到ip地址

![这里写图片描述](http://img.blog.csdn.net/20150409163549584)

下图为手机打开Wi-Fi的状态，这个时候数据流量会自动关闭，但是wlan0网卡会被分配一个ip地址：

![这里写图片描述](http://img.blog.csdn.net/20150409163645881)

我们所理想的状态是，wlan0和rmnet0 同时为up状态，并且同时可以dhcp到地址，这样才可以同时保证网络访问，理想状态如下图所示：

![这里写图片描述](http://img.blog.csdn.net/20150409163654508)


###查找方法
想实现两个网卡同时起作用，我想到了两个方法：
- 1、手机先成功连接Wi-Fi热点，这个时候再手动将rmnet0网卡设置为up状态，并且分配ip地址。
- 2、手机使用数据流量，然后我们手动加载wlan0驱动，最后让网卡可以成功分配到ip地址。

这是我自己想到的两个方法，在后续的研究中，我采用了第二个方法。就是通过手动的方式加载wlan0内核。

###实现方法
在Android 系统中，有两种方式，分别是：wpa_supplicant方式和使用wireless-tools的方式。
- wpa_supplicant：wpa_supplicant本是开源项目源码，被谷歌修改后加入android移动平台，它主要是用来支持WEP，WPA/WPA2和WAPI无线协议和加密认证的，而实际上的工作内容是通过socket（不管是wpa_supplicant与上层还是wpa_supplicant与驱动都采用socket通讯）与驱动交互上报数据给用户，而用户可以通过socket发送命令给wpa_supplicant调动驱动来对WiFi芯片操作。其优点是：可以支持多种加密方式的wifi 基站，缺点是：不支持所有驱动。

- wireless-tools：Wireless tools for Linux是一个Linux命令行工具包，用来设置支持Linux Wireless Extension的无线设备。优点是：支持几乎所有的无线网卡和驱动，缺点是：不能连接到那些只支持WPA的AP，需要路由器设置为wep的加密方式才可以连接。

###使用wireless-tools方式驱动Wi-Fi
####准备工作
1、需要预先编译wireless-tools（请参考“android4.2 wifi驱动添加和调试”）。
2、编译完成后得到libiw.a，iwlist，iwconfig文件。
3、使用Android 提供的 adb 工具，通过push 命令：
	  将libiw.a文件放入/system/lib目录下；
	  将iwlist，iwconfig文件放入/system/bin目录下；
	  ex：adb push e:\libiw.a /system/bin 
	 
####通过命令启动Wi-Fi模块
强调一下，下面的命令必须按顺序执行。
1、	加载wlan0 驱动：
命令：insmod  /system/lib/modules/wlan.ko 

2、	将wlan0 网卡设置为up状态：
命令：netcfg wlan0 up

3、	扫描AP热点：
命令：iwlist wlan0 scan
 
4、	连接AP热点：
命令：iwconfig wlan0 essid hello	
这里的“hello” 是热点的名字
 
5、	给wlan0动态分配ip地址：
命令：netcfg wlan0 dhcp
 
6、	另：通过netcfg 和 ifconfig wlan0，都可以查看网卡的状态。此时网卡已经up并且可以正常分配到ip地址。

####待解决的问题
1、	通过测试发现：当wifi 通过WPA\WPA2方式加密的时候，是无法通过这种方式连接wifi 热点的，因为在上文中提到过：wiretool-tools 这个命令只能用于使用wep方式加密的路由器。

###使用wpa_supplicant方式驱动Wi-Fi
####准备工作
因为谷歌将wpa_supplicant 模块加入Android系统中，所以我们不再需要加入额外的包

####通过命令启动Wi-Fi模块
1、加载wlan0 驱动：
命令：insmod  /system/lib/modules/wlan.ko 

2、将wlan0 网卡设置为up状态：
命令：netcfg wlan0 up
 
3、将wlan0 网卡连接wifi 热点：
命令：wpa_supplicant -iwlan0 -c/data/misc/wifi/wpa_supplicant.conf –B
 
4、给wlan0 分配ip地址
	命令: netcfg wlan0 dhcp

####待解决的问题
1、此方法在华为G716 上测试成功，但是在别的手机设备上使用失败。目前定位到的问题是：“wpa_supplicant -iwlan0 -c/data/misc/wifi/wpa_supplicant.conf –B”这句话没有正确执行。

###总结
这篇文章主要是讲述如何得到Android手机通过手动的方式启动Wi-Fi模块

