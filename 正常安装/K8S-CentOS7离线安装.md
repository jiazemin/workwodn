# 离线安装k8s集群

## 一、准备环境

### 三台集群主机硬件配置：

| 主机名称 |    IP地址     | 内存 | 磁盘 | CPU个数 | CPU核数 | CPU线程数 |  备注   |
| :------: | :-----------: | :--: | :--: | :-----: | :-----: | :-------: | :-----: |
|  master  | 192.168.10.11 |  4G  | 60G  |    2    |    2    |     1     | centos7 |
|  node1   | 192.168.10.12 |  4G  | 60G  |    2    |    2    |     1     | centos7 |
|  node2   | 192.168.10.13 |  4G  | 60G  |    2    |    2    |     1     | centos7 |

### 三台集群主机相关操作：

```shell
# 设置环境变量
export HOST_K8S_MASTER1=192.168.10.11
export HOST_K8S_NODE1=192.168.10.12
export HOST_K8S_NODE1=192.168.10.13
```



## 二、准备文件

|      文件名称       |            ftp地址             |              备注               |
| :-----------------: | :----------------------------: | :-----------------------------: |
|     ansible安装     |     \\\192.168.10.9\upload     |  内含ansible和python2.7安装包   |
|       k8s安装       |     \\\192.168.10.9\upload     | 内含k8s集群和插件系统需要的文件 |
|        k8sCJ        | \\\192.168.10.9\upload\k8s安装 |     内含插件系统相关文件包      |
|   k8sCJJX.tar.gz    | \\\192.168.10.9\upload\k8s安装 |   内含插件系统相关docker镜像    |
| ansible_test.tar.gz | \\\192.168.10.9\upload\k8s安装 |        K8s集群安装文件包        |
|    anzhuang.yml     | \\\192.168.10.9\upload\k8s安装 |   插件系统镜像上传docker文件    |



## 三、开始安装

>参考文档: https://github.com/easzlab/kubeasz/blob/master/docs/setup/offline_install.md

#### 前奏准备：

- 以下所有操作都在root用户中进行，文件存放在/tmp目录，除了node主机安装python2.7外其他命令都在master主机中执行

- 在master主机中安装ansible（相关操作在ftp地址：\\\192.168.10.9\upload\ansible安装）

- 将k8s安装文件夹中的ansible_test.tar.gz、k8sCJJX.tar.gz、anzhuang.yaml文件和k8sCJ文件夹使用远程工具（winscp、finalShell等）上传至master主机的/tmp目录下

- 将master主机中已经安装好的/etc/ansible文件夹删除

```shell
rm -rf /etc/ansible
```

- 将上传到master主机中的ansible.tar文件解压到/etc目录中

```shell
cd /tmp && tar zxvf ansible.tar -C /etc
```

#### 开始安装：

- 使用加密算法给集群配置ssh免密登录

```shell
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519

# 补充：第二种加密算法
ssh-keygen -t rsa -b 2048 -N '' -f ~/.ssh/id_rsa
```

- 将ssh密钥拷贝到集群主机中（按提示输入yes和虚拟机密码）

```shell
ssh-copy-id 192.168.10.11
ssh-copy-id 192.168.10.12
ssh-copy-id 192.168.10.13
```

- 将master主机中/etc/ansible/example/hosts.multi-node文件复制为ansible的hosts文件

```shell
cd /etc/ansible && cp example/hosts.multi-node hosts
```

- 将复制好的ansible的hosts文件修改成如下配置（只修改了etcd、kube-master、kube-node、chrony）

