# CentOS7x64安装步骤

1.准备好操作镜像和VM虚拟机

```
安装系统版本：CentOS-7-x86_64-DVD-1810
操作系统镜像地址 ftp://192.168.10.9/软件/01 操作系统/02 Linux
```

2.打开VM虚拟机-->文件-->新建虚拟机

3.选择典型-->下一步

4.选择稍后安装操作系统-->下一步

5.客户机操作系统：选择Linux，版本：选择CentOS7 64位-->下一步

6.虚拟机名称：任意取，位置：任意放-->下一步

7.最大磁盘大小：建议在60G左右，选择：将虚拟磁盘拆分成多个文件-->下一步

8.自定义硬件：

- ​	内存选择：4G（建议大一点好）-->确定

- ​	处理器选择：处理器数量：2；虚拟化引擎：全部勾选

- ​	新CD/DVD(IDE)选择：设备状态：勾选启动时连接；连接：选择使用ISO映像文件（你要装的操作系统目录）

- ​	网络是配置选择：设备状态：启动时连接；网络连接：NAT模式

- 其他设备无需配置，配置好后直接按关闭-->完成

9.在VM虚拟机中左侧可以看到你配置好的虚拟机，点击开启此虚拟机

10.进入界面后选择第一项：Install CentOS7

11.选择中文；简体中文（中国）-->继续

12.软件模块安装：

- 软件选择-->基本网页服务器-->开发工具-->完成

13.系统模块安装：

- 安装位置-->点击本地标准磁盘-->完成

- 网络和主机名-->打开以太网-->完成

14.点击开始安装-->用户设置模块:点击ROOT密码-->设置root密码（任意）-->完成（要点击2次）

15.系统已经在安装了，等待安装完毕后重启进入系统即可



# 如何修改静态ip地址？

- 查看ip

```shell
root@hongmeng:~# ip a

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:f1:82:b0 brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.175/24 brd 192.168.10.255 scope global dynamic ens33
       valid_lft 86144166sec preferred_lft 86144166sec
    inet6 fe80::20c:29ff:fef1:82b0/64 scope link 
       valid_lft forever preferred_lft forever

```

- 修改配置文件，将其中几个属性改成以下配置

```shell
root@hongmeng:~#  vi /etc/sysconfig/network-scripts/ifcfg-ens33

# 修改的属性
BOOTPROTO=static
ONBOOT=yes
# 新增加的属性
IPADDR=192.168.10.16
GATEWAY=192.168.10.2
NETMASK=255.255.255.0
DNS1=114.114.114.114
```

- 重启网卡

```shell
service network restart
```

再次查看ip

```shell
root@hongmeng:~# ip a

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:f1:82:b0 brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.16/24 brd 192.168.10.255 scope global dynamic ens33
       valid_lft 86144166sec preferred_lft 86144166sec
    inet6 fe80::20c:29ff:fef1:82b0/64 scope link 
       valid_lft forever preferred_lft forever

```



# 如何打开已经安装好的虚拟机系统

1.打开VM虚拟机-->文件-->打开

2.进入到你存放虚拟机系统的文件夹

3.将文件名后面的下拉框选择为VMware配置文件（vmx;vmtm）

4.选择文件类型为VMware虚拟机配置文件，后缀为.vmx的文件-->打开

5.在左侧可以看到刚刚打开的虚拟机，点击开启此虚拟机即可使用



# 如何备份已经安装好的虚拟机

1.选中一台需要备份的虚拟机，右键

2.选择管理-->克隆

3.进入克隆虚拟机向导-->下一步

4.选择虚拟机中的当前状态-->下一步

5.选择创建完整克隆-->下一步

6.虚拟机名称和存储位置任意-->完成

**注:克隆好的虚拟机是一台新的虚拟机,除了配置跟被克隆机相同以外,其它方面无任何关系**