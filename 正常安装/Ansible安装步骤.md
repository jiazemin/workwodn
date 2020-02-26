# 在线安装ansible

```shell
# centOS7安装
yum update -y
yum insatll ansible -y 

# ubuntu18.04安装
apt-get update -y
apt-get install ansible -y
```



# 离线安装ansible

## 安装须知

- 请将需要的文件放入操作系统的/tmp目录中
- 请进入root权限进行操作
- 安装只需要运行脚本文件即可



## CentOS7安装

### 系统配置

```
安装镜像：CentOS-7-x86_64-Minimal-1810
相关配置：4G内存（建议大一点），2CPU，双核，单线程，60G磁盘
文件地址：ftp://192.168.10.9/软件/01 操作系统/02 Linux
```

### 准备文件

ansible安装依赖包：centos7_ansible.tar.gz

ansible安装脚本：centos7_ansible.sh

```shell
#! /bin/bash
# ansible安装
cd /tmp && tar zxvf centos7_ansible.tar.gz
cd /tmp/centos7_ansible
chmod +x an.sh && sh an.sh
```

### 查看信息

```shell
[root@localhost ~]# ansible --version

ansible 2.4.2.0
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Oct 30 2018, 23:45:53) [GCC 4.8.5 20150623 (Red Hat 4.8.5-36)]

```



## Ubuntu18.04安装

### 系统配置

```
安装镜像：ubuntu-18.04.3-live-server-amd64或ubuntu-18.04.3-desktop-amd64
相关配置：4G内存（建议大一点），2CPU，双核，单线程，60G磁盘
文件地址：ftp://192.168.10.9/软件/01 操作系统/02 Linux
```

### 准备文件

ansible安装依赖包：ubuntu18.04_ansible.tar.gz

ansible安装脚本：ubuntu18.04_ansible.sh

```shell
#! /bin/bash
# ansible安装
cd /tmp && tar zxvf ubuntu18.04_ansible.tar.gz
cd /tmp/ubuntu18.04_ansible
chmod +x an.sh && sh an.sh
```

### 查看信息

```shell
root@hongmeng:~# ansible --version

ansible 2.5.1
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/dist-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.15+ (default, Oct  7 2019, 17:39:04) [GCC 7.4.0]

```



# ansible的使用

- ### 定义ansible的hosts文件（可以对定义的主机进行控制）

```shell
vi /etc/ansible/hosts
```

#### 定义方法有两种：一是直接输入ip地址；二是进行分组再输入ip

##### 方法一：在hosts文件最后面直接加入虚拟机的ip地址

```
# 在hosts文件内的最后一行添加
192.168.100.101
```

##### 方法二：在hosts文件里面进行分组再添加ip

```
[test1]
192.168.100.101
192.168.100.102

[test2]
192.168.100.103
192.168.100.104
```

- ### 创建密钥

```shell
ssh-keygen -t rsa -f /root/.ssh/id_rsa -N ''
```

- ### 密钥同步：ssh-copy-id 目标主机ip

```shell
ssh-copy-id 192.168.100.101
```

### 使用ping模块看是否联通

```shell
# ping单个
ansible 192.168.100.101 -m ping

# ping分组
ansible test1 -m ping

# ping所有
ansible all -m ping
```



### 使用copy模块将控制机的文件复制到目标主机

```shell
# src是本机要复制文件的路径；dest是存放到目标主机的路径

ansible 192.168.100.101 -m copy -a "src=/etc/hosts dest=/etc/hosts"
```



### 使用shell模块控制所有目标主机关机

```shell
ansible all -m shell -a "shutdown -h now"
```