```shell
[root@localhost ~]# vi hosts

# 'etcd' cluster should have odd member(s) (1,3,5,...)
# variable 'NODE_NAME' is the distinct name of a member in 'etcd' cluster
[etcd]
192.168.10.11 NODE_NAME=etcd1
192.168.10.12 NODE_NAME=etcd2
192.168.10.13 NODE_NAME=etcd3

# master node(s)
[kube-master]
192.168.10.11


# work node(s)
[kube-node]
192.168.10.12
192.168.10.13

# [optional] harbor server, a private docker registry
# 'NEW_INSTALL': 'yes' to install a harbor server; 'no' to integrate with existed one
[harbor]
#192.168.1.8 HARBOR_DOMAIN="harbor.yourdomain.com" NEW_INSTALL=no

# [optional] loadbalance for accessing k8s from outside
[ex-lb]
#192.168.1.6 LB_ROLE=backup EX_APISERVER_VIP=192.168.1.250 EX_APISERVER_PORT=8443
#192.168.1.7 LB_ROLE=master EX_APISERVER_VIP=192.168.1.250 EX_APISERVER_PORT=8443

# [optional] ntp server for the cluster
[chrony]
192.168.10.11

[all:vars]
# --------- Main Variables ---------------
# Cluster container-runtime supported: docker, containerd
CONTAINER_RUNTIME="docker"

# Network plugins supported: calico, flannel, kube-router, cilium, kube-ovn
CLUSTER_NETWORK="flannel"

# K8S Service CIDR, not overlap with node(host) networking
SERVICE_CIDR="10.68.0.0/16"

# Cluster CIDR (Pod CIDR), not overlap with node(host) networking
CLUSTER_CIDR="172.20.0.0/16"

# NodePort Range
NODE_PORT_RANGE="20000-40000"

# Cluster DNS Domain
CLUSTER_DNS_DOMAIN="cluster.local."

# -------- Additional Variables (don't change the default value right now) ---
# Binaries Directory
bin_dir="/opt/kube/bin"

# CA and other components cert/key Directory
ca_dir="/etc/kubernetes/ssl"

# Deploy Directory (kubeasz workspace)
base_dir="/etc/ansible"
```

- 验证ansible 安装 , 正常能看到节点返回 SUCCESS （期间会看到红色的告警： 组名中发现无效字符但未替换 。这并不会影响相关操作可以忽略！）

```shell
[root@localhost ~]# ansible all -m ping

192.168.10.11 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
192.168.10.12 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
192.168.10.13 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}

```

- 修改CA证书签名请求为以下配置


```shell
root@test-k8s-master1:~# vi /etc/ansible/roles/deploy/templates/ca-csr.json.j2

{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "GuangZhou",
      "L": "YX",
      "O": "hongmeng",
      "OU": "System"
    }
  ],
  "ca": {
    "expiry": "{{ CA_EXPIRY }}"
  }
}


```

- 修改admin证书签名请求为以下配置

```shell
[root@localhost ~]# vi /etc/ansible/roles/deploy/templates/admin-csr.json.j2

{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "GuangZhou",
      "L": "YX",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}

```

- 修改kube-proxy 证书请求为以下配置

```shell
[root@localhost ~]# vi /etc/ansible/roles/deploy/templates/kube-proxy-csr.json.j2

{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "GuangZhou",
      "L": "YX",
      "O": "hongmeng",
      "OU": "System"
    }
  ]
}

```

-  修改read证书请求为以下配置

```shell
[root@localhost ~]# vi /etc/ansible/roles/deploy/templates/read-csr.json.j2

{
  "CN": "read",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "GuangZhou",
      "L": "YX",
      "O": "group:read",
      "OU": "System"
    }
  ]
}

```

- 修改etcd证书请求为以下配置

```shell
[root@localhost ~]# vi /etc/ansible/roles/etcd/templates/etcd-csr.json.j2

{
  "CN": "etcd",
  "hosts": [
{% for host in groups['etcd'] %}
    "{{ host }}",
{% endfor %}
    "127.0.0.1"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "GuangZhou",
      "L": "YX",
      "O": "hongmeng",
      "OU": "System"
    }
  ]
}

```

- 修改kube-master中aggregator-proxy 证书签名请求为以下配置

```shell
[root@localhost ~]# vi /etc/ansible/roles/kube-master/templates/aggregator-proxy-csr.json.j2

{
  "CN": "aggregator",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "GuangZhou",
      "L": "YX",
      "O": "hongmeng",
      "OU": "System"
    }
  ]
}


```

- 修改kube-master中kubernetes 证书签名请求为以下配置

```shell
[root@localhost ~]# vi /etc/ansible/roles/kube-master/templates/kubernetes-csr.json.j2

{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
{% if groups['ex-lb']|length > 0 %}
    "{{ hostvars[groups['ex-lb'][0]]['EX_APISERVER_VIP'] }}",
{% endif %}
    "{{ inventory_hostname }}",
    "{{ CLUSTER_KUBERNETES_SVC_IP }}",
{% for host in MASTER_CERT_HOSTS %}
    "{{ host }}",
{% endfor %}
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "GuangZhou",
      "L": "YX",
      "O": "hongmeng",
      "OU": "System"
    }
  ]
}

```

