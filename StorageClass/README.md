# Kubernetes StorageClass

- 要使用 StorageClass，就得安装对应的自动配置程序，比如这里存储后端使用的是 nfs，那么就需要使用到一个 nfs-client 的自动配置程序，也叫它 Provisioner，这个程序使用我们已经配置好的 nfs 服务器，来自动创建持久卷，也就是自动创建 PV。

- 自动创建的 PV 以${namespace}-${pvcName}-${pvName}这样的命名格式创建在 NFS 服务器上的共享数据目录中;

- 而当这个 PV 被回收后会以archieved-${namespace}-${pvcName}-${pvName}这样的命名格式存在 NFS 服务器上。

- NFS服务端安装配置
```shell script
yum install -y nfs-utils rpcbind
cat >/etc/exports<<EOF
/data/nfs_data 192.168.111.0/24(rw,sync,all_squash,anonuid=1000,anongid=1000)
EOF
chmod -R 777 /data/nfs_data/
systemctl start rpcbind #启动rpcbind服务
systemctl start nfs #启动nfs服务，其实启动nfs服务时rpc相关服务也会启动
systemctl enable rpcbind #设置开机启动rpcbind
systemctl enable nfs #设置开机启动nfs，服务端设置，客户端不需要
showmount -e 192.168.111.128 #查看服务端共享的目录
yum install nfs-utils -y && mount -t nfs 192.168.111.128:/data/nfs_data /mnt #挂载共享目录
```
- 【踩坑】rpcbind启动失败
```shell script
[root@k8s-master01 data]# journalctl -u rpcbind
-- Logs begin at Tue 2020-01-07 15:00:01 CST, end at Tue 2020-01-07 15:08:51 CST. --
Jan 07 15:00:07 k8s-master01 systemd[1]: Dependency failed for RPC bind service.
Jan 07 15:00:07 k8s-master01 systemd[1]: Job rpcbind.service/start failed with result 'dependency'.
Jan 07 15:04:18 k8s-master01 systemd[1]: Dependency failed for RPC bind service.
Jan 07 15:04:18 k8s-master01 systemd[1]: Job rpcbind.service/start failed with result 'dependency'.
Jan 07 15:04:29 k8s-master01 systemd[1]: Dependency failed for RPC bind service.
Jan 07 15:04:29 k8s-master01 systemd[1]: Job rpcbind.service/start failed with result 'dependency'.

# 添加
[root@k8s-master01 data]# cat /etc/sysctl.conf
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
[root@k8s-master01 data]# sysctl -p
```


- NFS-Client安装
```shell script
# 第一步：配置 Deployment，将里面的对应的参数替换成我们自己的 nfs 配置（nfs-client.yaml）
[root@k8s-master01 data]# vim nfs-client-deployment.yaml 
kind: Deployment
apiVersion: apps/v1beta1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 192.168.111.128
            - name: NFS_PATH
              value: /data/nfs_data/storageclass
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.111.128
            path: /data/nfs_data/storageclass
```

```shell script
# 第二步：将环境变量 NFS_SERVER 和 NFS_PATH 替换，当然也包括下面的 nfs 配置，我们可以看到这里使用了一个名为 nfs-client-provisioner 的serviceAccount，所以需要创建一个 sa，然后绑定上对应的权限：（nfs-client-sa.yaml）
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io

# 这里新建的一个名为 nfs-client-provisioner 的ServiceAccount，然后绑定了一个名为 nfs-client-provisioner-runner 的ClusterRole，而该ClusterRole声明了一些权限，其中就包括对persistentvolumes的增、删、改、查等权限，所以可以利用该ServiceAccount来自动创建 PV
```

```shell script
# 第三步：nfs-client 的 Deployment 声明完成后，就可以来创建一个StorageClass对象了; (nfs-client-storageclass.yaml)
# 声明了一个名为 course-nfs-storage 的StorageClass对象，注意下面的provisioner对应的值一定要和上面的Deployment下面的 PROVISIONER_NAME 这个环境变量的值一样。
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: course-nfs-storage
provisioner: fuseim.pri/ifs
```

```shell script
# 现在创建这些资源对象
kubectl apply -f nfs-client-deployment.yaml
kubectl apply -f nfs-client-sa.yaml
kubectl apply -f nfs-client-storageclass.yaml
```
- 【踩坑】 nfs-client启动失败
```shell script
[root@k8s-master01 cert]# kubectl get pod 
NAME                                      READY   STATUS   RESTARTS   AGE
nfs-client-provisioner-745856c897-2klnx   0/1     Error    0          8s

[root@k8s-master01 cert]# kubectl logs -f  nfs-client-provisioner-745856c897-2klnx
F0107 08:23:04.296958       1 provisioner.go:180] Error getting server version: Get https://10.101.0.1:443/version?timeout=32s: x509: certificate is valid for 127.0.0.1, 192.168.111.128, 192.168.111.129, 192.168.111.130, 192.168.10.10, not 10.101.0.1

# 添加IP 重启api-server
[root@k8s-master01 cert]# cat kubernetes-csr.json 
{
    "CN": "kubernetes",
    "hosts": [
        "127.0.0.1",
        "192.168.111.128",
        "192.168.111.129",
        "192.168.111.130",
        "192.168.10.10",
        "10.101.0.1",
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
          "ST": "BeiJing",
          "L": "BeiJing",
          "O": "k8s",
          "OU": "4Paradigm"
        }
    ]
}
```







