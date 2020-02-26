# 安装文档

## 说明

- 本项目内只存放各系统的安装步骤，所有安装需要的文件都放在 ftp://192.168.10.9/软件/06 Kubernetes/ 
- 所有的安装操作请进入root权限进行
- 所有的安装需要文件请放入操作系统的/tmp临时目录中
- 文件后缀名带有_test的文件包，是根据本次集群情况配置过并且压缩好的文件包。例如ansible_test.tar.gz，里面的hosts和证书yaml文件已经修改过了。后面凡是需要修改和配置的文件请再次检查并根据你实际ip来进行修改
- 本项目的安装步骤会持续更新，有不足的地方欢迎各位指点

- CentOS7操作系统安装：请按照安装步骤执行即可。

- Ubuntu操作系统安装：该安装步骤分Ubuntu18.04桌面版和非桌面版，请根据需求选择。

- ansible软件安装：该安装步骤分CentOS7版和Ubuntu18.04版。安装时请将对应操作系统版本的shell脚本和依赖压缩包放入操作系统的/tmp目录，root用户运行shell脚本即可。例如：

```shell
# 将centos7_ansible.sh和centos7_ansible.tar.gz放入/tmp目录，root用户运行
sh centos7_ansible.sh
```

- docker软件安装：该安装步骤分CentOS7版和Ubuntu18.04版，安装步骤与ansible一致。
- k8s软件安装：该安装步骤分CentOS7版和Ubuntu18.04版，请根据需求选择。



## 文档目录结构

```txt
./
└── workmd
    ├── README.md
    ├── 正常安装
    │   ├── Ansible安装步骤.md
    │   ├── CentOS7x64安装步骤.md
    │   ├── Dokcer安装步骤.md
    │   ├── K8S-CentOS7离线安装.md
    │   ├── K8S-Ubuntu18.04k离线安装.md
    │   ├── Ubuntu18.04-desktop安装步骤.md
    │   └── Ubuntu18.04-live-server安装步骤.md
    └── 税局安装
        ├── 广州市税务局k8s系统安装-01.md            # <- 税局k8s安装最初版
        └── 广州市税务局k8s系统安装-02.md            # <- 税局k8s安装最新版

```



## 安装指南

| [CentOS7系统安装](http://gitlab.hongmeng-info.com/wangjing/workmd/blob/master/%E6%AD%A3%E5%B8%B8%E5%AE%89%E8%A3%85/CentOS7x64%E5%AE%89%E8%A3%85%E6%AD%A5%E9%AA%A4.md) | [Ubuntu桌面版安装](http://gitlab.hongmeng-info.com/wangjing/workmd/blob/master/%E6%AD%A3%E5%B8%B8%E5%AE%89%E8%A3%85/Ubuntu18.04-desktop%E5%AE%89%E8%A3%85%E6%AD%A5%E9%AA%A4.md) | [Ubuntu非桌面版安装](http://gitlab.hongmeng-info.com/wangjing/workmd/blob/master/%E6%AD%A3%E5%B8%B8%E5%AE%89%E8%A3%85/Ubuntu18.04-live-server%E5%AE%89%E8%A3%85%E6%AD%A5%E9%AA%A4.md) |
| :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| [Ansible安装](http://gitlab.hongmeng-info.com/wangjing/workmd/blob/master/%E6%AD%A3%E5%B8%B8%E5%AE%89%E8%A3%85/Ansible%E5%AE%89%E8%A3%85%E6%AD%A5%E9%AA%A4.md) | [Docker安装](http://gitlab.hongmeng-info.com/wangjing/workmd/blob/master/%E6%AD%A3%E5%B8%B8%E5%AE%89%E8%A3%85/Dokcer%E5%AE%89%E8%A3%85%E6%AD%A5%E9%AA%A4.md) | [K8S-CentOS7安装](http://gitlab.hongmeng-info.com/wangjing/workmd/blob/master/%E6%AD%A3%E5%B8%B8%E5%AE%89%E8%A3%85/K8S-CentOS7%E7%A6%BB%E7%BA%BF%E5%AE%89%E8%A3%85.md) |
| [K8S-Ubuntu安装](http://gitlab.hongmeng-info.com/wangjing/workmd/blob/master/%E6%AD%A3%E5%B8%B8%E5%AE%89%E8%A3%85/K8S-Ubuntu18.04k%E7%A6%BB%E7%BA%BF%E5%AE%89%E8%A3%85.md) | [税局K8S安装](http://gitlab.hongmeng-info.com/wangjing/workmd/blob/master/%E7%A8%8E%E5%B1%80%E5%AE%89%E8%A3%85/%E5%B9%BF%E5%B7%9E%E5%B8%82%E7%A8%8E%E5%8A%A1%E5%B1%80k8s%E7%B3%BB%E7%BB%9F%E5%AE%89%E8%A3%85-02.md) |                                                              |

