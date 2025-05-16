# 环境

# 容器运行时(所有节点)



docker、containerd、CRI-O 任选一个，根据个人喜好选择



## docker

```shell
# 更新yum
sudo yum -y update

# docker
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce

# 如果没有配置，则直接创建
sudo mkdir /etc/docker
vim /etc/docker/daemon.json
# 也可以加入 "root-data": "/home/docker_data" 修改docker存储目录
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}


sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```



## cri-docker

需要已安装docker

下载地址：[https://github.com/Mirantis/cri-dockerd/releases](https://github.com/Mirantis/cri-dockerd/releases)

下载文件：`cri-dockerd-<version>.<OS-version>.<ARCH>.rpm`

```shell
# 安装,centos 7，amd64，所以下载 cri-dockerd-0.3.1-3.el7.x86_64.rpm

rpm -ivh cri-dockerd-0.3.1-3.el7.x86_64.rpm

vim /usr/lib/systemd/system/cri-docker.service
# 指定imagePull地址
# 在ExecStart加入参数 --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.8
# 如果是子节点，则还需要加上 --network-plugin=cni
sudo systemctl enable cri-docker
sudo systemctl daemon-reload
sudo systemctl restart cri-docker
```



## containerd

[官方安装教程](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)

### 安装容器

下载地址：[https://github.com/containerd/containerd/releases](https://github.com/containerd/containerd/releases)

下载文件：`containerd-<VERSION>-<OS>-<ARCH>.tar.gz`

```shell
# 加载overlay和br_netfilter模块
modprobe overlay
modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sysctl --system

# 将下载好的文件解压到 /usr/local
tar Cxzvf /usr/local containerd-<version>-linux-amd64.tar.gz

# 如果使用systemd启动containerd
mkdir -p /usr/local/lib/systemd/system

curl -o /usr/local/lib/systemd/system/containerd.service \
https://raw.githubusercontent.com/containerd/containerd/main/containerd.service


mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
vim /etc/containerd/config.toml

# 1、找到[plugins."io.containerd.grpc.v1.cri"]
# 修改sandbox_image，将registry.k8s.io/pause:<version>修改为registry.aliyuncs.com/google_containers/pause:<version>

# 2、找到[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
# 将SystemdCgroup设置为true

# systemctl restart containerd 即可生效

systemctl daemon-reload
systemctl enable --now containerd
# ---------------------以下脚本可忽略----------------------
# 查看版本
ctr -v
```



### 安装 runc

下载地址：[https://github.com/opencontainers/runc/releases](https://github.com/opencontainers/runc/releases)

下载文件：`runc.amd64`

```shell
# 安装
install -m 755 runc.amd64 /usr/local/sbin/runc
```



### 安装CNI插件

下载地址：[https://github.com/containernetworking/plugins/releases](https://github.com/containernetworking/plugins/releases)

下载文件：`cni-plugins-<OS>-<ARCH>-<VERSION>.tgz`

```shell
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-<version>.tgz
```



## CRI-O

```shell
# 安装必要的软件包
yum install -y yum-utils device-mapper-persistent-data lvm2

# 添加 CRI-O 软件包源
yum-config-manager --add-repo https://cbs.centos.org/repos/paas7-crio-311-candidate/x86_64/os/

yum install -y cri-o

systemctl enable --now crio
```



# 防火墙(所有节点)

<font color='red'>云服务器</font>一般在安全组进行配置

- 关闭防火墙(<font color='red'>不推荐</font>)

  ```shell
  systemctl stop firewalld
  systemctl disable firewalld
  ```

- 配置端口(推荐)

  [官方文档](https://kubernetes.io/docs/reference/ports-and-protocols/)

  - master node(主节点)
  
    ```shell
    # 开启端口
    
    # Kubernetes API server
    sudo firewall-cmd --permanent --add-port=6443/tcp
    
    # etcd server client API
    sudo firewall-cmd --permanent --add-port=2379-2380/tcp
    
    # Kubelet API
    sudo firewall-cmd --permanent --add-port=10250/tcp
    
    # kube-scheduler
    sudo firewall-cmd --permanent --add-port=10259/tcp
    
    # kube-controller-manager
    sudo firewall-cmd --permanent --add-port=10257/tcp
    
    # 重新加载防火墙配置
    sudo firewall-cmd --reload
    
    # ---------------------以下脚本可忽略----------------------
    
    # 如果需要master作为worker节点
    sudo firewall-cmd --permanent --add-port=30000-32767/tcp
    
    # 查看端口开放情况
    firewall-cmd --list-ports
    
    # 关闭端口
    sudo firewall-cmd --permanent --remove-port={6443,2379-2380,10250-10252,10255}/tcp 
    ```
  
  
  
  - worker node(子节点)
  
      ```shell
      # Kubelet API
      sudo firewall-cmd --permanent --add-port=10250/tcp
      
      sudo firewall-cmd --reload
      
      # ---------------------以下脚本可忽略----------------------
  
      # 暴露nodePorts
      sudo firewall-cmd --permanent --add-port=30000-32767/tcp
      sudo firewall-cmd --reload
      ```
  
  
  ​    


# 配置host(所有节点)

```shell
# 设置主机名 master-node、worker-node-1、worker-node-2

# master
sudo hostnamectl set-hostname master-node
# node
sudo hostnamectl set-hostname worker-node-编号

```



# 其它配置(所有节点)

```shell
# 关闭SElinux，可以让容器顺利地读取主机文件系统
# 临时关闭，可以不用重启机器
sudo setenforce 0
# 永久关闭
sed -i 's/enforcing/disabled/' /etc/selinux/config
# sudo sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux

# 关闭swap，Swap是操作系统在内存吃紧的情况申请的虚拟内存，按照Kubernetes官网的说法，Swap会对Kubernetes的性能造成影响，不推荐使用Swap。
# 临时关闭，可以不用重启机器
swapoff -a
# 永久关闭
sed -i '/swap/d' /etc/fstab

# 更新 iptables 设置，确保了数据包在过滤和端口转发期间被 IP 表正确处理。将桥接的IPv4流量传递到iptables的链
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system

```



# kubernetes(所有节点)

[官方文档](https://v1-24.docs.kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/)：自 1.24 版起，Dockershim 已从 Kubernetes 项目中移除。阅读 [Dockershim 移除的常见问题](https://v1-24.docs.kubernetes.io/zh-cn/dockershim)了解更多详情。



```shell
# 添加yum源
sudo cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 安装最新版本
# 注意！！！如果容器运行时是docker，则需要先安装cri-docker
yum install -y kubelet kubeadm kubectl

# kubelet开机自启
systemctl enable kubelet

# ------------以下脚本可忽略---------------

# 安装 1.23.0，如果是docker并且没有安装cri-docker，则只能安装1.24之前的版本
yum install -y kubelet-1.23.0 kubeadm-1.23.0 kubectl-1.23.0

# 如果kubelet status有问题，可以查看日志
journalctl -xefu kubelet
```



# 初始化(master节点)

```shell
# 查看网络配置
curl https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
# 查看 kube-flannel.yml 配置下的 net-conf.json下的Network 
# 这里的Network为 10.244.0.0/16
# 对应 kubeadm init 中的 pod-network-cidr 参数

# --image-repository registry.aliyuncs.com/google_containers 一定要指定，否则需要翻墙
# --kubernetes-version v1.24.0
# --apiserver-advertise-address=172.16.12.210 注意要写成master的ip
# 注意！！！如果报错： Found multiple CRI endpoints on the host. Please define which one do you wish to use by setting the 'criSocket' field in the kubeadm configuration file:unix:///<xxxx>.sock, unix:///<zzzz>.sock
# 需要添加：--cri-socket=<socket链接> 进行指定，如 --cri-socket=unix:///<xxxx>.sock


# 需要等待比较长时间
kubeadm init \
--apiserver-advertise-address=<master节点ip> \
--image-repository registry.aliyuncs.com/google_containers \
--service-cidr=10.20.0.0/16 \
--pod-network-cidr=10.244.0.0/16 \
--ignore-preflight-errors=all


# 初始化成功后需要 认真阅读、认真阅读、认真阅读 控制台的输出，以下脚本以控制台输出为准！！！

# root用户
# 临时，控制台会输出改命令
export KUBECONFIG=/etc/kubernetes/admin.conf
# 永久，追加到/etc/profile
source /etc/profile

# 普通用户 拷贝k8s认证文件，命令在初始化成功后会打印到控制台，如下所示：
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


# cri-dockerd 注意！！！
# 集群创建好之后需要设置cri-dockerd网络插件，如果在初始化之前加上改参数，则无法下拉集群镜像！！！
# 在ExecStart加入参数 --network-plugin=cni
vim /usr/lib/systemd/system/cri-docker.service

sudo systemctl daemon-reload
sudo systemctl restart cri-docker
```



# flannel（master节点）

## master节点

```shell
# 安装网络插件 flannel
# 多网卡参考 https://medium.com/@anilkreddyr/kubernetes-with-flannel-understanding-the-networking-part-1-7e1fe51820e4 进行修改
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

## 所有节点

```shell
# flannel 所有节点 需要开通 udp 端口 8285 和 8472 和 ip伪装

# Flannel，参考官方文档 https://github.com/flannel-io/flannel/blob/master/Documentation/troubleshooting.md#firewalls

# Flannel
firewall-cmd --permanent --add-port=8285/udp 

# Flannel
firewall-cmd --permanent --add-port=8472/udp

# Flannel，开启伪装IP，可以进行端口转发到 另一个 ip:port
firewall-cmd --permanent --add-masquerade

# 重新加载防火墙配置
sudo firewall-cmd --reload

```



# master参与调度

kubernetes master节点一般不参与部署pod，但是有的时候只有一台主机，后续可能再增加，则需要将master节点设为调度节点，[参考资料](https://www.jianshu.com/p/ecdf843b120c)，注意：需要等到节点的状态为 Ready

```shell
# 查看node
kubectl get node

# 查看node详情
kubectl describe node <master节点名>

# 可以看到输出,找到Taints属性
Taints:		<key>:NoSchedule

# NoSchedule 表示不参与调度 - 表示删除该值
kubectl taint node <master节点名> <key>:NoSchedule-
```



# worker-node加入集群(worker节点)

```shell
# 如果忘记保存join语句，在master节点下查看join语句
kubeadm token create --print-join-command

# 在worker-node下执行join语句
# 注意！！！ 如果是cri-dockerd，则有多个criSocket，需要用 --cri-socket=<socket链接> 进行指定
kubeadm join xxxx

# 添加节点角色(roles)，master执行
kubectl label node <节点名> node-role.kubernetes.io/worker=worker

# ------------以下脚本可忽略---------------

# master节点查看node状态
# 查看节点，最开始时都是NotReady，需要等待一段时间
kubectl get nodes
# 查看kube-system pod状态
kubectl get pod -n kube-system

# 如果一直是Pending，查看pod详情，最底下有原因
kubectl describe pod -n kube-system pod名
# 通过原因在Google查询解决方法

# 解决kubectl get nodes NotReady的问题,也可能是没有安装网络插件
export kubever=$(kubectl version | base64 | tr -d '\n')
kubectl apply -f https://cloud.weave.works/k8s/net?k8s-version=$kubever
```





# Ingress Controller

[GitHub地址](https://github.com/kubernetes/ingress-nginx)

[安装地址](https://github.com/kubernetes/ingress-nginx/blob/main/docs/deploy/index.md)

- 在安装地址中找到 yaml 文件地址，下载到本地（里面的image必需翻墙才能下载，所以需要改为能下载的地址）

  如：https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.0/deploy/static/provider/cloud/deploy.yaml

- 打开文件、搜索 `image:` 

- 在 https://hub.docker.com/u/anjia0532 下面找到对应的 image （版本也需要注意）

- 将找到的image地址替换掉yaml文件中的image地址

- 固定NodePort，找到**service** -> **ingress-nginx-controller**，固定端口后方便后续配置nginx

  ```yaml
  ports:
    - appProtocol: http
      name: http
      port: 80
      protocol: TCP
      targetPort: http
      nodePort: 31080 # 新增
    - appProtocol: https
      name: https
      port: 443
      protocol: TCP
      targetPort: https
      nodePort: 31443 # 新增
  ```

- 将修改好的yaml文件传到master：`kubectl apply -f deploy.yaml`



# metrics-server

[指标服务](https://github.com/kubernetes-sigs/metrics-server)

下载yaml [https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml](https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml)

修改

```yaml
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls # 新增，否则无法访问 apiserver 10250端口
        
        image: bitnami/metrics-server:0.6.1 # 修改为能下载的镜像
```

重命名为 **metrics-server.yaml**

`kubectl apply -f metrics-server.yaml`

安装完毕之后测试

```shell
# 查看各个节点cup和内存使用情况
kubectl top node

# 查看pod的cup和内存使用情况
kubectl top pod
```



# 下面的都可以忽略





# NFS文件服务器

## NFS Master节点

找一个磁盘比较大的机器安装

```shell
# 安装nfs工具
yum install -y nfs-utils

# 创建nfs目录
mkdir -p /nfs/data

# 暴露 /nfs/data/目录  *表示任何人 括号内的表示有哪些操作
echo "/nfs/data/ *(insecure,rw,sync,no_root_squash)" > /etc/exports

# 配置生效
exportfs -r

# 开机自启 rpcbind  --now 表示现在启动
systemctl enable rpcbind --now

# 开机自启 nfs-server
systemctl enable nfs-server --now

# 防火墙配置
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --permanent --add-service=mountd
firewall-cmd --permanent --add-port=2049/tcp
firewall-cmd --permanent --add-port=2049/udp
firewall-cmd --reload

```



## NFS Node节点（PV挂载请忽略）

<font color='red'>重启后需要重新连接master，暂时没解决</font>

```shell
# 安装nfs工具
yum install -y nfs-utils

# 创建nfs目录
mkdir -p /nfs/data

# 开机自启 rpcbind  --now 表示现在启动
systemctl enable rpcbind --now

# 开机自启 nfs-server
systemctl enable nfs-server --now

# 查看远程服务器有哪些目录可以挂载
showmount -e <masterIp>

# 连接nfs远程服务器
mount -t nfs <masterIp>:/nfs/data /nfs/data

# 写入一个测试文件，然后在 nfs 服务器节点查看是否同步
echo "hello word" > /nfs/data/test.txt
```



# 测试kubernetes

## NameSpace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test
```

`kubectl create ns test`

## StorageClass & PV & PVC

>StorageClass：存储类（**StorageClass**），为管理员提供了描述存储 "类" 的方法
>
>PV：持久卷（**Persistent Volume**），将应用需要持久化的数据保存到指定位置
>
>PVC：持久卷申明（**Persistent Volume Claim**），申明需要使用的持久卷规格



### StorageClass

[官方文档](https://kubernetes.io/zh-cn/docs/concepts/storage/storage-classes/)

```yaml
# StorageClass属于全局资源，不用namespace
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs
provisioner: example.com/external-nfs
parameters:
  server: 172.16.12.210
  path: /nfs/data
```

### PV

[官方文档](https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-pv
spec:
  capacity:
    storage: 10M
  accessModes:
    - ReadWriteMany
  storageClassName: nfs
  nfs:
    path: /nfs/data/test-pv
    server: 172.16.12.210
```

### PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  namespace: test
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Mi
  storageClassName: nfs
```



## Deployment

[官方文档](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx-deployment
  namespace: test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1
        name: nginx
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          persistentVolumeClaim:
            claimName: test-pvc
```



```shell
# 查看pod
kubectl -n test get pod -o wide

# 在任意节点,nginx端口为80，可以忽略端口
curl podIp
```



## Service

[官方文档](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/)

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: nginx-service
  namespace: test
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
```



测试，有进行挂载不方便测试

```shell
# 查看service
kubectl -n test get svc

# 任意节点
curl svc的CLUSTER-IP

# 为了让负载均衡能够明显的看出来
kubectl -n test get pod

# 进入pod，跟docker进入容器基本一致
kubectl -n test exec -it pod名 /bin/bash

# 进入容器后修改nginx html.index的内容
echo 1111 > /usr/share/nginx/html/index.html

# 其它的 nginx-pod 进行同样的操作，需要对内容进行修改

# 然后再进行访问，多访问几次就会发现有进行负载均衡
curl svc的CLUSTER-IP
```



## Ingress

[官方文档](https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress/)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: test
spec:
  ingressClassName: nginx
  rules:
    - host: nginx.demo.cc
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: 
                name: nginx-service
                port: 
                  number: 8080
```

hosts文件

```shell
vim /etc/hosts

172.16.12.210	nginx.demo.cc
```

ingress controller

```shell
# 找到 ingress-nginx-controller 对应的端口
kubectl get svc -A

# 如 80:31342/TCP,443:31233/TCP

curl nginx.demo.cc:31342
```



## Secret

[从私有仓库拉取镜像](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/pull-image-private-registry/)

创建docker登录Secret

1、登录docker，docker login

2、cat ~/.docker/config.json

3、将json内容转为base64字符串

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: demo-docker-secret
  namespace: dev
data:
  .dockerconfigjson: json的base64字符串
type: kubernetes.io/dockerconfigjson
```





# 安装dashboard(master节点)

在[kubernetes/dashboard releases](https://github.com/kubernetes/dashboard/releases)找到对应的<font color='red'>版本</font>安装命令中的yaml地址

## 生成证书

如果不需要<font color='red'>https</font>，否则请跳过

```shell
domain='dashboard.test.cc'
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout dashboard.key -out dashboard.crt \
-subj "/CN=${domain}/O=${domain}"
```

## Namespace

```shell
kubectl create ns kubernetes-dashboard
```



## Secret(tls)

- 命令创建(推荐)

  ```shell
  kubectl create secret tls kubernetes-dashboard-certs --key dashboard.key --cert dashboard.crt -n kubernetes-dashboard
  ```

- yaml创建(不推荐)

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: kubernetes-dashboard-certs
    namespace: kubernetes-dashboard
  type: kubernetes.io/tls
  data:
    tls.crt: dashboard.crt文件内容转为base64
    tls.key: dashboard.key文件内容转为base64
  ```



## 安装dashboard

- 修改yaml文件
  - **deployment** -> **kubernetes-dashboard** -> **args **追加

    ```shell
    - --tls-cert-file=dashboard.crt # 证书
    - --tls-key-file=dashboard.key # 私钥
    - --token-ttl=7200 # token超时时间
    ```

  - 注释 **Secret** -> **kubernetes-dashboard-certs**

    ```yaml
    #---
    
    #apiVersion: v1
    #kind: Secret
    #metadata:
    #  labels:
    #    k8s-app: kubernetes-dashboard
    #  name: kubernetes-dashboard-certs
    #  namespace: kubernetes-dashboard
    #type: Opaque
    ```

- 应用：`kubectl apply -f dashboard.yaml`



## Ingress

- yaml: `vim dashboard-ingress.yaml`

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    namespace: kubernetes-dashboard
    name: dashboard-ingress
    annotations:
      kubernetes.io/ingress.class: "nginx"
      nginx.ingress.kubernetes.io/use-regex: "true"
      nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
      nginx.ingress.kubernetes.io/rewrite-target: /
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
      nginx.org/ssl-services: "kubernetes-dashboard"
  spec:
    tls:
      - hosts: 
        - dashboard.test.cc
        secretName: kubernetes-dashboard-certs
    rules:
    - host: dashboard.test.cc
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: 
                 name: kubernetes-dashboard
                 port: 
                   number: 443
  ```

- 应用：`kubectl apply -f dashboard-ingress.yaml`

## Nginx

- 查看ingress-nginx-controller 443 对应的端口：`kubectl get svc -n ingress-nginx`

- 添加端口监听

  ```nginx
  server {
      	# 443 为https默认端口
  		listen       443 ssl http2;
      	# 有域名给域名，没域名给ip
          server_name  172.16.12.210;
  		ssl_certificate /home/demo/dashboard.crt;
          ssl_certificate_key /home/demo/dashboard.key;
  		
  		location / {
              proxy_pass https://dashboard.demo.cc:32702;
          }
      }
  ```

- 刷新nginx配置：`nginx -s reload`

- 防火墙

  ```shell
  # 查看443端口是否开放
  firewall-cmd --list-ports
  
  # 如果未开放，需要将443端口开放
  sudo firewall-cmd --permanent --add-port=443/tcp
  
  # 刷新防火墙配置
  sudo firewall-cmd --reload
  ```



## 创建admin账号

- 命令创建（推荐）

  ```shell
  # 创建账号,放在kubernetes-dashboard下方便与kubernetes-dashboard一起删掉
  kubectl create serviceaccount admin-user -n kubernetes-dashboard
  
  # 绑定角色
  kubectl create clusterrolebinding admin-user-binding \
    --clusterrole=cluster-admin \
    --serviceaccount=kubernetes-dashboard:admin-user
  ```

- yaml创建

  `vim dashboard-admin-user.yaml`

  ```
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: admin-user
    namespace: kubernetes-dashboard
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: admin-user-binding
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: cluster-admin
  subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
  ```

  

## 获取登录Token

```shell
kubectl -n kubernetes-dashboard get secret \
$(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") \
-o go-template="{{.data.token | base64decode}}"
```



## 使用账号密码登录(<font color='red'>测试</font>)

### 备份数据

```shell
# 将kube-apiserver.yaml备份到当前目录，不要备份到/etc/kubernetes/manifests目录下，否则会导致配置不生效
cp /etc/kubernetes/manifests/kube-apiserver.yaml  .
```



### 创建账号密码

```yaml
# id不可重复,username为已经创建的用户名
echo "<password>,<username>,<id>" > /etc/kubernetes/basic_auth.csv

```



### 修改apiserver.yaml

```shell
vim /etc/kubernetes/manifests/kube-apiserver.yaml
# spec -> containers -> command 追加

- --token-auth-file=/etc/kubernetes/pki/basic_auth_file
- --authorization-policy-file=/etc/kubernetes/pki/auth_policy_file
# 监听kube-system容器变化
watch crictl ps

# 查看是否生效
ps aux | grep kube-apiserver | grep token-auth-file
```



### 修改dashboard容器启动参数

**deployment** -> **kubernetes-dashboard** -> **args **追加

```
- --authentication-mode=basic
```





## 参考

- ingress-nginx暴露dashboard [https://blog.csdn.net/hans99812345/article/details/124834841](https://blog.csdn.net/hans99812345/article/details/124834841)
- dashboard 官方GitHub [https://github.com/kubernetes/dashboard](https://github.com/kubernetes/dashboard)
- nginx转发https [https://www.cnblogs.com/CGCong/p/16326926.html](https://www.cnblogs.com/CGCong/p/16326926.html)



# 常用命令

```shell
# 查看kubeadm连接命令
kubeadm token create --print-join-command

# 查看当前支持的所有apiVersion
kubectl api-versions

# 节点日志
journalctl -u kubelet

# 添加属性 label、taint、annotation等，这里示例为label
kubectl label <类型名/缩写> <对象名> <label_name>=<label_value>

# 修改属性 label、taint、annotation等，这里示例为label
kubectl label <类型名/缩写> <对象名> <label_name>=<label_value> --overwrite

# 删除属性 label、taint、annotation等，这里示例为label，在label_name后面加上减号
kubectl label <类型名/缩写> <对象名> <label_name>-

# 节点退出集群
kubeadm reset

# 创建或更新对象
kubectl apply -f <清单文件>

# 删除对象
kubectl delete -f <清单文件>

# 查询对象 
kubectl get <类型名/缩写>

# 删除对象
kubectl delete <类型名/缩写> <对象名>

# 对象详情
kubectl describe <类型名/缩写> <对象名>

# 编辑对象
kubectl edit <类型名/缩写> <对象名>

# 生命都不加，表示默认namespace：default

# 某个命名空间的对象
-n <namespace>
# 所有命名空间的对象
-A 

# 查询对象列表更多的字段
-o wide

# 查询对象对应的yaml配置,如：kubectl get <kindName> <objectName> -o yaml
-o yaml

# 查看pod日志，跟docker差不多
kubectl logs -f <podName>

# 进入pod，跟docker差不多
kubectl exec -it <podName> /bin/bash

# 创建configMap
kubectl create configmap <configMapName> --from-file=文件名

# 清单文件中使用变量,如 tag=12，在yaml中使用 $tag或${tag} 即可引入变量,变量需要 export
envsubst < <清单文件> | kubectl apply -f -

# 重启deployment
kubectl rollout restart deploy <deployment-name> -n <namespace>

# 查看 ingress-controller nginx.conf 配置，先kubectl get pod -n ingress-nginx查看ingress-nginx-controller pod的名字
kubectl exec -it ingress-nginx-controller-xxxx -n ingress-nginx --  cat /etc/nginx/nginx.conf

# 监听kube-system下容器的变化
watch crictl ps

# 强制删除
--force --grace-period=0

# 扩缩
kubectl scale deploy/<deployment-name> --replicas=数量

```



# 常见问题与解决

## docker重启后k8s连接不上，连接6443失败

```shell
# 设置docker驱动
vim /etc/docker/daemon.json
"exec-opts": ["native.cgroupdriver=systemd"]

# 在KUBELET_KUBECONFIG_ARGS 后面追加 --cgroup-driver=systemd
vim /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
#如: Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --cgroup-driver=systemd"

# KUBELET_KUBEADM_ARGS 添加 --cgroup-driver=systemd
vim /var/lib/kubelet/kubeadm-flags.env
# 如: KUBELET_KUBEADM_ARGS="--cgroup-driver=systemd --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.6"

# 重启 kubelet
systemctl daemon-reload
systemctl restart kubelet

# 查看ssl证书创建和过期时间
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates

# 更新ssl证书
sudo kubeadm certs renew all
```





























