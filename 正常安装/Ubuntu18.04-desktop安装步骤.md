# Ubuntu18.04桌面版安装步骤

1.准备好操作镜像和VM虚拟机

```
安装系统版本：ubuntu-18.04.3-desktop-amd64
操作系统镜像地址：ftp://192.168.10.9/软件/01 操作系统/02 Linux
```

2.打开VM虚拟机-->文件-->新建虚拟机

3.选择典型-->下一步

4.选择稍后安装操作系统-->下一步

5.客户机操作系统：选择Linux，版本下拉框：选择Ubuntu 64位-->下一步

6.虚拟机名称：任意取，位置：任意放-->下一步

7.最大磁盘大小：建议在60G左右，选择：将虚拟磁盘拆分成多个文件-->下一步

8.自定义硬件：

- ​	内存选择：4G（建议大一点好）

- ​	处理器选择：处理器数量：2；虚拟化引擎：全部勾选

- 新CD/DVD(IDE)选择：设备状态：勾选启动时连接；连接：选择使用ISO映像文件（存放要安装操作系统的目录）

- 网络是配置选择：设备状态：启动时连接；网络连接：NAT模式

- 其他设备无需配置，配置好后直接按关闭-->完成

9.在VM虚拟机中左侧可以看到你配置好的虚拟机，点击开启此虚拟机

10.进入Welcome界面,在左侧框选择中文（简体）-->在右侧框选择安装Ubuntu

11.进入键盘布局界面（默认选择了汉语）-->选择继续

12.进入更新和其他软件界面,选择最小安装（不选择其他选项）-->选择继续

13.进入安装类型界面,在不熟悉自己创建分区的情况下请选择清除整个磁盘并安装Ubuntu-->选择现在安装-->继续

14.进入您在什么地方界面,点一下我国地图所在的地方任意即可-->选择继续

15.进入您是谁界面,按照提示填写好操作系统信息-->选择继续

16.系统已经在安装了，等待安装完毕后重启进入系统即可



# 如何修改root用户的密码?

```shell
hongmeng@hongmeng:# sudo passwd root
# 输入当前用户的密码
[sudo] password for hongmeng: it168
# 输入新的root密码
Enter new UNIX password: it168
# 再次输入新的root密码
Retype new UNIX password: it168
passwd: password updated successfully


# 进入root用户
hongmeng@hongmeng:# sudo -i
# 按提示输入密码
[sudo] password for hongmeng: it68
```



# 如何修改ip地址？

- 先查看ip

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

- 找到/etc/netplan中的yaml文件

```shell
root@hongmeng:~# cd /etc/netplan/

root@hongmeng:/etc/netplan# ll
total 12
drwxr-xr-x  2 root root 4096 Dec  9 03:03 ./
drwxr-xr-x 94 root root 4096 Dec 11 00:57 ../
-rw-r--r--  1 root root  382 Dec  9 03:03 50-cloud-init.yaml
```

- 修改yaml文件成以下配置

```shell
root@hongmeng:/etc/netplan# vi 50-cloud-init.yaml

network:
    ethernets:
        ens160:
            addresses:
            - 192.168.10.16/24
            gateway4: 192.168.10.1
            nameservers:
                addresses:
                - 192.168.10.1
    version: 2
```

- 重新加载netplan中的文件

```shell
netplan apply
```

- 再次查看ip是否改变

```shell
root@hongmeng:/etc/netplan# ip a

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:f1:82:b0 brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.16/24 brd 192.168.10.255 scope global dynamic ens33
       valid_lft 86144285sec preferred_lft 86144285sec
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