- 修改kube-node中kubelet证书签名请求为以下配置

```shell
[root@localhost ~]# vi /etc/ansible/roles/kube-node/templates/kubelet-csr.json.j2

{
  "CN": "system:node:{{ inventory_hostname }}",
  "hosts": [
    "127.0.0.1",
    "{{ inventory_hostname }}"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "GuangZhou",
      "L": "YX",
      "O": "system:nodes",
      "OU": "System"
    }
  ]
}

```

- 修改calico证书签名请求为以下配置

```shell
[root@localhost ~]# vi /etc/ansible/roles/calico/templates/calico-csr.json.j2

{
  "CN": "calico",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "GuangZhou",
      "L": "YX",
      "O": "hongmeng",
      "OU": "System"
    }
  ]
}

```

- 进入/etc/ansible文件夹中，分步安装集群 

```shell
ansible-playbook 01.prepare.yml
ansible-playbook 02.etcd.yml
ansible-playbook 03.docker.yml
ansible-playbook 04.kube-master.yml
ansible-playbook 05.kube-node.yml
ansible-playbook 06.network.yml
ansible-playbook 07.cluster-addon.yml

# 补充：一步安装
ansible-playbook 90.setup.yml
```

- 查看集群安装情况


```shell
[root@localhost ~]# kubectl get node

NAME              STATUS                     ROLES    AGE     VERSION
192.168.10.11   Ready,SchedulingDisabled   master   5m55s   v1.15.0
192.168.10.12   Ready                      node     4m31s   v1.15.0
192.168.10.13   Ready                      node     4m31s   v1.15.0
```

- 查看集群所有的命名空间

```shell
[root@localhost ~]# kubectl get pod --all-namespaces
NAMESPACE     NAME                                          READY   STATUS    RESTARTS   AGE
kube-system   coredns-797455887b-5g987                      1/1     Running   0          2m21s
kube-system   coredns-797455887b-dgbbp                      1/1     Running   0          2m21s
kube-system   heapster-5f848f54bc-b8lw8                     1/1     Running   0          85s
kube-system   kube-flannel-ds-amd64-cghn8                   1/1     Running   0          2m57s
kube-system   kube-flannel-ds-amd64-qp9h9                   1/1     Running   0          2m57s
kube-system   kube-flannel-ds-amd64-t65cd                   1/1     Running   0          2m57s
kube-system   kubernetes-dashboard-5c7687cf8-h2x4k          1/1     Running   0          86s
kube-system   metrics-server-85c7b8c8c4-vlrzb               1/1     Running   0          2m8s
kube-system   traefik-ingress-controller-766dbfdddd-dql7w   1/1     Running   0          61s


```

- 修改master节点策略，允许容器在内运行。输入以下命令后再次查看集群情况发现master中SchedulingDisabled已经消失

```shell
kubectl patch node 192.168.10.16 -p '{"spec": {"unschedulable": false}}'
```

检查etcd集群状态，在任意 etcd 集群节点上执行如下命令

```shell
[root@localhost ~]# export NODE_IPS="192.168.10.11 192.168.10.12 192.168.10.13"
[root@localhost ~]# for ip in ${NODE_IPS}; do
  ETCDCTL_API=3 etcdctl \
  --endpoints=https://${ip}:2379  \
  --cacert=/etc/kubernetes/ssl/ca.pem \
  --cert=/etc/etcd/ssl/etcd.pem \
  --key=/etc/etcd/ssl/etcd-key.pem \
  endpoint health; done
 
  
https://192.168.10.11:2379 is healthy: successfully committed proposal: took = 2.246331ms
https://192.168.10.12:2379 is healthy: successfully committed proposal: took = 2.060241ms
https://192.168.10.13:2379 is healthy: successfully committed proposal: took = 1.512273ms

```

### 安装后使用tab键遇到的问题

```
_get_comp_words_by_ref: command not found
```

- 解决：

```shell
source /etc/bash_completion
source <(kubectl completion bash)
```



