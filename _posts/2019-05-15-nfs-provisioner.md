---
layout: post
title: Openshift 中部署 NFS Provisioner
tags:
  - Openshift
  - NFS

---

<br>

### 主机列表

|IP|NFS角色|Openshift角色|
|---|---|---|
|192.168.1.222|Server|Master|
|192.168.1.223|Client|Worker|
|192.168.1.224|Client|Worker|

### NFS Server 

yum 安装 nfs rpcbind

```
yum install -y nfs-utils rpcbind
```

创建目录

```
mkdir /nfsdata
chmod 777 /nfsdata
```

编辑/etc/exports文件, 添加如下内容

```
/nfsdata  192.168.0.0/16(rw,sync,no_root_squash)
```

启动nfs

```
systemctl enable rpcbind
systemctl enable nfs-server
systemctl start rpcbind
systemctl start nfs-server
```

查看开放的端口

```
rpcinfo -p
```

![](https://raw.githubusercontent.com/arugaki/arugaki.github.io/master/images/nfs_rpcinfo.jpg)

编辑 /etc/sysconfig/iptables, 添加如下内容, 开放之前 `rpcinfo -p` 显示的端口

```
-A INPUT -p tcp -m state --state NEW -m tcp --dport 111 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 42979 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 20048 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 2049 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 37266 -j ACCEPT
-A INPUT -p udp --dport 111 -j ACCEPT
-A INPUT -p udp --dport 45338 -j ACCEPT
-A INPUT -p udp --dport 20048 -j ACCEPT
-A INPUT -p udp --dport 2049 -j ACCEPT
-A INPUT -p udp --dport 54248 -j ACCEPT
```

重启 iptables

```
systemctl reload iptables
```

检查是否成功安装

```
showmount -e
// 返回如下
Export list for 192.168.1.222:
/nfsdata 192.168.0.0/16
```

### NFS Client

yum 安装 nfs

```
yum install -y nfs-utils
```

检查是否成功安装

```
showmount -e 192.168.1.222
// 返回如下
Export list for 192.168.1.222:
/nfsdata 192.168.0.0/16
```

### NFS Provisioner

从<a href="https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client/deploy"> 该地址 </a> 下载所有 yaml 文件到本地, 假设下载至目录 /root/nfs_provisioner/

```
wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/rbac.yaml
wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/deployment.yaml
wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/class.yaml
wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/test-claim.yaml
wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/test-pod.yaml

```
修改 deployment.yaml

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
---
kind: Deployment
apiVersion: extensions/v1beta1
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
              value: 192.168.1.222
            - name: NFS_PATH
              value: /nfsdata
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.1.222
            path: /nfsdata
```

创建serviceaccount等权限信息

```
// 将nfs provisioner 部署在 default namespace 下
oc login -u system:admin
oc project default

NAMESPACE=`oc project -q`
sed -i'' "s/namespace:.*/namespace: $NAMESPACE/g" /root/nfs_provisioner/rbac.yaml
oc create -f /root/nfs_provisioner/rbac.yaml
oadm policy add-scc-to-user hostmount-anyuid system:serviceaccount:$NAMESPACE:nfs-client-provisioner
```

部署nfs provider

```
oc create -f /root/nfs_provisioner/deployment.yaml
```

部署 storageclass 并设为默认

```
oc create -f /root/nfs_provisioner/class.yaml

kubectl patch storageclass managed-nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### 测试部署是否成功

```
// 创建pvc
oc create -f /root/nfs_provisioner/test-claim.yaml
// 创建使用pvc的pod
oc create -f /root/nfs_provisioner/test-pod.yaml
```





