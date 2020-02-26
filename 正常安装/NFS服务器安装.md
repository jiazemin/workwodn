# K8S-动态pv-NFS服务器配置

#### NFS服务器创建

```shell
# 将ubuntu18.04_nfs-server.sh和ubuntu18.04_nfs-server.tar.gz放入/tmp目录
sh ubuntu18.04_nfs-server.sh
```

#### 配置共享目录

| 参数             | 说明                                                         |
| ---------------- | ------------------------------------------------------------ |
| ro               | 只读访问                                                     |
| rw               | 读写访问                                                     |
| sync             | 所有数据在请求时写入共享                                     |
| async            | nfs在写入数据前可以响应请求                                  |
| secure           | nfs通过1024以下的安全TCP/IP端口发送                          |
| insecure         | nfs通过1024以上的端口发送                                    |
| wdelay           | 如果多个用户要写入nfs目录，则归组写入（默认）                |
| no_wdelay        | 如果多个用户要写入nfs目录，则立即写入，当使用async时，无需此设置 |
| hide             | 在nfs共享目录中不共享其子目录                                |
| no_hide          | 共享nfs目录的子目录                                          |
| subtree_check    | 如果共享/usr/bin之类的子目录时，强制nfs检查父目录的权限（默认） |
| no_subtree_check | 不检查父目录权限                                             |
| all_squash       | 共享文件的UID和GID映射匿名用户anonymous，适合公用目录        |
| no_all_squash    | 保留共享文件的UID和GID（默认）                               |
| root_squash      | root用户的所有请求映射成如anonymous用户一样的权限（默认）    |
| no_root_squash   | root用户具有根目录的完全管理访问权限                         |
| anonuid=xxx      | 指定nfs服务器/etc/passwd文件中匿名用户的UID                  |
| anongid=xxx      | 指定nfs服务器/etc/passwd文件中匿名用户的GID                  |

- 编辑`/etc/exports`文件添加需要共享目录，每个目录的设置独占一行，编写格式如下：

NFS共享目录路径 客户机IP或者名称(参数1,参数2,...,参数n)

```shell
/mnt/nfs       192.168.10.*(rw,sync,insecure,no_subtree_check,no_root_squash)
```

- 注1： 尽量指定主机名或IP或IP段最小化授权可以访问NFS 挂载的资源的客户端 
- 注2：经测试参数insecure必须要加，否则客户端挂载出错mount.nfs: access denied by server while mounting

####  配置完成后，您可以在终端提示符后运行以下命令来启动 NFS 服务器： 

```shell
systemctl start nfs-kernel-server.service
```

#### 在其他机器中创建一个文件夹与NFS服务器共享目录相连接

```shell
mount -t nfs 192.168.10.16:/mnt/nfs  /mnt/
```

#### 创建动态pv

-  编辑自定义配置文件：/etc/ansible/roles/cluster-storage/defaults/main.yml

```yaml
# 动态存储类型, 目前支持自建nfs和aliyun_nas
storage:
  # nfs server 参数
  nfs:
    enabled: "yes"
    server: "192.168.10.16"
    server_path: "/mnt/nfs"
    storage_class: "nfs-dynamic-class"
    provisioner_name: "nfs-provisioner-01"

  # aliyun_nas 参数
  aliyun_nas:
    enabled: "no"
    server: "xxxxxxxxxxx.cn-hangzhou.nas.aliyuncs.com"
    server_path: "/"
    storage_class: "class-aliyun-nas-01"
    controller_name: "aliyun-nas-controller-01"

```

-  创建 nfs provisioner 

```shell
ansible-playbook /etc/ansible/roles/cluster-storage/cluster-storage.yml
```

- 验证 nfs provisioner 

```shell
root@TPASS-K8S-MASTER1:~# kubectl get pod --all-namespaces |grep nfs-prov
kube-system   nfs-provisioner-01-5947746749-5zfxl            1/1     Running            0          3d19h
```

#### 配置一个test用例测试pv是否能用

- 修改/etc/ansible/manifests/storage/test.yaml（注意storageClassName与上方pv设置相同）

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
spec:
  storageClassName: nfs-dynamic-class
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi

---
kind: Pod
apiVersion: v1
metadata:
  name: test
spec:
  containers:
  - name: test
    image: busybox:latest
    imagePullPolicy: IfNotPresent
    command:
      - "/bin/sh"
    args:
      - "-c"
      - "echo 'hello k8s' > /mnt/SUCCESS && sleep 36000 || exit 1"
    volumeMounts:
      - name: nfs-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-claim

```

```shell
kubectl apply -f /etc/ansible/manifests/storage/test.yaml
```

- 验证测试用例

```shell
root@TPASS-K8S-MASTER1:~# kubectl get pod --all-namespaces |grep test
default       test                                           1/1     Running            0          4m40s

root@TPASS-K8S-MASTER1:~# kubectl get pvc
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
test-claim   Bound    pvc-5635edec-5727-4bf7-aa20-a97b0ab38872   1Mi        RWX            nfs-dynamic-class   5m1s

```

# 问题附录:

#### 出现创建的测试用例无法挂载情况pending：/etc/exports文件格式不正确

```shell
# ip后面不要有空格，与()相连
/mnt/nfs       192.168.10.*(rw,sync,insecure,no_subtree_check,no_root_squash)
```