## k8s插件安装

- 使用ansible命令将插件压缩包分发到每个node节点主机的/tmp目录，并且解压

```shell
# 先解压再复制
tar zxvf /tmp/k8sCJJX.tar.gz
ansible kube-node -m unarchive -a "src=/tmp/k8sCJJX.tar.gz dest=/tmp/ copy=yes"
```

- 在master主机运行anzhuang.yml文件，可以将三台集群主机中导入的docker镜像包上传至docker

```shell
ansible-playbook /tmp/anzhuang.yml
```



## 插件dashboard 仪表盘 

**注：因为dashboard 在k8s集群安装好后就会自动创建好，所以只需要验证service服务后登录即可**

- dashboard 运行状态

```shell
[root@localhost ~]# kubectl get pod -n kube-system | grep dashboard

kubernetes-dashboard-5c7687cf8-rjmsx          1/1     Running   0          2m3s
```

- 查看dashboard service服务可以获得NodePort端口，用于登录


```shell
[root@localhost ~]# kubectl get svc -n kube-system|grep dashboard

kubernetes-dashboard      NodePort    10.68.196.47    <none>        443:26373/TCP    2m3s 
```

### dashboard 登录方式：令牌（token）

- 获取管理员身份的Token（该身份可以对集群进行管理），找到输出中 ‘token:’ 开头那一行

```shell
[root@localhost ~]# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')

token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWh4bjltIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI3N2FiMjQ1NS0xZjM4LTRjZTctYTQxNS04ZTQ3NzMyY2Y2MzYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.b-0Jksk_8us7p69gcXBT2nXUmptdRE0z_brUXQh8hccFToPAcGKhxX9aG9Su56AEa7bFcCwDtHwB9hbNP9RJTAGDARMNtB9ER0vpuSRblOlsoOwh3HKesW9b4Dd0otRtuEzuVnRqJ2kvIuNhJPdtyCx6HHu7yQ14AavbG5hSYzT4dFtKU_CUamC9XEh9W5aGTggTSCYQ0ivAzI_NVQZgfwItonwokCbW3ajYEr5_c7ADelNNQOc5pLNluULRiUfLZIQtx16vYsUeKUEcfKWuLlrkWfJhG2ukSTbljBvqifsAfllEv_zM-4TO-g56EnhzSYFCZisCXBelP2vJtJAgpg
```

获取只读用户身份的token（该身份只能查看集群中的信息），找到输出中 ‘token:’ 开头那一行

```shell
[root@localhost ~]# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep read-user | awk '{print $1}')

token:
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtcmVhZC11c2VyLXRva2VuLXFkOWJnIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImRhc2hib2FyZC1yZWFkLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJlNDI1OTQ0ZC05MDU2LTRjZTEtYjk4ZC1lYjlhN2ZhNWYxODgiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06ZGFzaGJvYXJkLXJlYWQtdXNlciJ9.OYQt4cB1BB4y7DlTo6mL-pKFB4parVpP9a1c_qPcANFwC0xMdH0Nqelp6LHylb41P9S_9Xao0RaFefBFdLYpw1eljEBpYEwc0jd5v6aDJHrfTrZy236ZeE1LzHeyw8ojesbjJSz5H-Ayisih8a54W6LfMkeo2-8a--I3JXpjruXQSDNFNZ2_N-WPCi0KpweVwMMaOqotgPngx9hvIirAYQv1grOm0avhkX6XahPN_Dmw_L5RP8raQ1wxME5G-uv6Yd52fJhPBUOGDuIwZZPnawJABNg9TI8I3TIAUT5lLK_fCq5iQFFxBtqd7mQyBKUvr7pLxTP8VvJcHSFNqSzK1A
```

- 使用火狐浏览器访问dashboard的web界面：https://$NodeIP:31845（因为k8s证书不被认同，所以这里推荐使用火狐浏览器；协议为https，端口和上方查看service服务中看到的NodePort端口26373一致）

```http
https://192.168.10.11:26373
```

- 在浏览器中选择-->高级-->接受风险并继续
- 看到Kubernetes 仪表板 ，选择令牌，然后将获取到的token复制到输入框登录即可

### dashboard 登录方式：配置文件（Kubeconfig）

