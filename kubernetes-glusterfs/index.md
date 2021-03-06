# kubernetes glusterfs



# kubernetes glusterfs

## 安装 glusterfs

```
# 先安装 gluster 源
yum install centos-release-gluster -y

# 安装 glusterfs 组件
yum install -y glusterfs glusterfs-server glusterfs-fuse glusterfs-rdma glusterfs-geo-replication glusterfs-devel
```

```
## 创建 glusterfs 目录

mkdir /opt/glusterd

## 修改 glusterd 目录
sed -i 's/var\/lib/opt/g' /etc/glusterfs/glusterd.vol
```



```
# 启动 glusterfs
systemctl start glusterd.service

# 设置开机启动
systemctl enable glusterd.service

#查看状态
systemctl status glusterd.service
```


## 配置 glusterfs

```
# 配置 hosts

vi /etc/hosts

gluster-1  10.6.0.52
gluster-2  10.6.0.53
gluster-3  10.6.0.55
gluster-4  10.6.0.56
```

```
# 开放端口

iptables -I INPUT -p tcp --dport 24007 -j ACCEPT

```

```
# 创建存储目录

mkdir /opt/gfs_data

```


```
# 添加节点到 集群
# 执行操作的本机不需要probe 本机
[root@gluster-1 ~]#
gluster peer probe gluster-2
gluster peer probe gluster-3
gluster peer probe gluster-4



# 查看集群状态
gluster peer status

```


## 配置 volume 

```
# 创建 分布卷
gluster volume create dht-volume transport tcp gluster-1:/opt/gfs_data gluster-2:/opt/gfs_data

# 查看volume状态
gluster volume info

...
Type: Distribute
...

# 启动 分布卷
gluster volume start dht-volume

```

```
# 创建 复制卷

gluster volume create afr-volume replica 2 transport tcp gluster-1:/opt/afr_data gluster-2:/opt/afr_data


# 查看volume状态
gluster volume info

...
Type: Replicate
...

# 启动 复制卷
gluster volume start afr-volume

```


```
# 创建 条带卷
gluster volume create str-volume stripe 2 transport tcp gluster-1:/opt/str_data gluster-2:/opt/str_data

# 查看volume状态
gluster volume info

...
Type: Stripe
...

# 启动 条带卷
gluster volume start str-volume

```


```
# 上面三种基本模式，可以互相组合

# 这里我们使用 组合 分布式复制卷 需要最少4台服务器 replica 必须为倍数

gluster volume create k8s-volume replica 2 transport tcp gluster-1:/opt/gfs_data gluster-2:/opt/gfs_data gluster-3:/opt/gfs_data gluster-4:/opt/gfs_data


# 查看 volume 状态

gluster volume info

Volume Name: k8s-volume
Type: Distributed-Replicate
Volume ID: 981c41fa-bbe1-4a36-a1e2-9f76de1dc8f1
Status: Created
Snapshot Count: 0
Number of Bricks: 2 x 2 = 4
Transport-type: tcp
Bricks:
Brick1: gluster-1:/opt/gfs_data
Brick2: gluster-2:/opt/gfs_data
Brick3: gluster-3:/opt/gfs_data
Brick4: gluster-4:/opt/gfs_data
Options Reconfigured:
transport.address-family: inet
nfs.disable: on


# 启动 k8s-volume
gluster volume start k8s-volume

```


## gluster 调优

```
# 开启 指定 volume 的配额
gluster volume quota k8s-volume enable

# 限制 指定 volume 的配额
gluster volume quota k8s-volume limit-usage / 5TB

# 设置 cache 大小, 默认32MB
gluster volume set k8s-volume performance.cache-size 4GB

# 设置 io 线程, 太大会导致进程崩溃
gluster volume set k8s-volume performance.io-thread-count 16

# 设置 网络检测时间, 默认42s
gluster volume set k8s-volume network.ping-timeout 10

# 设置 目录索引的自动愈合进程
gluster volume set k8s-volume cluster.self-heal-daemon on

# 设置 自动愈合的检测间隔, 默认600s
gluster volume set k8s-volume cluster.heal-timeout 300

# 设置 写缓冲区的大小, 默认1M
gluster volume set k8s-volume performance.write-behind-window-size 1024MB
```


# kubernetes 配置 glusterfs

```
官方的文档 https://github.com/kubernetes/kubernetes/tree/master/examples/volumes/glusterfs
```

## kubernetes 安装客户端

```
# 在所有 k8s node 中安装 glusterfs 客户端

yum install -y glusterfs glusterfs-fuse

# 配置 hosts

vi /etc/hosts

gluster-1  10.6.0.52
gluster-2  10.6.0.53
gluster-3  10.6.0.55
gluster-4  10.6.0.56

```


## 配置 endpoints

```
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/examples/volumes/glusterfs/glusterfs-endpoints.json

# 修改 endpoints.json ，配置 glusters 集群节点ip
# 每一个 addresses 为一个 ip 组

    {
      "addresses": [
        {
          "ip": "10.6.0.52"
        }
      ],
      "ports": [
        {
          "port": 1
        }
      ]
    },



# 导入 glusterfs-endpoints.json

kubectl apply -f glusterfs-endpoints.json


# 查看 endpoints 信息
kubectl get ep

```

## 配置 service

