# 离线安装k8s集群

## 一、准备环境

### 三台集群主机硬件配置：

| 主机名称 |     IP地址      | 内存大小 | 磁盘大小 | CPU个数 | CPU核数 | CPU线程数 | 磁盘数量 |    备注     |
| :------: | :-------------: | :------: | :------: | :-----: | :-----: | :-------: | :------: | :---------: |
|  master  | 192.168.100.101 |    4G    | 60G+20G  |    2    |    2    |     1     |    2     | ubuntu18.04 |
|  node1   | 192.168.100.102 |    4G    | 60G+20G  |    2    |    2    |     1     |    2     | ubuntu18.04 |
|  node2   | 192.168.100.103 |    4G    | 60G+20G  |    2    |    2    |     1     |    2     | ubuntu18.04 |

### 三台集群主机相关操作：

```shell
# 在当前用户hongmeng设置root密码
root@test-k8s-master1:~# sudo passwd root
[sudo] password for hongmeng: it168
Enter new UNIX password: it168
Retype new UNIX password: it168
passwd: password updated successfully

# 进入root用户
root@test-k8s-master1:~# su -
Password: it168
root@test-k8s-master1:/home/hongmeng#

# 允许ssh远程登录
root@test-k8s-master1:~# vi /etc/ssh/sshd_config
# 在PermitRootLogin prohibit-password这行下面添加
PermitRootLogin yes
# 重启ssh服务
/etc/init.d/ssh restart

# 设置环境变量(后期使用，目前暂不需要)
export HOST_K8S_MASTER1=192.168.100.101
export HOST_K8S_NODE1=192.168.100.102
export HOST_K8S_NODE1=192.168.100.103
```



## 二、准备文件

|           文件名称            |                    ftp地址                    |              备注               |
| :---------------------------: | :-------------------------------------------: | :-----------------------------: |
|          ansible安装          |    ftp://192.168.10.9/软件/06 Kubernetes/     |  内含ansible和python2.7安装包   |
|            k8s安装            |    ftp://192.168.10.9/软件/06 Kubernetes/     | 内含k8s集群和监控系统需要的文件 |
|        k8s-plug.tar.gz        | ftp://192.168.10.9/软件/06 Kubernetes/k8s安装 |     内含插件系统相关文件包      |
| k8s-plug-docker-images.tar.gz | ftp://192.168.10.9/软件/06 Kubernetes/k8s安装 |   内含插件系统相关docker镜像    |
|  k8s-ansible-install.tar.gz   | ftp://192.168.10.9/软件/06 Kubernetes/k8s安装 |        K8s集群安装文件包        |
|   upload-docker-images.yml    | ftp://192.168.10.9/软件/06 Kubernetes/k8s安装 |     上传docker镜像操作文件      |



## 三、k8s集群安装

>参考文档: https://github.com/easzlab/kubeasz/blob/master/docs/setup/offline_install.md

#### 安装准备：

- 以下所有操作都在root用户中进行，文件全部存放在/tmp目录，除了node主机安装python2.7外其他命令都在master主机中执行
- 在master主机中安装ansible
- 在所有node主机中安装python2.7
- 将k8s安装文件夹中的所有文件取出后上传至master主机的/tmp目录下
- 将master主机中已经安装好的/etc/ansible文件夹删除

```shell
rm -rf /etc/ansible
```

- 将上传到master主机中的k8s-ansible-install.tar.gz文件解压到/etc目录中

```shell
tar zxvf /tmp/k8s2.0.2-ansible-install.tar.gz -C /etc
```

#### 开始安装：

- 使用加密算法给集群配置ssh免密登录

```shell
# 默认使用第一种加密算法
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519

# 补充：第二种加密算法
ssh-keygen -t rsa -b 2048 -N '' -f ~/.ssh/id_rsa
```

- 将ssh密钥拷贝到集群主机中（按提示输入yes和虚拟机root账号的密码）

```shell
ssh-copy-id 192.168.100.101
ssh-copy-id 192.168.100.102
ssh-copy-id 192.168.100.103
```

- 将master主机中/etc/ansible/example/hosts.multi-node文件复制为ansible的hosts文件

```shell
cd /etc/ansible && cp example/hosts.multi-node hosts
```

- 将复制好的ansible的hosts文件修改成如下配置（只修改了etcd、kube-master、kube-node、chrony）

```shell
root@test-k8s-master1:~# vi hosts

# 'etcd' cluster should have odd member(s) (1,3,5,...)
# variable 'NODE_NAME' is the distinct name of a member in 'etcd' cluster
[etcd]
192.168.100.101 NODE_NAME=etcd1
192.168.100.102 NODE_NAME=etcd2
192.168.100.103 NODE_NAME=etcd3

# master node(s)
[kube-master]
192.168.100.101


# work node(s)
[kube-node]
192.168.100.102
192.168.100.103

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
192.168.100.101

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

- 验证ansible 安装 , 正常能看到节点返回 SUCCESS （如果出现：discovered_interpreter_python": "/usr/bin/python，说明你的node节点没有安装python）

```shell
root@test-k8s-master1:~# ansible all -m ping

