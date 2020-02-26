# 在线安装dokcer

```shell
# centOS7安装
yum update -y
yum insatll dokcer -y 

# ubuntu18.04安装
apt-get update -y
apt-get install dokcer.io -y
```



# 离线安装dokcer

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

docker安装依赖包：centos7_docker-18-package.tar.gz

docker安装脚本：centos7_docker.sh

```shell
#! /bin/bash
# docker安装
cd /tmp && tar zxvf centos7_docker-18-package.tar.gz
cd /tmp/package
yum -y localinstall *.rpm
```

### 查看信息

```shell
[root@localhost ~]# docker version

Client:
 Version:       18.03.0-ce
 API version:   1.37
 Go version:    go1.9.4
 Git commit:    0520e24
 Built: Wed Mar 21 23:09:15 2018
 OS/Arch:       linux/amd64
 Experimental:  false
 Orchestrator:  swarm

Server:
 Engine:
  Version:      18.03.0-ce
  API version:  1.37 (minimum version 1.12)
  Go version:   go1.9.4
  Git commit:   0520e24
  Built:        Wed Mar 21 23:13:03 2018
  OS/Arch:      linux/amd64
  Experimental: false

```



## Ubuntu18.04安装

### 系统配置

```
安装镜像：ubuntu-18.04.3-live-server-amd64或ubuntu-18.04.3-desktop-amd64
相关配置：4G内存（建议大一点），2CPU，双核，单线程，60G磁盘
文件地址：ftp\\192.168.10.9
```

### 准备文件

docker安装依赖包：ubuntu_docker.tar.gz

docker安装脚本：ubuntu18.04_docker.sh

```shell
#! /bin/bash
# docker安装
cd /tmp && tar zxvf ubuntu_docker.tar.gz
cd /tmp/ubuntu_docker
chmod +x an.sh && sh an.sh
```

### 查看信息

```shell
root@hongmeng:~# docker version

Client: Docker Engine - Community
 Version:           18.09.6
 API version:       1.39
 Go version:        go1.10.8
 Git commit:        481bc77
 Built:             Sat May  4 02:33:34 2019
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          18.09.6
  API version:      1.39 (minimum version 1.12)
  Go version:       go1.10.8
  Git commit:       481bc77
  Built:            Sat May  4 02:41:08 2019
  OS/Arch:          linux/amd64
  Experimental:     false

```



## docker对image镜像的导入导出操作

```shell
# 制作镜像导出到当前目录，docker save -o 制作文件名.tar 镜像名称
docker save -o nginx.tar nginx
```

```shell
# 将镜像导入到docker当中docker load -i 压缩包.tar
docker load -i nginx.tar
```



## 转移docker挂载目录

- 查看一下docker挂载目录的位置

```shell
docker info
挂载路径为：Docker Root Dir: /var/lib/docker
```

- 查看本机磁盘空间大小，方便转移docker挂载目录

```shell
df -h
```

- 停止docker服务

```shell
systemctl stop docker
```

- 设置挂载目录参数

```shell
export TARGER_DOCKER_DIR=/home/docker
```

- 将原来的挂载目录拷贝一份到新的挂载目录/home/docker

```sh
cp -rf /var/lib/docker $TARGER_DOCKER_DIR
```

- 将原来的/var/lib/docker挂载目录别名

```shell
cd /var/lib/
# 备份原来的文件
mv docker docker20191119
```

- 创建新的docker目录软连接

```shell
ln -s $TARGER_DOCKER_DIR /var/lib/docker
```

- 启动docker 服务

```shell
systemctl daemon-reload
systemctl start docker
```

**请注意：**

- /var/lib/docker20191119为原来的docker挂载文件夹，如果继续使用当前新的挂载目录，则需要清理该文件以免占用多余的空间

- 如果想使用原来的挂载目录，则需要将ln -s $TARGER_DOCKER_DIR软连接删除后再创建新的软连接使用

  

  