```
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/examples/volumes/glusterfs/glusterfs-service.json

# service.json 不需要配置，里面查找的是 enpointes 的名称与端口，端口默认配置为 1

# 导入 glusterfs-service.json
kubectl apply -f glusterfs-service.json

# 查看 service 信息
kubectl get svc
```

## 创建 测试 pod

```
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/examples/volumes/glusterfs/glusterfs-pod.json

# 编辑 glusterfs-pod.json
# 修改 volumes  下的 path 为上面创建的 volume 名称

"path": "k8s-volume"


# 导入 glusterfs-pod.json
kubectl apply -f glusterfs-pod.json


# 查看 pods 状态
kubectl get pods               
NAME                             READY     STATUS    RESTARTS   AGE
glusterfs                        1/1       Running   0          1m


# 查看 pods 所在 node

kubectl describe pods/glusterfs

# 登陆 node 物理机 使用 df 可查看 挂载目录
```


## 配置 pv 

> PersistentVolume（pv）和 PersistentVolumeClaim（pvc）是k8s提供的两种API资源，用于抽象存储细节。管理员关注于如何通过pv提供存储功能而无需关注用户如何使用，同样的用户只需要挂载pvc到容器中而不需要关注存储卷采用何种技术实现。

> pvc和pv的关系与pod和node关系类似，前者消耗后者的资源。pvc可以向pv申请指定大小的存储资源并设置访问模式。

> pv 属性 
> 1. storage 容量
> 2. 读写属性 分别为 - ReadWriteOnce：单个节点读写 , ReadOnlyMany：多节点只读 , ReadWriteMany：多节点读写 。


```
vi glusterfs-pv.yaml

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gluster-dev-volume
spec:
  capacity:
    storage: 8Gi
  accessModes:
    - ReadWriteMany
  glusterfs:
    endpoints: "glusterfs-cluster"
    path: "k8s-volume"
    readOnly: false
---


# 导入 pv

kubectl apply -f glusterfs-pv.yaml


# 查看 pv

kubectl get pv

```

> pvc 属性
> 1. 访问属性 与 pv 相同
> 2. 容量，向pv申请的容量 <= pv 总容量

## 配置 pvc

```

---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: glusterfs-nginx
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 8Gi
---


# 导入 pvc
kubectl apply -f glusterfs-pvc.yaml

# 查看 pvc

kubectl get pv

# STATUS 为 Bound ， VOLUME 为 pv name

```


## 创建 nginx  deployment 挂载 volume

```
vi nginx-deployment.yaml

---
apiVersion: extensions/v1beta1 
kind: Deployment 
metadata: 
  name: nginx-dm
spec: 
  replicas: 2
  template: 
    metadata: 
      labels: 
        name: nginx 
    spec: 
      containers: 
        - name: nginx 
          image: nginx:alpine 
          imagePullPolicy: IfNotPresent
          ports: 
            - containerPort: 80
          volumeMounts:
            - name: gluster-dev-volume
              mountPath: "/usr/share/nginx/html"
      volumes:
      - name: gluster-dev-volume
        persistentVolumeClaim:
          claimName: glusterfs-nginx
          
---


# 导入 deployment

kubectl apply -f nginx-deployment.yaml 


# 查看 deployment

kubectl get pods |grep nginx-dm
nginx-dm-2784556780-cnjdw        1/1       Running   0          8m
nginx-dm-2784556780-pt8vf        1/1       Running   0          8m


# 查看 挂载
kubectl exec -it nginx-dm-2784556780-cnjdw -- df -h|grep k8s-volume

# 创建文件 测试

kubectl exec -it nginx-dm-2784556780-cnjdw -- touch /usr/share/nginx/html/index.html

kubectl exec -it nginx-dm-2784556780-pt8vf -- ls -lt /usr/share/nginx/html/index.html


# 验证 glusterfs
# 因为我们使用 分布式复制卷，所以可以看到2个节点中有文件

[root@gluster-1 ~] ls /opt/gfs_data/
[root@gluster-2 ~] ls /opt/gfs_data/
[root@gluster-3 ~] ls /opt/gfs_data/
[root@gluster-4 ~] ls /opt/gfs_data/

```



## FAQ 问题

```
# 在使用 pv 与 pvc 的过程中遇到问题

# 第一次创建 pv 与 pvc 的时候 状态都是OK的

# 当删除 pvc 以后，再创建 pvc 状态一直 pending 无论如何都不正常

# 查看日志 报 no persistent volumes available for this claim and no storage class is set

# 这里我们可以跳过 pv 与 pvc 只需要创建一个 ep 就可以

# 挂载的时候我们直接在 yaml 目录下 写 ep 就行

# 如下为 deployment 的 yaml 文件


apiVersion: extensions/v1beta1 
kind: Deployment 
metadata: 
  name: nginx-dm
spec: 
  replicas: 2
  template: 
    metadata: 
      labels: 
        name: nginx 
    spec: 
      containers: 
        - name: nginx 
          image: nginx:alpine 
          imagePullPolicy: IfNotPresent
          ports: 
            - containerPort: 80
          volumeMounts:
            - name: gluster-efk-volume
              mountPath: "/usr/share/nginx/html"
      volumes:
      - name: gluster-efk-volume
        glusterfs:
          endpoints: glusterfs-cluster
          path: efk-volume
          readOnly: False


```