192.168.100.102 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
192.168.100.103 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
192.168.100.101 | SUCCESS => {
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
      "O": "k8s",
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
root@test-k8s-master1:~# vi /etc/ansible/roles/deploy/templates/admin-csr.json.j2

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
root@test-k8s-master1:~# vi /etc/ansible/roles/deploy/templates/kube-proxy-csr.json.j2

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
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

-  修改read证书请求为以下配置

```shell
root@test-k8s-master1:~# vi /etc/ansible/roles/deploy/templates/read-csr.json.j2

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
root@test-k8s-master1:~# vi /etc/ansible/roles/etcd/templates/etcd-csr.json.j2

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
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

- 修改kube-master中aggregator-proxy 证书签名请求为以下配置

```shell
root@test-k8s-master1:~# vi /etc/ansible/roles/kube-master/templates/aggregator-proxy-csr.json.j2

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
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

- 修改kube-master中kubernetes 证书签名请求为以下配置

```shell
root@test-k8s-master1:~# vi /etc/ansible/roles/kube-master/templates/kubernetes-csr.json.j2

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
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

- 修改kube-node中kubelet证书签名请求为以下配置

```shell
root@test-k8s-master1:~# vi /etc/ansible/roles/kube-node/templates/kubelet-csr.json.j2

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

- 修改calico网络证书签名请求为以下配置（使用90.setup.yml文件安装会装这个网络）


```shell
root@test-k8s-master1:~# vi /etc/ansible/roles/calico/templates/calico-csr.json.j2

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
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

- 设置参数，启用离线安装

```shell
sed -i 's/^INSTALL_SOURCE.*$/INSTALL_SOURCE: "offline"/g' /etc/ansible/roles/chrony/defaults/main.yml

sed -i 's/^INSTALL_SOURCE.*$/INSTALL_SOURCE: "offline"/g' /etc/ansible/roles/ex-lb/defaults/main.yml

sed -i 's/^INSTALL_SOURCE.*$/INSTALL_SOURCE: "offline"/g' /etc/ansible/roles/kube-node/defaults/main.yml

sed -i 's/^INSTALL_SOURCE.*$/INSTALL_SOURCE: "offline"/g' /etc/ansible/roles/prepare/defaults/main.yml

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
root@test-k8s-master1:~# kubectl get node

NAME            STATUS                     ROLES    AGE     VERSION
192.168.100.101   Ready,SchedulingDisabled   master   5m1s    v1.15.0
192.168.100.102   Ready                      node     3m34s   v1.15.0
192.168.100.103   Ready                      node     3m41s   v1.15.0

```

- 查看集群所有的命名空间

```shell
root@test-k8s-master1:~# kubectl get pod --all-namespaces
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
kubectl patch node 192.168.100.101 -p '{"spec": {"unschedulable": false}}'
```

- 检查etcd集群状态，在任意 etcd 集群节点上执行如下命令 

```bash
root@test-k8s-master1:~# export NODE_IPS="192.168.100.101 192.168.100.102 192.168.100.103"
root@test-k8s-master1:~# for ip in ${NODE_IPS}; do
  ETCDCTL_API=3 etcdctl \
  --endpoints=https://${ip}:2379  \
  --cacert=/etc/kubernetes/ssl/ca.pem \
  --cert=/etc/etcd/ssl/etcd.pem \
  --key=/etc/etcd/ssl/etcd-key.pem \
  endpoint health; done
 
  
https://192.168.100.101:2379 is healthy: successfully committed proposal: took = 2.111512ms
https://192.168.100.102:2379 is healthy: successfully committed proposal: took = 4.530936ms
https://192.168.100.103:2379 is healthy: successfully committed proposal: took = 1.879708ms


```

- 开启 apiserver basic-auth(用户名/密码认证)，后面EFK插件会使用到

```shell
# 给文件写的权限
chmod +x /usr/bin/easzctl
# 开启认证并设置账号密码
easzctl basic-auth -s -u hongmeng -p it168
```





### 插件dashboard 仪表盘

**注：dashboard 在k8s集群安装好后就会自动创建好，所以只需要验证service服务后登录即可**

- 查看dashboard 运行状态

```shell
root@test-k8s-master1:~# kubectl get pod -n kube-system | grep dashboard

kubernetes-dashboard-5c7687cf8-rjmsx          1/1     Running   0          2m3s

```

- 查看dashboard service服务可以获得NodePort端口，用于登录


```shell
root@test-k8s-master1:~# kubectl get svc -n kube-system|grep dashboard

kubernetes-dashboard      NodePort    10.68.196.47    <none>        443:31845/TCP    2m3s 

```

### dashboard 登录方式：令牌（token）

- 获取管理员身份的Token（该身份可以对集群进行管理），找到输出中 ‘token:’ 开头那一行

```shell
root@test-k8s-master1:~# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')

token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXQ3NHRiIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI5YzNjZGQ3ZC1iNzQyLTQ4ZjUtOTBjMS04OTJkZWFhZjg5ZTUiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.KbFikixNWay5yPTioWbRi-mD-WP85QR31M5hm00LC3Eule_5iWHEQc_ENgLebt7DrC1QJUSYc9YjDfsPSTWOGij_DGf4LCc9G5aEInlrRYbkiconbtEfkHATOZZ7zNtgwejkbEQ-T7mnHEZCHzNK67zs33-P7cM21f2L798fPdZjvqfmbmHnitjcfszP3V3Ff-3UAWb0SYbgYvdNEW8X6Vv9tXn8IVK881PGYQc5EfFgMjOXv6QYjo0LZ2KtKRIKPhJ7Zes8Z2IrU0H624oENmwbqsE937mjdySOSQHM_BYEPCU3sHgzeReYXOhgIceptBImaO6iddRNHGNErpmM2Q

```

- 获取只读用户身份的token（该身份只能查看集群中的信息），找到输出中 ‘token:’ 开头那一行

```shell
root@test-k8s-master1:~# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep read-user | awk '{print $1}')

token:
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtcmVhZC11c2VyLXRva2VuLXFkOWJnIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImRhc2hib2FyZC1yZWFkLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJlNDI1OTQ0ZC05MDU2LTRjZTEtYjk4ZC1lYjlhN2ZhNWYxODgiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06ZGFzaGJvYXJkLXJlYWQtdXNlciJ9.OYQt4cB1BB4y7DlTo6mL-pKFB4parVpP9a1c_qPcANFwC0xMdH0Nqelp6LHylb41P9S_9Xao0RaFefBFdLYpw1eljEBpYEwc0jd5v6aDJHrfTrZy236ZeE1LzHeyw8ojesbjJSz5H-Ayisih8a54W6LfMkeo2-8a--I3JXpjruXQSDNFNZ2_N-WPCi0KpweVwMMaOqotgPngx9hvIirAYQv1grOm0avhkX6XahPN_Dmw_L5RP8raQ1wxME5G-uv6Yd52fJhPBUOGDuIwZZPnawJABNg9TI8I3TIAUT5lLK_fCq5iQFFxBtqd7mQyBKUvr7pLxTP8VvJcHSFNqSzK1A

```

- 使用火狐浏览器访问dashboard的web界面：https://$NodeIP:31845（因为k8s证书不被认同，所以这里推荐使用火狐浏览器；协议为https，端口和上方查看service服务中看到的NodePort端口31845一致）

```http
https://192.168.100.101:31845

```

- 在浏览器中选择-->高级-->接受风险并继续
- 看到Kubernetes 仪表板 ，选择令牌，然后将获取到的不同身份的token复制到输入框登录即可

### dashboard 登录方式：配置文件（Kubeconfig）

由于配置文件方式登录的操作比较麻烦，这里不建议使用！感兴趣的小伙伴可以点击旁边链接阅读[dashboard使用配置文件方式打开](https://github.com/easzlab/kubeasz/blob/master/docs/guide/dashboard.md)



## k8s其他插件镜像上传

- 使用ansible命令将插件压缩包分发到每个node节点主机的/tmp目录，并且解压

```shell
# 先解压再复制
tar zxvf /tmp/k8s-plug.tar.gz
tar zxvf /tmp/k8s-plug-docker-images.tar.gz
ansible kube-node -m unarchive -a "src=/tmp/k8s-plug-docker-images.tar.gz dest=/tmp/ copy=yes"
```

- 在master主机运行anzhuang.yml文件，可以将三台集群主机中导入的docker镜像包上传至docker

```shell
ansible-playbook /tmp/upload-docker-images.yml
```



## 插件helm k8s包管理工具

- 修改helm证书签名请求为以下配置

```shell
root@test-k8s-master1:~# vi /etc/ansible/roles/helm/templates/helm-csr.json.j2

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
root@test-k8s-master1:~# vi /etc/ansible/roles/helm/templates/tiller-csr.json.j2

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

- 解压将上传好的helm-v2.14.1-linux-amd64.tar.gz压缩包

```shell
tar zxvf /tmp/k8s-plug/helm-v2.14.1-linux-amd64.tar.gz -C /home/hongmeng/
```

- 将解压后得到的helm文件移动到所需的位置

```shell
mv /home/hongmeng/linux-amd64/helm /usr/local/bin/helm

```

- 查看版本（只能看到客户端的版本）

```shell
root@test-k8s-master1:~# helm version

Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
Error: could not find tiller

```



### 安装helm服务端tiller

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

- 运行安装helm命令

```shell
ansible-playbook /etc/ansible/roles/helm/helm.yml

```

- 再次查看版本（可以看到完成版本）

```shell
root@test-k8s-master1:~# helm version

Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}

```



## 插件Prometheus时序数据库（需要先安装好helm）

- 进入prometheus文件夹内


```shell
cd /etc/ansible/manifests/prometheus

```

- 安装 prometheus 软件包

```shell
helm install --tls \
        --name monitor \
        --namespace monitoring \
        -f prom-settings.yaml \
        -f prom-alertsmanager.yaml \
        -f prom-alertrules.yaml \
        prometheus

```

- 安装 grafana 软件包      

```shell
helm install --tls \
	--name grafana \
	--namespace monitoring \
	-f grafana-settings.yaml \
	-f grafana-dashboards.yaml \
	grafana

```

- 查看安装情况


```shell
root@test-k8s-master1:~# kubectl get pod,svc -n monitoring

NAME                                                         READY   STATUS    RESTARTS   AGE
pod/grafana-78747c76-cs92l                                   1/1     Running   0          2m46s
pod/monitor-prometheus-alertmanager-59cc5c4b6-6hfvc          2/2     Running   0          2m53s
pod/monitor-prometheus-kube-state-metrics-7556696ff7-pzw59   1/1     Running   0          2m53s
pod/monitor-prometheus-node-exporter-ttv4n                   1/1     Running   0          7s
pod/monitor-prometheus-node-exporter-w52mv                   1/1     Running   0          2m53s
pod/monitor-prometheus-node-exporter-xj9dl                   1/1     Running   0          2m53s
pod/monitor-prometheus-server-579954d46d-xwvx2               2/2     Running   0          2m53s

NAME                                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/grafana                                 NodePort    10.68.222.143   <none>        80:39002/TCP   2m46s
service/monitor-prometheus-alertmanager         NodePort    10.68.218.89    <none>        80:39001/TCP   2m53s
service/monitor-prometheus-kube-state-metrics   ClusterIP   None            <none>        80/TCP         2m53s
service/monitor-prometheus-node-exporter        ClusterIP   None            <none>        9100/TCP       2m53s
service/monitor-prometheus-server               NodePort    10.68.143.230   <none>        80:39000/TCP   2m53s



```

- 访问prometheus的web界面：http://$NodeIP:39000

```http
http://192.168.100.101:39000

```

- 访问alertmanager的web界面：http://$NodeIP:39001

```http
http://192.168.100.101:39001

```

- 访问grafana的web界面：http://$NodeIP:39002 (默认用户密码 admin:admin，可在web界面修改)

```http
http://192.168.100.101:39002

```



## 插件Ambassador网关（需要先安装好helm）

- 将上传好的ambassador.tar.gz包解压

```shell
tar zxvf /tmp/k8s-plug/ambassador.tar.gz -C /home/hongmeng

```

- 进入/home/hongmeng目录执行安装命令

```shell
cd /home/hongmeng
helm install --tls --name ambassador --set service.type=NodePort ambassador

```

- 查看ambassador的service状态

```shell
root@hongmeng:~# kubectl get svc
NAME               TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ambassador         NodePort    10.68.42.114   <none>        80:38178/TCP,443:37391/TCP   4m10s
ambassador-admin   ClusterIP   10.68.87.22    <none>        8877/TCP                     4m10s
kubernetes         ClusterIP   10.68.0.1      <none>        443/TCP                      10h

```

- 浏览器访问地址

```http
http://192.168.100.101:38178/ambassador/v0/diag/

```



#### 装stable/ambassador时遇到的问题

```shell
Error: customresourcedefinitions.apiextensions.k8s.io "consulresolvers.getambassador.io" already exists

```

- 解决：执行命令删除相关的crds即可:

```shell
# 查看已有的crds
root@test-k8s-master1:~# kubectl get crds|grep ambassador

authservices.getambassador.io                  2019-12-11T09:45:35Z
consulresolvers.getambassador.io               2019-12-11T09:45:37Z
filterpolicies.getambassador.io                2019-12-11T09:45:38Z
filters.getambassador.io                       2019-12-11T09:45:39Z
kubernetesendpointresolvers.getambassador.io   2019-12-11T09:45:40Z
kubernetesserviceresolvers.getambassador.io    2019-12-11T09:45:41Z
mappings.getambassador.io                      2019-12-11T09:45:42Z
modules.getambassador.io                       2019-12-11T09:45:43Z
ratelimits.getambassador.io                    2019-12-11T09:45:44Z
ratelimitservices.getambassador.io             2019-12-11T09:45:45Z
tcpmappings.getambassador.io                   2019-12-11T09:45:46Z
tlscontexts.getambassador.io                   2019-12-11T09:45:47Z
tracingservices.getambassador.io               2019-12-11T09:45:48Z


# 删除相关的crds
kubectl get crds|grep ambassador | awk '{ print $1; }' | xargs -I {}  kubectl delete crds {}

```



## 插件metallb 软负载均衡器

- 将上传好的metallb.tar.gz包解压

```shell
tar zxvf /tmp/k8s-plug/metallb_test.tar.gz -C /home/hongmeng

```

- 修改ip池配置文件/metallb/values.yaml，将configInline:{}改为以下配置

```shell
root@test-k8s-master1:~# vi /home/hongmeng/metallb/values.yaml

configInline:
  address-pools:
  - name: default
    protocol: layer2
    addresses: 
    # 这里的ip与节点ip相同
    - 192.168.100.101-192.168.100.103

```

- 进入/home/hongmeng目录执行安装命令

```shell
cd /home/hongmeng
helm install --tls --name metallb --namespace kube-system metallb

```

- 查看metallb创建的状态

```shell
root@hongmeng:~# kubectl get pod -n kube-system

NAME                                          READY   STATUS    RESTARTS   AGE
metallb-controller-b7c58df4f-x29dc            1/1     Running   0          76s
metallb-speaker-hqp95                         1/1     Running   0          76s
metallb-speaker-lktf8                         1/1     Running   0          76s
metallb-speaker-zqzrg                         1/1     Running   0          76s

```

- （可选）配置一个测试案例：metallb-nginx.yaml（已放入/tmp/k8s-plug）

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
  labels:
    role: web
    app: nginx
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
        role: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
---
kind: Service
apiVersion: v1
metadata:
  name: my-nginx-service
  annotations:
    getambassador.io/config: |
      ---
      apiVersion: ambassador/v1
      kind: Mapping
      name: nginx_mapping
      host: nginx.kestrel-info.com
      service: my-nginx-service:80
      prefix: /
spec:
  selector:
    app: nginx
    role: web
  ports:
  - protocol: TCP
    port: 80

```

- 启动测试案例

```shell
cd /tmp/k8s-plug
kubectl apply -f metallb-nginx.yaml

```

- 查看nginx的service服务

```shell
root@test-k8s-master1:~# kubectl get svc

NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
kubernetes                  ClusterIP   10.68.0.1       <none>        443/TCP                      29h
my-nginx-service            ClusterIP   10.68.143.131   <none>        80/TCP                       47m
ugly-bat-ambassador         NodePort    10.68.83.217    <none>        80:20894/TCP,443:36023/TCP   116m
ugly-bat-ambassador-admin   ClusterIP   10.68.171.160   <none>        8877/TCP                     116m


```

- 访问nginx

```shell
root@test-k8s-master1:~# curl 10.68.143.131

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>


```



## 插件CEPH安装

- 将上传好的rook文件夹解压

```shell
tar zxvf /tmp/k8s-plug/rook-1.1.4_test.tar.gz -C /home/hongmeng

```

- 进入rook文件夹中进行部署

```shell
cd /home/hongmeng/rook-1.1.4/cluster/examples/kubernetes/ceph

kubectl apply -f common.yaml

kubectl apply -f operator.yaml

```

- 查看operator状态

```shell
root@hongmeng:~# kubectl -n rook-ceph get pod -o wide

rook-ceph-tools-cfb9d595b-sbp2w                1/1     Running     2          26h   192.168.100.101   192.168.100.101   <none>           <none>
rook-discover-28vvn                            1/1     Running     1          26h   172.20.2.21       192.168.100.103   <none>           <none>
rook-discover-kkw94                            1/1     Running     1          26h   172.20.0.21       192.168.100.101   <none>           <none>
rook-discover-x7pd4                            1/1     Running     1          26h   172.20.1.29       192.168.100.102   <none>           <none>


```

- 给ceph-mon节点打上标签

```shell
kubectl label nodes {192.168.100.101,192.168.100.102,192.168.100.103} ceph-mon=enabled

```

- 给ceph-osd节点打上标签

```shell
kubectl label nodes {192.168.100.101,192.168.100.102,192.168.100.103} ceph-osd=enabled

```

- 给ceph-mgr节点打上标签，mgr只能支持一个节点运行，这是ceph跑k8s里的局限

```shell
kubectl label nodes 192.168.100.101 ceph-mgr=enabled

```

- 配置cluster.yaml文件

**务必注意：**

dataDirHostPath: 这个路径是会在宿主机上生成的，保存的是ceph的相关的配置文件，再重新生成集群的时候要确保这个目录为空，否则mon会无法启动  
useAllDevices: 使用所有的设备，建议为false，否则会把宿主机所有可用的磁盘都干掉  
useAllNodes：使用所有的node节点，建议为false，肯定不会用k8s集群内的所有node来搭建ceph的  
databaseSizeMB和journalSizeMB：当磁盘大于100G的时候，就注释这俩项就行了 

```yaml
#################################################################################################################
# Define the settings for the rook-ceph cluster with common settings for a production cluster.
# All nodes with available raw devices will be used for the Ceph cluster. At least three nodes are required
# in this example. See the documentation for more details on storage settings available.

# For example, to create the cluster:
#   kubectl create -f common.yaml
#   kubectl create -f operator.yaml
#   kubectl create -f cluster.yaml
#################################################################################################################

apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    # The container image used to launch the Ceph daemon pods (mon, mgr, osd, mds, rgw).
    # v13 is mimic, v14 is nautilus, and v15 is octopus.
    # RECOMMENDATION: In production, use a specific version tag instead of the general v14 flag, which pulls the latest release and could result in different
    # versions running within the cluster. See tags available at https://hub.docker.com/r/ceph/ceph/tags/.
    image: ceph/ceph:v14.2.4-20190917
    # Whether to allow unsupported versions of Ceph. Currently mimic and nautilus are supported, with the recommendation to upgrade to nautilus.
    # Octopus is the version allowed when this is set to true.
    # Do not set to true in production.
    allowUnsupported: false
  # The path on the host where configuration files will be persisted. Must be specified.
  # Important: if you reinstall the cluster, make sure you delete this directory from each host or else the mons will fail to start on the new cluster.
  # In Minikube, the '/data' directory is configured to persist across reboots. Use "/data/rook" in Minikube environment.
  dataDirHostPath: /var/lib/rook
  # Whether or not upgrade should continue even if a check fails
  # This means Ceph's status could be degraded and we don't recommend upgrading but you might decide otherwise
  # Use at your OWN risk
  # To understand Rook's upgrade process of Ceph, read https://rook.io/docs/rook/master/ceph-upgrade.html#ceph-version-upgrades
  skipUpgradeChecks: false
  # set the amount of mons to be started
  mon:
    count: 3
    allowMultiplePerNode: false
  # mgr:
    # modules:
    # Several modules should not need to be included in this list. The "dashboard" and "monitoring" modules
    # are already enabled by other settings in the cluster CR and the "rook" module is always enabled.
    # - name: pg_autoscaler
    #   enabled: true
  # enable the ceph dashboard for viewing cluster status
  dashboard:
    enabled: true
    # serve the dashboard under a subpath (useful when you are accessing the dashboard via a reverse proxy)
    # urlPrefix: /ceph-dashboard
    # serve the dashboard at the given port.
    # port: 8443
    # serve the dashboard using SSL
    ssl: true
  # enable prometheus alerting for cluster
  monitoring:
    # requires Prometheus to be pre-installed
    enabled: false
    # namespace to deploy prometheusRule in. If empty, namespace of the cluster will be used.
    # Recommended:
    # If you have a single rook-ceph cluster, set the rulesNamespace to the same namespace as the cluster or keep it empty.
    # If you have multiple rook-ceph clusters in the same k8s cluster, choose the same namespace (ideally, namespace with prometheus
    # deployed) to set rulesNamespace for all the clusters. Otherwise, you will get duplicate alerts with multiple alert definitions.
    rulesNamespace: rook-ceph
  network:
    # toggle to use hostNetwork
    hostNetwork: false
  rbdMirroring:
    # The number of daemons that will perform the rbd mirroring.
    # rbd mirroring must be configured with "rbd mirror" from the rook toolbox.
    workers: 0
  # To control where various services will be scheduled by kubernetes, use the placement configuration sections below.
  # The example under 'all' would have all services scheduled on kubernetes nodes labeled with 'role=storage-node' and
  # tolerate taints with a key of 'storage-node'.
  placement:
#    all:
#      nodeAffinity:
#        requiredDuringSchedulingIgnoredDuringExecution:
#          nodeSelectorTerms:
#          - matchExpressions:
#            - key: role
#              operator: In
#              values:
#              - storage-node
#      podAffinity:
#      podAntiAffinity:
#      tolerations:
#      - key: storage-node
#        operator: Exists
# The above placement information can also be specified for mon, osd, and mgr components
#    mon:
# Monitor deployments may contain an anti-affinity rule for avoiding monitor
# collocation on the same node. This is a required rule when host network is used
# or when AllowMultiplePerNode is false. Otherwise this anti-affinity rule is a
# preferred rule with weight: 50.
#    osd:
#    mgr:
    mon:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: ceph-mon
              operator: In
              values:
              - enabled
    ods:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: ceph-osd
              operator: In
              values:
              - enabled

    mgr:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: ceph-mgr
              operator: In
              values:
              - enabled
  annotations:
#    all:
#    mon:
#    osd:
# If no mgr annotations are set, prometheus scrape annotations will be set by default.
#   mgr:
  resources:
# The requests and limits set here, allow the mgr pod to use half of one CPU core and 1 gigabyte of memory
#    mgr:
#      limits:
#        cpu: "500m"
#        memory: "1024Mi"
#      requests:
#        cpu: "500m"
#        memory: "1024Mi"
# The above example requests/limits can also be added to the mon and osd components
#    mon:
#    osd:
#    prepareosd:
  storage: # cluster level storage configuration and selection
    useAllNodes: false
    useAllDevices: false
    deviceFilter:
    location:
    config:
      # The default and recommended storeType is dynamically set to bluestore for devices and filestore for directories.
      # Set the storeType explicitly only if it is required not to use the default.
      # storeType: bluestore
      # metadataDevice: "md0" # specify a non-rotational storage so ceph-volume will use it as block db device of bluestore.
      # databaseSizeMB: "1024" # uncomment if the disks are smaller than 100 GB
      # journalSizeMB: "1024"  # uncomment if the disks are 20 GB or smaller
      # osdsPerDevice: "1" # this value can be overridden at the node or device level
      # encryptedDevice: "true" # the default value for this option is "false"
# Cluster level list of directories to use for filestore-based OSD storage. If uncomment, this example would create an OSD under the dataDirHostPath.
    #directories:
    #- path: /var/lib/rook
# Individual nodes and their config can be specified as well, but 'useAllNodes' above must be set to false. Then, only the named
# nodes below will be used as storage resources.  Each node's 'name' field should match their 'kubernetes.io/hostname' label.
    nodes:
#    - name: "172.17.4.101"
#      directories: # specific directories to use for storage can be specified for each node
#      - path: "/rook/storage-dir"
#      resources:
#        limits:
#          cpu: "500m"
#          memory: "1024Mi"
#        requests:
#          cpu: "500m"
#          memory: "1024Mi"
#    - name: "172.17.4.201"
#      devices: # specific devices to use for storage can be specified for each node
#      - name: "sdb"
#      - name: "nvme01" # multiple osds can be created on high performance devices
#        config:
#          osdsPerDevice: "5"
#      config: # configuration can be specified at the node level which overrides the cluster level config
#        storeType: filestore
#    - name: "172.17.4.301"
#      deviceFilter: "^sd."
     - name: "192.168.100.101"
       devices:
       - name: "sdb"
     - name: "192.168.100.102"
       devices:
       - name: "sdb"
     - name: "192.168.100.103"
       devices:
       - name: "sdb"
  # The section for configuring management of daemon disruptions during upgrade or fencing.
  disruptionManagement:
    # If true, the operator will create and manage PodDisruptionBudgets for OSD, Mon, RGW, and MDS daemons. OSD PDBs are managed dynamically
    # via the strategy outlined in the [design](https://github.com/rook/rook/blob/master/design/ceph-managed-disruptionbudgets.md). The operator will
    # block eviction of OSDs by default and unblock them safely when drains are detected.
    managePodBudgets: false
    # A duration in minutes that determines how long an entire failureDomain like `region/zone/host` will be held in `noout` (in addition to the
    # default DOWN/OUT interval) when it is draining. This is only relevant when  `managePodBudgets` is `true`. The default value is `30` minutes.
    osdMaintenanceTimeout: 30
    # If true, the operator will create and manage MachineDisruptionBudgets to ensure OSDs are only fenced when the cluster is healthy.
    # Only available on OpenShift.
    manageMachineDisruptionBudgets: false
    # Namespace in which to watch for the MachineDisruptionBudgets.
    machineDisruptionBudgetNamespace: openshift-machine-api


```

- 启动cluster.yaml

```shell
kubectl apply -f cluster.yaml

```

- 查看整个rook-ceph资源，看到所有的pod都Running就行了


```shell
root@hongmeng:~# kubectl -n rook-ceph get pod -o wide
NAME                                           READY   STATUS             RESTARTS   AGE     IP                NODE              NOMINATED NODE   READINESS GATES
csi-cephfsplugin-cqv6r                         3/3     Running            4          27h     192.168.100.102   192.168.100.102   <none>           <none>
csi-cephfsplugin-jhrkx                         3/3     Running            3          27h     192.168.100.103   192.168.100.103   <none>           <none>
csi-cephfsplugin-kr68m                         3/3     Running            4          27h     192.168.100.101   192.168.100.101   <none>           <none>
csi-cephfsplugin-provisioner-974b566d9-9mgg7   4/4     Running            38         27h     172.20.2.23       192.168.100.103   <none>           <none>
csi-cephfsplugin-provisioner-974b566d9-j7rjr   4/4     Running            47         27h     172.20.1.25       192.168.100.102   <none>           <none>
csi-rbdplugin-cj2g6                            3/3     Running            6          27h     192.168.100.101   192.168.100.101   <none>           <none>
csi-rbdplugin-njt7h                            3/3     Running            4          27h     192.168.100.102   192.168.100.102   <none>           <none>
csi-rbdplugin-provisioner-579c546f5-sxm6g      5/5     Running   56         27h     172.20.0.26       192.168.100.101   <none>           <none>
csi-rbdplugin-provisioner-579c546f5-z44dh      5/5     Running            58         27h     172.20.2.25       192.168.100.103   <none>           <none>
csi-rbdplugin-qkxmv                            3/3     Running            4          27h     192.168.100.103   192.168.100.103   <none>           <none>
rook-ceph-detect-version-b2lsf                 1/1     Running           0          4m14s   <none>            192.168.100.101   <none>           <none>
rook-ceph-mgr-a-65bff76dfd-vb6t6               1/1     Running            25         26h     172.20.0.22       192.168.100.101   <none>           <none>
rook-ceph-mon-a-7c5f69c694-657f8               1/1     Running            1          26h     172.20.1.24       192.168.100.102   <none>           <none>
rook-ceph-mon-d-769794bf6-x7nzk                1/1     Running            1          26h     172.20.0.25       192.168.100.101   <none>           <none>
rook-ceph-mon-e-76968c9945-nfc4z               1/1     Running            1          26h     172.20.2.28       192.168.100.103   <none>           <none>
rook-ceph-operator-7c948fc7c7-cnhhz            1/1     Running            28         27h     172.20.0.24       192.168.100.101   <none>           <none>
rook-ceph-osd-0-c96d6f444-tcmbk                1/1     Running            3          26h     172.20.0.23       192.168.100.101   <none>           <none>
rook-ceph-osd-1-6755d6d96-2gnnc                1/1     Running            1          26h     172.20.1.27       192.168.100.102   <none>           <none>
rook-ceph-osd-2-6ff8b495d-hj7n2                1/1     Running            7          26h     172.20.2.20       192.168.100.103   <none>           <none>
rook-ceph-osd-prepare-192.168.100.101-v9v94    1/1     Running          0          77m     172.20.0.43       192.168.100.101   <none>           <none>
rook-ceph-osd-prepare-192.168.100.102-zdr6m    1/1     Running          0          77m     172.20.1.46       192.168.100.102   <none>           <none>
rook-ceph-osd-prepare-192.168.100.103-r4849    1/1     Running          0          76m     172.20.2.45       192.168.100.103   <none>           <none>
rook-ceph-tools-cfb9d595b-sbp2w                1/1     Running            2          26h     192.168.100.101   192.168.100.101   <none>           <none>
rook-discover-28vvn                            1/1     Running            1          27h     172.20.2.21       192.168.100.103   <none>           <none>
rook-discover-kkw94                            1/1     Running            1          27h     172.20.0.21       192.168.100.101   <none>           <none>
rook-discover-x7pd4                            1/1     Running            1          27h     172.20.1.29       192.168.100.102   <none>           <none>


```

- 在每台集群主机中输入以下命令查看存储空间是否开辟成功

```shell
root@hongmeng:~# lsblk
NAME                                                                                                 MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0                                                                                                  7:0    0 88.5M  1 loop /snap/core/7270
sda                                                                                                    8:0    0   60G  0 disk 
├─sda1                                                                                                 8:1    0    1M  0 part 
└─sda2                                                                                                 8:2    0   60G  0 part /
sdb                                                                                                    8:16   0   20G  0 disk 
└─ceph--c9c8db31--0d3a--4d5f--8f45--673193abfb7b-osd--data--8d432219--7773--440b--a913--76396e63c9f1 253:0    0   19G  0 lvm  
sr0                                                                                                   11:0    1  848M  0 rom  


```

- 启动ceph dashboard

```shell
kubectl apply -f dashboard-external-https.yaml

```

- 查看service服务状态

```shell
root@hongmeng:~# kubectl -n rook-ceph get service
NAME                                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
csi-cephfsplugin-metrics                 ClusterIP   10.68.34.45     <none>        8080/TCP,8081/TCP   27h
csi-rbdplugin-metrics                    ClusterIP   10.68.220.246   <none>        8080/TCP,8081/TCP   27h
rook-ceph-mgr                            ClusterIP   10.68.204.98    <none>        9283/TCP            26h
rook-ceph-mgr-dashboard                  ClusterIP   10.68.244.107   <none>        8443/TCP            26h
rook-ceph-mgr-dashboard-external-https   NodePort    10.68.59.174    <none>        8443:21713/TCP      27h
rook-ceph-mon-a                          ClusterIP   10.68.66.138    <none>        6789/TCP,3300/TCP   27h
rook-ceph-mon-d                          ClusterIP   10.68.101.219   <none>        6789/TCP,3300/TCP   26h
rook-ceph-mon-e                          ClusterIP   10.68.128.162   <none>        6789/TCP,3300/TCP   26h


```

- 查看dashboard密码（用户为admin）

```shell
root@hongmeng:~# kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo

SddC1B35Xt

```

- 浏览器访问地址

```http
https://192.168.100.101:21713

```

- 修改flex文件夹中的ceph存储池配置文件成以下配置：storageclass.yaml

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
# 参考页面中 apiVersion使用了 ceph.rook.io/v1beta1
# 且 kind 使用了Pool，执行的时候找不到Pool的kind，因此将其改为上面两个值
metadata:
  #这个name就是创建成ceph pool之后的pool名字
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 2
  # size 池中数据的副本数,1就是不保存任何副本
  failureDomain: host
  #  failureDomain：数据块的故障域，
  #  值为host时，每个数据块将放置在不同的主机上
  #  值为osd时，每个数据块将放置在不同的osd上
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph
  # StorageClass的名字，pvc调用时填的名字
provisioner: ceph.rook.io/block
parameters:
  pool: replicapool
  # Specify the namespace of the rook cluster from which to create volumes.
  # If not specified, it will use `rook` as the default namespace of the cluster.
  # This is also the namespace where the cluster will be
  clusterNamespace: rook-ceph
  # Specify the filesystem type of the volume. If not specified, it will use `ext4`.
  fstype: xfs
# 设置回收策略默认为：Retain
reclaimPolicy: Retain
allowVolumeExpansion: true

```

- 启动storageclass.yaml

```shell
cd flex
kubectl apply -f storageclass.yaml

```

- 查看storageclass信息

```shell
root@hongmeng:~# kubectl get storageclasses.storage.k8s.io  -n rook-ceph

NAME   PROVISIONER          AGE
ceph   ceph.rook.io/block   26h


# 进入详情
root@hongmeng:~# kubectl describe storageclasses.storage.k8s.io  -n rook-ceph

Name:            ceph
IsDefaultClass:  No
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"allowVolumeExpansion":true,"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"ceph"},"parameters":{"clusterNamespace":"rook-ceph","fstype":"xfs","pool":"replicapool"},"provisioner":"ceph.rook.io/block","reclaimPolicy":"Retain"}

Provisioner:           ceph.rook.io/block
Parameters:            clusterNamespace=rook-ceph,fstype=xfs,pool=replicapool
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Retain
VolumeBindingMode:     Immediate
Events:                <none>

```

- （可选）配置带存储池，带ambassador设置的nginx的示例：sto-amb-nginx.yaml（已放入/tmp/k8s-plug）

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ceph
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
  labels:
    role: web
    app: nginx
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
        role: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80

        volumeMounts:
        - mountPath: /html
          name: http-file
      volumes:
      - name: http-file
        persistentVolumeClaim:
          claimName: nginx-pvc
---
kind: Service
apiVersion: v1
metadata:
  name: my-nginx-service
  annotations:
    getambassador.io/config: |
      ---
      apiVersion: ambassador/v1
      kind: Mapping
      name: nginx_mapping
      host: nginx.kestrel-info.com
      service: my-nginx-service:80
      prefix: /
spec:
  selector:
    app: nginx
    role: web
  ports:
  - protocol: TCP
    port: 80


```

- 启动nginx示例可以在ceph dashboard中看到有pool创建

```shell
cd /tmp/k8s-plug
kubectl apply -f sto-amb-nginx.yaml

```

- 查看nginx示例的请求，也可以在ceph的dashboard内Block-->images选项卡中看到

```shell
root@hongmeng:~# kubectl get pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nginx-pvc   Bound    pvc-ba12ab9d-481d-4cb1-862e-b82e0399174f   1Gi        RWX            ceph           4m9s

```



#### 在CEPH dashboard中遇到HEALTH_WARN的问题

```
HEALTH_WARN
            too few PGs per OSD (5 < min 30)

```

- 解决：创建tookit容器

```shell
cd /home/hongmeng/rook-1.1.4/cluster/examples/kubernetes/ceph
kubectl create -f toolbox.yaml

```

- 查看pod

```shell
root@hongmeng:~# kubectl get pods -n rook-ceph | grep tools

rook-ceph-tools-cfb9d595b-bnb8h                1/1     Running            2          27h

```

- 进入pod内部


```shell
kubectl exec -it rook-ceph-tools-cfb9d595b-bnb8h -n rook-ceph -- /bin/bash

```

- 执行以下命令

```shell
rados lspools
ceph osd pool set replicapool pg_num 128
ceph osd pool set replicapool pgp_num 128

```



## 插件EFK安装

- 修改EFK的挂载存储池文件es-statefulset.yaml，将storageClassName改为 "ceph"与插件创建的匹配

```shell
root@hongmeng:~# vi /etc/ansible/manifests/efk/es-dynamic-pv/es-statefulset.yaml

 volumeClaimTemplates:
  - metadata:
      name: elasticsearch-logging
    spec:
      accessModes: [ "ReadWriteMany" ]
      storageClassName: "ceph"
      resources:
        requests:
          storage: 4Gi

```

- 运行EFK安装文件


```shell
# 安装
kubectl apply -f /etc/ansible/manifests/efk/
kubectl apply -f /etc/ansible/manifests/efk/es-dynamic-pv/

# 删除
kubectl delete -f /etc/ansible/manifests/efk/
kubectl delete -f /etc/ansible/manifests/efk/es-dynamic-pv/

```

- 验证EFK安装


```shell
root@test-k8s-master1:~# kubectl get pods -n kube-system|grep -E 'elasticsearch|fluentd|kibana'

elasticsearch-logging-0                       1/1     Running   0          33s
elasticsearch-logging-1                       1/1     Running   0          23s
fluentd-es-v2.4.0-6tqbq                       1/1     Running   0          40s
fluentd-es-v2.4.0-qqgmp                       1/1     Running   0          40s
fluentd-es-v2.4.0-vhq4p                       1/1     Running   0          40s
kibana-logging-ffd866674-kcz98                1/1     Running   0          39s

```

- 获取kibana运行的地址

```shell
root@test-k8s-master1:~# kubectl cluster-info | grep Kibana

Kibana is running at https://192.168.100.101:6443/api/v1/namespaces/kube-system/services/kibana-logging/proxy

```

- 浏览器访问kibana地址(默认用户密码为之前认证的账号密码:hongmeng ; it168)


```http
https://192.168.100.101:6443/api/v1/namespaces/kube-system/services/kibana-logging/proxy

```

- 创建kibana实例


1. 选择Management-->Index Patterns
2. 在index pattern输入框中输入默认的pattern：logstash-*	-->Next step
3. 在Time Filter field name下拉框中选择@timestamp -->Create index pattern
4. 日志索引已经完成，可以在界面中看到



## 可选k8s项目测试操作

- 启动私有仓库容器 


```shell
docker run -di --name=registry -p 5000:5000 registry 

```

- 打 开 浏 览 器 输 入 地 址 **http://192.168.100.101:5000/v2/_catalog** 看 到 


**{"repositories":[]}** 表示私有仓库搭建成功并且内容为空 。**IP地址是自己虚拟机的地址**

- 修改 /etc/docker/daemon.json


添加以下内容，保存退出。 此步用于让 docker 信任私有仓库地址 

```json
{
	"insecure-registries":["192.168.100.101:5000"]
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
docker tag testdemo 192.168.100.101:5000/testdemo

```

- 再次启动私服容器 


```shell
docker start registry 

```

- 上传标记的镜像 ,再次刷新浏览器地址http://192.168.100.101:5000/v2/_catalog可以看到新增加的镜像


```shell
docker push 192.168.100.101:5000/testdemo

```

- 将准备好的两个yaml进行运行,只需要修改name和image这两个选项（注意要空格）


```yaml
name: hello
image: 192.168.100.101:5000/testdemo

```

- 根据两个yaml创建运行服务


```shell
kubectl create -f hello-rc.yaml
kubectl create -f hello-svc.yaml

```

- 查看kubectel中的服务


```shell
root@test-k8s-master1:~# kubectl get svc
hello	NodePort	10.68.160.46	<none>	8080:30003/TCP

```

- 访问项目地址:


```http
http://192.168.100.101:30003/hello

hello docker

```



## 附录

### 安装后使用tab键遇到的问题

```shell
_get_comp_words_by_ref: command not found
```

- 解决：

```shell
source /etc/bash_completion
source <(kubectl completion bash)
```



### 已经创建的PVC如何扩容？

- 首先storageclass开启allowVolumeExpansion:


```shell
allowVolumeExpansion: true
```

- 修改PVC的容器（不能比之前小）

- 重启pod　


