# 使用K8S部署Selenium Grid集群

## 前言

[speedy](https://gitlab.com/AngryTester/speedy)是基于Selenium Grid+Docker的自动化测试框架，之前一直是将Grid集群中在公司的私有云平台上。虽然之前也用docker-compose搭建过Grid集群，但是并没有考虑多节点横向扩展的能力，而这恰恰就是K8S的强项。正好最近有一个微服务项目应用到K8S，借此机会学习了一把K8S的搭建和简单使用，正好拿Grid集群的搭建作为例子，在此做个记录。
参考： https://www.bladewan.com/2018/01/02/kubernetes_install/
https://juejin.cn/post/6844904072240168973  
## 搭建

### > 1.环境准备

通过VirtualBox准备了三台centos，分别为：
```
192.168.5.101-master1

192.168.5.102-minion2

192.168.5.105-master
```
虚拟机镜像选择`CentOS-7-x86_64-Minimal-1804.iso`


### 1.1虚拟机启动之后更新yum源

分别执行：

```
yum update -y
```

### 1.2配置三台服务器的Host

分别执行：

```
echo -e "192.168.5.140 minion1\n\
192.168.5.106 minion2\n\
192.168.5.191 master" >> /etc/hosts

```

#### 1.3.关闭防火墙和SELINUX

分别执行：
```
systemctl stop firewalld
systemctl disable firewalld

sed -ri 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

或者 将 SELinux 设置为 permissive 模式（相当于将其禁用）
setenforce 0

```

### 1.4.修改网桥设置

```
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.d/k8s.conf

echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.d/k8s.conf

echo "vm.swappiness=0" >> /etc/sysctl.d/k8s.conf
sysctl -p /etc/sysctl.d/k8s.conf
```
出现No such file or directory
```
 centos7添加bridge-nf-call-ip6tables出现No such file or directory

    解决方法：

    [root@localhost ~]# modprobe br_netfilter
    [root@localhost ~]# ls /proc/sys/net/bridge
    bridge-nf-call-arptables bridge-nf-filter-pppoe-tagged
    bridge-nf-call-ip6tables bridge-nf-filter-vlan-tagged
    bridge-nf-call-iptables bridge-nf-pass-vlan-input-dev
 
```

### 1.5.关闭swap

分别执行：

```
sed -i "s/^.*swap.*/#&/g" /etc/fstab
```


### 1.6.配置master和minion的免密互联

分别执行：
```
ssh-keygen -t rsa
```
一路回车。

分别执行：
```
cat ~/.ssh/id_rsa.pub > ~/.ssh/authorized_keys
```
master上执行：
```
ssh-copy-id minion1&&ssh-copy-id minion2&&ssh-copy-id 
```

minion上分别执行：
```
ssh-copy-id master
```
### 1.7 时钟同步

```
# 由于是最小安装系统，需要单独安装 ntpdate
yum install -y ntpdate

# 使用阿里的时钟源，每隔一小时同步一次
crontab -e
# 将打开编辑，写入  
 0 */1 * * * ntpdate time1.aliyun.com
# 验证
  crontab -l
0 */1 * * * ntpdate time1.aliyun.com

# 如果是首次，可以主动同步一次
 ntpdate time1.aliyun.com
22 Feb 22:24:18 ntpdate[1659]: adjust time server 203.107.6.88 offset -0.005327 sec

# 验证多台主机时间是否一致
  date

2021年 01月 19日 星期二 23:23:50 CST
您在 /var/spool/mail/root 中有新邮件

```
### 1.8.1 添加网桥过滤 
在kubernetes集群中添加网桥过滤，为了实现内核的过滤。

### 添加网桥过滤及地址转发
```
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
EOF

加载 br_netfilter 模块
modprobe br_netfilter

查看是否加载
[root@master ~]# modprobe br_netfilter
[root@master ~]# lsmod | grep br_netfilter
br_netfilter           22256  0 
bridge                151336  1 br_netfilter

加载网桥过滤配置文件

[root@master ~]# sysctl -p /etc/sysctl.d/k8s.conf

net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
```


### 1.8.2 开启 IPVS  

由于Kubernets在使用Service的过程中需要用到iptables或者是ipvs，ipvs整体上比iptables的转发效率要高，因此这里我们直接部署ipvs就可以了。  
 **安装 ipset 及 ipvsadm **  
```
yum install -y ipset ipvsadm
```
添加需要加载的模块  
```
cat > /etc/sysconfig/modules/ipvs.modules << EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
```

### 授权、运行、检查是否加载
```
[root@master01 ~]# chmod +x /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
nf_conntrack_ipv4      15053  0
nf_defrag_ipv4         12729  1 nf_conntrack_ipv4
ip_vs_sh               12688  0
ip_vs_wrr              12697  0
ip_vs_rr               12600  0
ip_vs                 145497  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          139224  2 ip_vs,nf_conntrack_ipv4
libcrc32c              12644  3 xfs,ip_vs,nf_conntrack

```

重启服务器。

## 2. docker 

 ## 安装docker version:  18.09.9



 参考[docker 安装官网](https://docs.docker.com/engine/install/centos/)  

查寻版本
```
yum list docker-ce-cli --showduplicates | sort -r
```

###  2.1  指定版安装, 分别执行：：

```
yum install docker-ce-3:18.09.9-3.el7  docker-ce-cli-1:18.09.9-3.el7 containerd.io -y

```


Docker从1.13版本开始调整了默认的防火墙规则，禁用了iptables filter表中FOWARD链，这样会引起Kubernetes集群中跨Node的Pod无法通信，在各个Docker节点执行下面的命令：

###  2.2  配置docker的策略

```
vi /usr/lib/systemd/system/docker.service
#ExecStart=上面加一行：
ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT
```

###  2.3 修改docker 配置文件 [参考](https://juejin.cn/post/6844904072240168973#heading-18 )

```
在 /etc/docker/daemon.json 写入如下内容
cat > /etc/docker/daemon.json << EOF
 { "exec-opts": ["native.cgroupdriver=systemd"]}
EOF

```

### 2.4 启动docker：
 ```
systemctl enable docker
systemctl start docker
 ```

## 3.搭建本地镜像仓库

由于后续需要拉取镜像，避免直接从公网拉取，在master节点上搭建本地镜像仓库（也可搭建在其他局域网服务器上），

### 3.1 启动master节点的docker服务：
```
systemctl enable docker
systemctl start docker
```

### 3.2 启动一个registry容器：
```
docker run -d -p 5000:5000 --restart=always registry 
```

### 3.3 分别修改三台服务器的docker启动配置后重启docker：

```
echo {\"insecure-registries\": [\"172.26.X.60:5000\"]} > /etc/docker/daemon.json
systemctl daemon-reload
systemctl restart docker
```

####  另一种方法：
```
vi  /usr/lib/systemd/system/docker.service
将  --insecure-registry=192.168.5.192  加到

例如  
ExecStart=/usr/bin/dockerd -H fd:// --insecure-registry=192.168.5.192  --containerd=/run/containerd/containerd.sock
````

## 4. kus version 1.17.3  
有三个软件需要安装：

kubeadm: 初始化集群，管理集群等  
kubelet: 用于接收 api-server 指令，对 Pod 生命周期进行管理  
Kubectl: 集群命令管理工具  

我这里三个软件的版本均为：1.17.3-0
接下来分别在三台主机上安装这三个软件。



### 4.1. 导入kubeadm、kubelet、kubectl

### 配置 k8s 源： 

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
 http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

```
安装：
```

yum list kubeadm.x86_64 --showduplicates | sort -r

yum install -y kubeadm-1.17.3-0 kubelet-1.17.3-0 kubectl-1.17.3-0    --disableexcludes=kubernetes


```
docker 的 cgroup driver 做了修改，这里 kubelet 需要保持一致，需修改如下配置内容：
```
[root@master01 ~]# cat /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"

```

### 4.2 相关镜像 [参考](https://juejin.cn/post/6844904072240168973)

    需要准备镜像如下：
    ```
    k8s.gcr.io/kube-apiserver:v1.17.3
    k8s.gcr.io/kube-controller-manager:v1.17.3
    k8s.gcr.io/kube-scheduler:v1.17.3
    k8s.gcr.io/etcd:3.4.3-0
    k8s.gcr.io/coredns:1.6.5
    k8s.gcr.io/kube-proxy:v1.17.3
    k8s.gcr.io/pause:3.1

    ```

    master 上安装  
    在镜像文件目录新建文件`images—master.txt`,文件内容如下:

    ```
    k8s.gcr.io/kube-apiserver:v1.17.3
    k8s.gcr.io/kube-controller-manager:v1.17.3
    k8s.gcr.io/kube-scheduler:v1.17.3
    k8s.gcr.io/etcd:3.4.3-0
    k8s.gcr.io/coredns:1.6.5
    k8s.gcr.io/kube-proxy:v1.17.3

    ```

    在镜像文件目录新建文件`images-work.txt`,文件内容如下:

    ```
    k8s.gcr.io/kube-proxy:1.17.3
    k8s.gcr.io/pause
    ```

    在镜像文件目录执行：

    ```
    for i in `cat images-master(work).txt ` ; do docker pull  < `echo $i` ; done

    ```


###  4.3. 初始化集群

```
kubeadm init --kubernetes-version=1.17.3 --pod-network-cidr=10.244.0.0/16 --token-ttl=0
kubeadm init --kubernetes-version=1.17.3  --token-ttl=0
```

`--token-ttl=0`表示token永不过期

最后会返回类似如下：
```
kubeadm join --token 1ec757.1865c26203561363 172.26.X.60:6443 --discovery-token-ca-cert-hash sha256:004d45cc446b613a054451c680b1506145987b9b53c6290fa7040eb40e01ef7f
```

在两个minion上分别执行如上命令加入集群。

可通过`docker ps -a`查看master节点和minion节点上容器的启动情况：

master节点上应该启动5个pause容器以及`kube-proxy`,`kube-controller`,`kube-scheduler`,`kube-apiserver`和`etcd`。

minion节点上应该启动1个pause容器以及`kube-proxy`。

K8S上所有pod的启动都会对应启动一个pause容器，主要作用是为了使pod里面的容器共享pause容器的网络。


### 配置kubectl命令执行权限

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
如果需要在minion节点上执行kubectl命令，也需要将`admin.conf`复制到minion上。

通过如下命令获取pod信息：
```
kubectl  get pods --all-namespaces
kube-system   etcd-master                      1/1       Running   0          2s
kube-system   kube-apiserver-master            1/1       Running   0          2s
kube-system   kube-controller-manager-master   1/1       Running   0          2s
kube-system   kube-dns-6f4fd4bdf-x27zx         0/3       Pending   0          9m
kube-system   kube-proxy-kdkj2                 1/1       Running   0          9m
kube-system   kube-scheduler-master            1/1       Running   0          2s
```

### 将KUBECONFIG写入环境变量
```
echo KUBECONFIG=/etc/kubernetes/admin.conf >> ~/.bash_profile
source ~/.bash_profile
```

### 创建flannel

在未创建flannel之前，kube-dns状态为pending。

执行如下两句命令创建flannel：
```
kubectl apply -f kube-flannel-rbac.yaml  
kubectl apply -f  kube-flannel.yaml
```

`kube-flannel-rbac.yaml`文件[参考](https://raw.githubusercontent.com/AngryTester/K8S_Selenium_Grid/master/kube-flannel-rbac.yaml)

`kube-flannel.yaml`文件[参考](https://raw.githubusercontent.com/AngryTester/K8S_Selenium_Grid/master/kube-flannel.yml)

### 创建dashboard

执行如下命令创建dashboard：
```
kubectl apply -f kubernetes-dashboard.yaml
```
`kubernetes-dashboard.yaml`文件[参考](https://raw.githubusercontent.com/AngryTester/K8S_Selenium_Grid/master/kubernetes-dashboard.yaml)

### 创建dashboard认证

```
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```
增加如下内容：
```
- --basic-auth-file=/etc/kubernetes/pki/basic_auth_file
```

`/etc/kubernetes/pki/basic_auth_file`文件内容如下：
```
admin,admin,2
```

重启kubelet：
```
systemctl daemon-reload
systemctl restart kubelet
```

然后创建账号和角色的关联
```
kubectl create clusterrolebinding login-on-dashboard-with-cluster-admin --clusterrole=cluster-admin --user=admin
```

访问 https://172.26.X.60:30000 ,输入用户名`admin`和密码`admin`即可登录dashboard。

另外一种方式token，首先需要将`kubernetes-dashboard.yaml`中验证方式由`basic`修改为token。

token 可以通过相关指令获取：
```
kubectl -n kube-system describe $(kubectl -n kube-system get secret -n kube-system -o name | grep namespace) | grep token
```

 不同类型的token关联的权限是不一样的
 ```
# 输入下面命令查询kube-system命名空间下的所有secret，然后获取token。只要是type为service-account-token的secret的token都可以使用
kubectl get secret -n kube-system
# 比如我们获取replicaset-controller-token-wsv4v的touken
kubectl -n kube-system describe replicaset-controller-token-wsv4v
```

### 创建selenium-hub

`selenium.namespace.yml`内容[参考](https://raw.githubusercontent.com/Polygon-io/selenium-on-kubernetes/master/selenium.namespace.yml)
`selenium-hub.deployment.yml`内容[参考](https://raw.githubusercontent.com/Polygon-io/selenium-on-kubernetes/master/selenium-hub.deployment.yml)
`selenium-hub.service.yml`内容[参考](https://raw.githubusercontent.com/Polygon-io/selenium-on-kubernetes/master/selenium-hub.service.yml)

```
kubectl apply -f selenium.namespace.yml
kubectl apply -f selenium-hub.deployment.yml
kubectl apply -f selenium-hub.service.yml
```

### 创建selenium-node

`selenium-chrome.deployment.yml`内容[参考](https://raw.githubusercontent.com/Polygon-io/selenium-on-kubernetes/master/selenium-chrome.deployment.yml)

```
kubectl apply -f selenium-chrome.deployment.yml
```

扩展node可参考：

```
kubectl scale --replicas=3 -f selenium-chrome.deployment.yml
```

访问http://172.26.X.60:30001/grid/console  即能看到Grid集群情况。