由于配置文件方式登录的操作比较麻烦，这里不建议使用！感兴趣的小伙伴可以点击旁边链接阅读

[使用配置文件方式打开](https://github.com/easzlab/kubeasz/blob/master/docs/guide/dashboard.md)



## 插件helm k8s包管理工具

- 修改helm证书签名请求为以下配置

```shell
[root@localhost ~]# vi /etc/ansible/roles/helm/templates/helm-csr.json.j2

{
  "CN": "{{ helm_cert_cn }}",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "GuangZhou",
      "L": "YX",
      "O": "hongmeng",
      "OU": "System"
    }
  ]
}

```

- 修改helm客户端tiller证书签名请求为以下配置

```shell
[root@localhost ~]# vi /etc/ansible/roles/helm/templates/tiller-csr.json.j2

{
  "CN": "{{ tiller_cert_cn }}",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "GuangZhou",
      "L": "YX",
      "O": "hongmeng",
      "OU": "System"
    }
  ]
}

```



### 安装helm客户端

- 运行安装helm命令

```shell
ansible-playbook /etc/ansible/roles/helm/helm.yml
```

- 查看版本（只能看到客户端的版本）

```shell
$ helm version

Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
Error: could not find tiller
```



##### 安装helm服务端tiller

- 创建本地repo仓库


```shell
mkdir -p /opt/helm-repo
```

- 启动helm repo server，在后台挂载一个本地仓库


```shell
nohup helm serve --address 127.0.0.1:8879 --repo-path /opt/helm-repo &
```

- 更改helm 配置文件,将/etc/ansible/roles/helm/defaults/main.yml中repo的地址改为 127.0.0.1:8879


```sh
cat <<EOF >/etc/ansible/roles/helm/defaults/main.yml
helm_namespace: kube-system 
helm_cert_cn: helm001
tiller_sa: tiller
tiller_cert_cn: tiller001
tiller_image: easzlab/tiller:v2.14.1
#repo_url: https://kubernetes-charts.storage.googleapis.com
history_max: 5
repo_url: http://127.0.0.1:8879
# 如果默认官方repo 网络访问不稳定可以使用如下的阿里云镜像repo
#repo_url: https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
EOF
```

- 再次运行安装helm命令

```shell
ansible-playbook /etc/ansible/roles/helm/helm.yml
```

- 再次查看版本（可以看到完成版本）

```shell
$ helm version

Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
```



## 插件Prometheus（需要先安装好helm）

- 更新配置文件


```shell
source ~/.bashrc
```

- 进入prometheus文件夹内


```shell
cd /etc/ansible/manifests/prometheus
```

- 安装 prometheus 软件包

```shell
helm install \
        --name monitor \
        --namespace monitoring \
        -f prom-settings.yaml \
        -f prom-alertsmanager.yaml \
        -f prom-alertrules.yaml \
        prometheus
```

- 安装 grafana 软件包      

```shell
helm install \
	--name grafana \
	--namespace monitoring \
	-f grafana-settings.yaml \
	-f grafana-dashboards.yaml \
	grafana
```

- 验证安装


```shell
$ kubectl get pod,svc -n monitoring 

NAME                                                         READY   STATUS    RESTARTS   AGE
pod/grafana-78747c76-88pgw                                   1/1     Running   0          79s
pod/monitor-prometheus-alertmanager-59cc5c4b6-d6bgk          2/2     Running   0          106s
pod/monitor-prometheus-kube-state-metrics-7556696ff7-7pqbb   1/1     Running   0          105s
pod/monitor-prometheus-node-exporter-cxw9j                   1/1     Running   0          106s
pod/monitor-prometheus-node-exporter-dg9wq                   1/1     Running   0          106s
pod/monitor-prometheus-node-exporter-jpnqr                   1/1     Running   0          106s
pod/monitor-prometheus-server-579954d46d-2pchk               2/2     Running   0          104s

NAME                                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/grafana                                 NodePort    10.68.26.181    <none>        80:39002/TCP   80s
service/monitor-prometheus-alertmanager         NodePort    10.68.130.178   <none>        80:39001/TCP   107s
service/monitor-prometheus-kube-state-metrics   ClusterIP   None            <none>        80/TCP         107s
service/monitor-prometheus-node-exporter        ClusterIP   None            <none>        9100/TCP       107s
service/monitor-prometheus-server               NodePort    10.68.193.62    <none>        80:39000/TCP   107s

```

- 访问prometheus的web界面：http://$NodeIP:39000

```http
https://192.168.10.11:39000
```

- 访问alertmanager的web界面：http://$NodeIP:39001

```http
https://192.168.10.11:39001
```

- 访问grafana的web界面：http://$NodeIP:39002 (默认用户密码 admin:admin，可在web界面修改)

```http
https://192.168.10.11:39002
```



## 插件EFK安装

- 运行EFK安装文件


```shell
# 安装
kubectl apply -f /etc/ansible/manifests/efk/
kubectl apply -f /etc/ansible/manifests/efk/es-without-pv/

# 删除
kubectl delete -f /etc/ansible/manifests/efk/
kubectl delete -f /etc/ansible/manifests/efk/es-without-pv/
```

- 验证EFK安装


```shell
$ kubectl get pods -n kube-system|grep -E 'elasticsearch|fluentd|kibana'

elasticsearch-logging-0                       1/1     Running   0          33s
elasticsearch-logging-1                       1/1     Running   0          23s
fluentd-es-v2.4.0-6tqbq                       1/1     Running   0          40s
fluentd-es-v2.4.0-qqgmp                       1/1     Running   0          40s
fluentd-es-v2.4.0-vhq4p                       1/1     Running   0          40s
kibana-logging-ffd866674-kcz98                1/1     Running   0          39s
```

- 开启 apiserver basic-auth(用户名/密码认证)


```shell
# 给文件写的权限
chmod +x /usr/bin/easzctl
# 开启认证
easzctl basic-auth -s -u hongmeng -p it168
```

- 获取kibana运行的地址

```shell
$ kubectl cluster-info | grep Kibana

Kibana is running at https://192.168.10.11:6443/api/v1/namespaces/kube-system/services/kibana-logging/proxy
```

- 浏览器访问kibana地址(默认用户密码为上方认证的账号密码:hongmeng ; it168)


```http
https://192.168.100.101:6443/api/v1/namespaces/kube-system/services/kibana-logging/proxy
```

- 创建kibana实例


1. 选择Management-->Index Patterns
2. 在index pattern输入框中输入默认的pattern：logstash-*	-->Next step
3. 在Time Filter field name下拉框中选择@timestamp -->Create index pattern
4. 日志索引已经完成，可以在界面中看到



## 可选k8s项目测试操作

- 上传私有仓库的镜像文件


```shell
docker load -i registry.tar 
```

- 启动私有仓库容器 


```shell
docker run -di --name=registry -p 5000:5000 registry 
```

- 打 开 浏 览 器 输 入 地 址 **http://192.168.10.11:5000/v2/_catalog** 看 到 


**{"repositories":[]}** 表示私有仓库搭建成功并且内容为空 。**IP地址是自己虚拟机的地址**

- 修改 /etc/docker/daemon.json


添加以下内容，保存退出。 此步用于让 docker 信任私有仓库地址 

```json
{
	"insecure-registries":["192.168.10.11:5000"]
} 
```

- 重启 docker 服务


```shell
systemctl restart docker
```

- 先把测试项目文件夹test放入虚拟机中，然后打开终端输入命令构建docker镜像


```shell
docker build -t testdemo .
```

- 标记此镜像为私有仓库的镜像 


```shell
docker tag testdemo 192.168.10.11:5000/testdemo
```

- 再次启动私服容器 


```shell
docker start registry 
```

- 上传标记的镜像 ,再次刷新浏览器地址http://192.168.10.11:5000/v2/_catalog可以看到新增加的镜像


```shell
docker push 192.168.10.11:5000/testdemo
```

- 将准备好的两个yaml进行运行,只需要修改name和image这两个选项（注意要空格）


```yaml
name: hello
image: 192.168.10.11:5000/testdemo
```

- 根据两个yaml创建运行服务


```shell
kubectl create -f hello-rc.yaml
kubectl create -f hello-svc.yaml
```

- 查看kubectel中的服务


```shell
$ kubectl get svc
hello	NodePort	10.68.160.46	<none>	8080:30003/TCP
```

- 访问项目地址:


```http
http://192.168.10.11:30003/hello

hello docker
```





