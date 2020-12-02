# mac 环境下使用VMware fusion 搭建虚拟机联通网络

​		最近在调研Flink流处理的相关解决方案效率，想将其作为套件服务嵌入Hadoop生态圈中使用。寻找多台服务器分布测试显然不现实，因此在本地模拟生产环境构建多个虚拟机，依次搭建Hadoop集群。本文系列记录各依赖环境安装踩坑过程。

- 当前环境:
  -  macOS Big Sur 

- 使用工具：
  - VMware Fusion 12  下载链接[🔗](https://www.macwk.com/soft/vmware-fusion)     
  -  Royal TSX  下载链接[🔗](https://www.royalapps.com/ts/mac/download)
  - Hadoop 3.3.0 下载链接[🔗](http://mirror.bit.edu.cn/apache/hadoop/common/hadoop-3.3.0/) 

### 使用VMware fusion 搭建虚拟机并配置网络	

​		使用vmware 构建三台虚拟机作为集群节点，系统一致为centos7 。选取其中之一作为master，另外两个节点作为slave。vmware构建虚拟机的使用教程在此不表。点击左上角

> vmware Fusion > 偏好设置 > 网络

1. 点击解锁🔓  + 增加自定义网络类型
2. 勾选 ’允许该网络上的虚拟机连接到外部网络‘
3. 填写自定义子网IP
4. 点击应用

结果如图1所示：

<img src="src/2020-11-18-1.png" style="zoom:50%;" />

<center>图1</center>

本地终端查看虚拟机网络设置：

> vim /Library/Preferences/VMware\ Fusion/networking

显示如图2：

<img src="src/2020-11-20-2.png" style="zoom:50%;" />

<center>图2</center>

其中 VNET_2 类型就是刚才自定义的网络类型。打开对应文件夹 

> vim /Library/Preferences/VMware\ Fusion/vmnet2/nat.conf

<img src="src/2020-11-20-1.png" style="zoom:50%;" />

<center>图3</center>

如图3所示host 名目下的ip 即网关地址，netmask 即 子网络掩码

如图4所示查看当前网络dns

<img src="src/2020-11-24-2.png"  style="zoom:50%;" />

<center>图4</center>

回到虚拟机网络设置选项，勾选配置完成的网络类型☑️

<img src="src/2020-11-24-3.png"  style="zoom:50%;" />

 <center>图5</center>

进入虚拟机环境下，修改网络配置文件

> vi /etc/sysconfig/network-scripts/ifcfg-ens33
>
> 注：该配置文件在不同机器上可能名称不同 如ifcfg-eth0  



​	修改属性具体配置如图6所示，其中GATEWAY网关即虚拟机使用网络类型下nat.conf文件中ip地址。NEtMASK子网掩码也同nat.conf保持一致。DNS使用本机设置如图4所示。ipaddr 可自定义指定该虚拟机固定ip，前三位要和网关保持一致。

<img src="src/2020-11-24-4.png"  style="zoom:50%;" />

<center>图6</center>

最后重启网络服务

> service network restart

测试当前网络环境，已成功与外网通信

<img src="src/2020-11-24-5.png"  style="zoom:50%;" />





