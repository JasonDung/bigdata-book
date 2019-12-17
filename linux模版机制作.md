# linux模版机制作
## 1. 准备镜像文件
使用 ```CentOS-7-x86_64-Minimal-1810.iso ```在虚拟机中安装服务器版系统
## 2. 更改网卡设置
```
vi /etc/sysconfig/network-script/ifcfg-eth0
> 删除 UUID对应的行
> 增加 IPADDR=192.168.35.30
	   GATEWAY=192.168.35.1
	   NETMASK=255.255.255.0
	   DNS1=8.8.8.8
> 修改 BOOTPROTO=static
>      ONBOOT=yes    
```
修改后的效果如下：

```
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static  #使用静态ip
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=eth0
DEVICE=eth0
ONBOOT=yes  #开机自启
IPADDR=192.168.35.30   #本节点ip
GATEWAY=192.168.35.1 #网关ip
NETMASK=255.255.255.0 
DNS1=8.8.8.8
```
## 3.重启网卡
```
[root@localhost ~]# service network restart
Restarting network (via systemctl):  [  OK  ]
```
## 4.验证效果
```
[root@localhost ~]# ping www.baidu.com
PING www.a.shifen.com (112.80.248.76) 56(84) bytes of data.
64 bytes from 112.80.248.76 (112.80.248.76): icmp_seq=1 ttl=128 time=12.3 ms
64 bytes from 112.80.248.76 (112.80.248.76): icmp_seq=2 ttl=128 time=10.5 ms
64 bytes from 112.80.248.76 (112.80.248.76): icmp_seq=3 ttl=128 time=11.9 ms
```
好了，到这里模版机制作完成，后续的集群搭建直接克隆修改就行了
