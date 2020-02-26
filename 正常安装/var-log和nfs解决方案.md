

# 方案背景：

我们在k8s系统上部署接口的时候，发现master1节点上有部分pod出现DiskPressure的现象。经排查发现master1节点上的/var/log文件夹出现了两个较大的日志文件,分别是syslog和kern.log。经排查发现为k8s系统中storageclass版本问题。为了确保剩下的接口pod能够顺利部署，因此创建了以下解决方案。

# 操作内容列表

- 添加了一个新的lv分区

- 将/var/log挂载到新的分区上

- 修改了k8s系统中创建storageclass的版本



# 操作参考文档

详细请参考：[广州市税务局tpass-k8s集群安装初始版.md](http://gitlab.hongmeng-info.com/wangjing/workmd/blob/GZSW-TPASS/k8s/%E5%B9%BF%E5%B7%9E%E5%B8%82%E7%A8%8E%E5%8A%A1%E5%B1%80tpass-k8s%E9%9B%86%E7%BE%A4%E5%AE%89%E8%A3%85%E5%88%9D%E5%A7%8B%E7%89%88.md)



# /var/log解决方案

先将/var/log取别名保存

```shell
mv /var/log /var/log-old
```

使用软链接

```shell
ln -s /var/log-old /var/log
```

创建新的文件夹

```shell
mkdir /log
```

创建新的lv分区lv-data-log并格式化

```shell
lvcreate -L 5G -n lv-data-log volume-data

mkfs.ext4 /dev/volume-data/lv-data-log
```

获取UUID并将新的分设置为开启自动挂载

```shell
# 获取UUID
blkid /dev/volume-data/lv-data-log
# /dev/volume-data/lv-data-log: UUID="1959183f-996d-4515-b873-281c96c04f9c" TYPE="ext4"

#添加开机自动挂载
vi /etc/fstab
# 添加以下内容
# UUID=1959183f-996d-4515-b873-281c96c04f9c /log ext4 defaults 0 0

# 挂载分区到磁盘
mount /log
```

清除原来的/var/log

```shell
rm /var/log
```

设置新的软链接

```shell
ln -s /log /var/log
```



# NFS版本解决方案

修改配置文件

```shell
vi /etc/ansible/roles/cluster-storage/templates/nfs/nfs-client-provisioner.yaml.j2
```

在kind: StorageClass这部分修改为以下内容

```shell
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: {{ storage.nfs.storage_class }}
provisioner: {{ storage.nfs.provisioner_name }}
mountOptions:
- hard
- vers=4.0

```

找到原先的三个storageclass：nfs-1 nfs-2 nfs-3

```shell
root@test-k8s-master1:/var/log-1# kubectl get deployments --all-namespaces
NAMESPACE     NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
                    1/1     1            1           12d
kube-system   nfs-provisioner-01                      1/1     1            1           16h
kube-system   nfs-provisioner-02                      1/1     1            1           16h
kube-system   nfs-provisioner-03                      1/1     1            1           

```

删除这三个storageclass：nfs-1 nfs-2 nfs-3

```shell
kubectl -n kube-system delete deployments nfs-provisioner-01
kubectl -n kube-system delete deployments nfs-provisioner-02
kubectl -n kube-system delete deployments nfs-provisioner-03
```

修改storageclass的配置文件重新创建三个新的storageclass

```shell
vi /etc/ansible/roles/cluster-storage/defaults/main.yml
```

```shell
# 修改storage.server;storage.storage_class;storage.provisioner_name;aliyun_nas.storage_class;aliyun_nas.controller_name五个属性即可成功创建新的storageclass
# 动态存储类型, 目前支持自建nfs和aliyun_nas
storage:
  # nfs server 参数
  nfs:
    enabled: "yes"
    server: "192.168.10.18"
    server_path: "/mnt/nfs-data1"
    storage_class: "nfs-3"
    provisioner_name: "nfs-provisioner-03"

  # aliyun_nas 参数
  aliyun_nas:
    enabled: "no"
    server: "xxxxxxxxxxx.cn-hangzhou.nas.aliyuncs.com"
    server_path: "/"
    storage_class: "class-aliyun-nas-03"
    controller_name: "aliyun-nas-controller-03"

```

执行命令创建storageclass

```shell
ansible-playbook /etc/ansible/roles/cluster-storage/cluster-storage.yml
```

查看新的三个storageclass信息

```shell
root@test-k8s-master1:/var/log-1# kubectl describe storageclasses nfs-1
Name:            nfs-1
IsDefaultClass:  No
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"nfs-1"},"mountOptions":["hard","vers=4.0"],"provisioner":"nfs-provisioner-01"}

Provisioner:           nfs-provisioner-01
Parameters:            <none>
AllowVolumeExpansion:  <unset>
MountOptions:
  hard
  vers=4.0
ReclaimPolicy:      Delete
VolumeBindingMode:  Immediate
Events:             <none>


```

```shell
root@test-k8s-master1:/var/log-1# kubectl describe storageclasses nfs-2
Name:            nfs-2
IsDefaultClass:  No
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"nfs-2"},"mountOptions":["hard","vers=4.0"],"provisioner":"nfs-provisioner-02"}

Provisioner:           nfs-provisioner-02
Parameters:            <none>
AllowVolumeExpansion:  <unset>
MountOptions:
  hard
  vers=4.0
ReclaimPolicy:      Delete
VolumeBindingMode:  Immediate
Events:             <none>

```

```shell
root@test-k8s-master1:/var/log-1# kubectl describe storageclasses nfs-3
Name:            nfs-3
IsDefaultClass:  No
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"nfs-3"},"mountOptions":["hard","vers=4.0"],"provisioner":"nfs-provisioner-03"}

Provisioner:           nfs-provisioner-03
Parameters:            <none>
AllowVolumeExpansion:  <unset>
MountOptions:
  hard
  vers=4.0
ReclaimPolicy:      Delete
VolumeBindingMode:  Immediate
Events:             <none>

```

注：新的三个storageclass的应用等于k8s系统插件的安装过程,这里省略！详细请转到上方的参考文档！