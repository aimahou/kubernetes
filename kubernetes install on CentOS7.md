

1.1概述
1.1.1架构
k8s-master  安装etcd，kubernetes-server/client
k8s-node1   安装docker,kubernetes-node/client，flannel
k8s-node2   安装docker,kubernetes-node/client，flannel

本文选择二进制包版本安装最新版测试
1.1.2下载k8s 1.5.2
https://dl.k8s.io/v1.5.2/kubernetes-server-linux-amd64.tar.gz
会产生11个二进制程序hyperkube  kubectl   kubelet  kube-scheduler  kubeadm  kube-controller-manager  kube-discovery  kube-proxy  kube-apiserver kube-dns  kubefed

1.1.3下载k8s 1.5.2客户端即NODE节点上部署的包    
https://dl.k8s.io/v1.5.2/kubernetes-client-linux-amd64.tar.gz
会产生两个二进制程序kube-proxy  kubefed

1.1.4下载etcd 3.1.10
https://github.com/coreos/etcd/releases/download/v3.1.0/etcd-v3.1.0-linux-amd64.tar.gz

1.1.5下载docker 1.13.1
https://get.docker.com/builds/Linux/x86_64/docker-1.13.1.tgz

1.1.6下载flannel
https://github.com/coreos/flannel/releases/download/v0.7.0/flannel-v0.7.0-linux-amd64.tar.gz

使用WinSCP软件将下载的所有包上传至一台虚拟机上，再用SCP传到其它两台虚拟机上
可以等“1.2.6配置多台SSH密钥共享”配置后再使用SCP。
[root@k8s-master rpms]# scp -P 22 /home/rpms/kubernetes-server-linux-amd64.tar.gz root@172.16.160.207:/home/rpms/kubernetes-server-linux-amd64.tar.gz 
kubernetes-server-linux-amd64.tar.gz                                                                          100%  300MB 149.8MB/s   00:02    
[root@k8s-master rpms]# scp -P 22 /home/rpms/kubernetes-server-linux-amd64.tar.gz root@172.16.160.206:/home/rpms/kubernetes-server-linux-amd64.tar.gz 
kubernetes-server-linux-amd64.tar.gz                                                                          100%  300MB 299.7MB/s   00:01
[root@k8s-master rpms]# scp -P 22 /home/rpms/kubernetes-client-linux-amd64.tar.gz root@172.16.160.207:/home/rpms/kubernetes-client-linux-amd64.tar.gz 
kubernetes-client-linux-amd64.tar.gz                                                                          100%   22MB  22.2MB/s   00:00    
[root@k8s-master rpms]# scp -P 22 /home/rpms/kubernetes-client-linux-amd64.tar.gz root@172.16.160.206:/home/rpms/kubernetes-client-linux-amd64.tar.gz 
kubernetes-client-linux-amd64.tar.gz                                                                          100%   22MB  22.2MB/s   00:00    
[root@k8s-master rpms]# 
1.2环境准备：

1.2.1系统“基础网络设备”模式安装,然后运行yum update,将所有包升级到最新版本

1.2.2域名解析配置
[root@k8s-master rpms]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

172.16.160.205  k8s-master
172.16.160.206  k8s-node1
172.16.160.207  k8s-node2

1.2.3校对时间
[root@k8s-master ~]# ntpdate ntp1.aliyun.com &&hwclock -w

1.2.4关闭selinux及防火墙
[root@k8s-master ~]# sed -i s'/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
[root@k8s-master ~]# systemctl disable firewalld; systemctl stop firewalld

1.2.5重启服务器

1.2.6配置多台SSH密钥共享
未配置多台ssh前，三台虚拟机之间使用SCP命令互传文件是需要对方密码的。

k8s-master
[root@k8s-master ~]# ssh-keygen -b 1024 -t rsa
一路回车
[root@k8s-master ~]# cd /root/.ssh
将本机公钥复制到其它两台虚拟机上并改名
[root@k8s-master ~]# scp -P 22 /root/.ssh/id_rsa.pub root@172.16.160.206:/root/.ssh/authorized_keys
[root@k8s-master ~]# scp -P 22 /root/.ssh/id_rsa.pub root@172.16.160.207:/root/.ssh/authorized_keys

k8s-node1
[root@k8s-master ~]# ssh-keygen -b 1024 -t rsa
一路回车
[root@k8s-master ~]# cd /root/.ssh
将本机公钥复制到其它两台虚拟机上并改名
[root@k8s-master ~]# ssh-copy-id -i  ~/.ssh/id_rsa.pub root@172.16.160.205
[root@k8s-master ~]# ssh-copy-id -i  ~/.ssh/id_rsa.pub root@172.16.160.207

k8s-node2
[root@k8s-master ~]# ssh-keygen -b 1024 -t rsa
一路回车
[root@k8s-master ~]# cd /root/.ssh
将本机公钥复制到其它两台虚拟机上并改名
[root@k8s-master ~]# ssh-copy-id -i  ~/.ssh/id_rsa.pub root@172.16.160.205
[root@k8s-master ~]# ssh-copy-id -i  ~/.ssh/id_rsa.pub root@172.16.160.207

配置完成后，三台虚拟机之前用SCP命令传文件，是不需要输入密码的。

k8s-master完成上面配置后测试是否还需要密码
[root@k8s-master ~]# vi 1.txt
[root@k8s-master ~]# scp -P 22 /root/.ssh/1.txt root@172.16.160.206:/root/.ssh/1.txt
[root@k8s-master ~]# scp -P 22 /root/.ssh/1.txt root@172.16.160.207:/root/.ssh/1.txt
k8s-node1完成上面配置后测试是否还需要密码
[root@k8s-master ~]# vi 2.txt
[root@k8s-master ~]# scp -P 22 /root/.ssh/2.txt root@172.16.160.205:/root/.ssh/2.txt
[root@k8s-master ~]# scp -P 22 /root/.ssh/2.txt root@172.16.160.207:/root/.ssh/2.txt
k8s-node1完成上面配置后测试是否还需要密码
[root@k8s-master ~]#vi 3.txt
[root@k8s-master ~]#scp -P 22 /root/.ssh/3.txt root@172.16.160.205:/root/.ssh/3.txt
[root@k8s-master ~]#scp -P 22 /root/.ssh/3.txt root@172.16.160.206:/root/.ssh/3.txt


1.3部署etcd服务
（集群方式）

1.3.1解压，创建链接
[root@k8s-master ~]# tar zxvf etcd-v3.1.0-linux-amd64.tar.gz -C /usr/local/
[root@k8s-master ~]# mv /usr/local/etcd-v3.1.0-linux-amd64/ /usr/local/etcd
[root@k8s-master ~]# ln -s /usr/local/etcd/etcd /usr/local/bin/etcd
[root@k8s-master ~]# ln -s /usr/local/etcd/etcdctl /usr/local/bin/etcdctl

1.3.2设置systemd服务文件
三台虚拟机都是一样的文件内容。
[root@k8s-master ~]# vi /usr/lib/systemd/system/etcd.service
[Unit]
Description=Eted Server 
After=network.target

[Service]
WorkingDirectory=/data/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/usr/local/bin/etcd
Type=notify
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

[root@k8s-master ~]#
其中WorkingDirector表示etcd数据保存的目录，需要在启动etcd服务之前进行创建

1.3.3etcd三节点配置
k8s-master
[root@k8s-master rpms]#  cat /etc/etcd/etcd.conf 
ETCD_NAME=etcd-master
ETCD_DATA_DIR="/data/etcd/etcd-master.etcd"
ETCD_LISTEN_PEER_URLS="http://172.16.160.205:2380"
ETCD_LISTEN_CLIENT_URLS="http://172.16.160.205:2379,http://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.16.160.205:2380"
ETCD_INITIAL_CLUSTER="etcd-master=http://172.16.160.205:2380,etcd-node01=http://172.16.160.206:2380,etcd-node02=http://172.16.160.207:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-00"
ETCD_ADVERTISE_CLIENT_URLS="http://172.16.160.205:2379"

k8s-node1
[root@k8s-node1 ~]# cat /etc/etcd/etcd.conf 
ETCD_NAME=kube-minion1
ETCD_DATA_DIR="/var/lib/etcd/kube-minion1"
ETCD_LISTEN_PEER_URLS="http://172.16.160.206:2380"
ETCD_LISTEN_CLIENT_URLS="http://172.16.160.206:2379, http://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.16.160.206:2380"
ETCD_INITIAL_CLUSTER="kube-master=http://172.16.160.205:2380,kube-minion1=http://172.16.160.206:2380,kube-minion2=http://172.16.160.207:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-00"
ETCD_ADVERTISE_CLIENT_URLS="http://172.16.160.206:2379"

k8s-node2
[root@k8s-node2 rpms]# cat /etc/etcd/etcd.conf 
ETCD_NAME=kube-minion2
ETCD_DATA_DIR="/var/lib/etcd/kube-minion2"
ETCD_LISTEN_PEER_URLS=http://172.16.160.207:2380
ETCD_LISTEN_CLIENT_URLS="http://172.16.160.207:2379, http://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.16.160.207:2380"
ETCD_INITIAL_CLUSTER="kube-master=http://172.16.160.205:2380,kube-minion1=http://172.16.160.206:2380,kube-minion2=http://172.16.160.207:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-00"
ETCD_ADVERTISE_CLIENT_URLS="http://172.16.160.207:2379"
[root@k8s-node2 rpms]# 

1.3.4三台的etcd服务启动
[root@k8s-master ~]# systemctl daemon-reload
[root@k8s-master ~]# systemctl enable etcd.service
[root@k8s-master ~]# systemctl start etcd.service

1.3.5etcd服务检查
三台虚拟机显示的结果相同。
[root@k8s-master rpms]# etcdctl cluster-health
member 48e928c68a0c1cd6 is healthy: got healthy result from http://172.16.160.205:2379
member 7eccc73b94e5aaea is healthy: got healthy result from http://172.16.160.207:2379
member d7378dd58f633a96 is healthy: got healthy result from http://172.16.160.206:2379
cluster is healthy
[root@k8s-master rpms]# etcdctl member list
48e928c68a0c1cd6: name=etcd-master peerURLs=http://172.16.160.205:2380 clientURLs=http://172.16.160.205:2379 isLeader=true
7eccc73b94e5aaea: name=kube-minion2 peerURLs=http://172.16.160.207:2380 clientURLs=http://172.16.160.207:2379 isLeader=false
d7378dd58f633a96: name=kube-minion1 peerURLs=http://172.16.160.206:2380 clientURLs=http://172.16.160.206:2379 isLeader=false
[root@k8s-master rpms]# 

1.4部署kube-apiserver服务
1.4.1解压，创建链接
[root@k8s-master ~]# tar zxvf kubernetes-server-linux-amd64.tar.gz  -C /usr/local/
[root@k8s-master ~]# ln -s /usr/local/kubernetes/server/bin/kube-apiserver /usr/local/bin/kube-apiserver
其他服务顺便做下软链接
[root@k8s-master ~]# ln -s /usr/local/kubernetes/server/bin/hyperkube /usr/local/bin/hyperkube
[root@k8s-master ~]# ln -s /usr/local/kubernetes/server/bin/kubeadm /usr/local/bin/kubeadm
[root@k8s-master ~]# ln -s /usr/local/kubernetes/server/bin/kube-controller-manager /usr/local/bin/kube-controller-manager
[root@k8s-master ~]# ln -s /usr/local/kubernetes/server/bin/kubectl  /usr/local/bin/kubectl
[root@k8s-master ~]# ln -s /usr/local/kubernetes/server/bin/kube-discovery /usr/local/bin/kube-discovery
[root@k8s-master ~]# ln -s /usr/local/kubernetes/server/bin/kube-dns  /usr/local/bin/kube-dns
[root@k8s-master ~]# ln -s /usr/local/kubernetes/server/bin/kubefed /usr/local/bin/kubefed
[root@k8s-master ~]# ln -s /usr/local/kubernetes/server/bin/kubelet /usr/local/bin/kubelet      
[root@k8s-master ~]# ln -s /usr/local/kubernetes/server/bin/kube-proxy /usr/local/bin/kube-proxy
[root@k8s-master ~]# ln -s /usr/local/kubernetes/server/bin/kube-scheduler /usr/local/bin/kube-scheduler

1.4.2配置kubernetes system config
[root@k8s-master rpms]# cat /etc/kubernetes/config 
KUBE_LOGTOSTDERR="--logtostderr=true"
KUBE_LOG_DIR="--log-dir=/data/logs/kubernetes"
KUBE_LOG_LEVEL="--v=2"
KUBE_ALLOW_PRIV="--allow-privileged=false"
KUBE_MASTER="--master=http://172.16.160.205:8080"
#KUBE_ETCD_SERVERS="--etcd-servers=http://172.16.160.205:2379"

1.4.3配置kuber-apiserver启动参数
[root@k8s-master rpms]# cat /etc/kubernetes/apiserver
KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"
#KUBE_API_PORT="--port=8080"
#KUBELET_PORT="--kubelet-port=10250"
KUBE_ETCD_SERVERS="--etcd-servers=http://172.16.160.205:2379,http://172.16.160.206:2379,http://172.16.160.207:2379"
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"
KUBE_API_ARGS=" "

[root@k8s-master rpms]# 

1.4.4设置systemd服务文件
[root@k8s-master rpms]# cat /usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
After=etcd.service

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/apiserver
#User=kube
ExecStart=/usr/local/bin/kube-apiserver \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_ETCD_SERVERS \
            $KUBE_API_ADDRESS \
            $KUBE_API_PORT \
            $KUBELET_PORT \
            $KUBE_ALLOW_PRIV \
            $KUBE_SERVICE_ADDRESSES \
            $KUBE_ADMISSION_CONTROL \
            $KUBE_API_ARGS
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

[root@k8s-master rpms]# 

1.4.5启动kube-api-servers服务
[root@k8s-master ~]# systemctl daemon-reload
[root@k8s-master ~]# systemctl enable kube-apiserver.service
Created symlink from /etc/systemd/system/multi-user.target.wants/kube-apiserver.service to /usr/lib/systemd/system/kube-apiserver.service.
[root@k8s-master ~]# systemctl start kube-apiserver.service

1.4.6验证服务
在其它电脑上访问：http://172.16.160.205:8080/
[root@k8s-master rpms]# systemctl status kube-apiserver.service
● kube-apiserver.service - Kubernetes API Server
   Loaded: loaded (/usr/lib/systemd/system/kube-apiserver.service; enabled; vendor preset: disabled)
   Active: active (running) since 三 2017-02-22 10:09:01 CST; 47min ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 3741 (kube-apiserver)
   CGroup: /system.slice/kube-apiserver.service
           └─3741 /usr/local/bin/kube-apiserver --logtostderr=true --v=2 --etcd-servers=http://172.16.160.205:2379,http://172.16.160.206:2379...

2月 22 10:56:05 k8s-master kube-apiserver[3741]: I0222 10:56:05.109387    3741 panics.go:76] PUT /api/v1/namespaces/kube-system/endpo...:36622]
2月 22 10:56:07 k8s-master kube-apiserver[3741]: I0222 10:56:07.081740    3741 panics.go:76] GET /api/v1/namespaces/kube-system/endpo…05:36590]
2月 22 10:56:07 k8s-master kube-apiserver[3741]: I0222 10:56:07.084096    3741 
[root@k8s-master rpms]# 

1.5部署kube-controller-manager服务
1.5.1设置systemd服务文件
[root@k8s-master rpms]# cat /usr/lib/systemd/system/kube-controller-manager.service 
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=kube-apiserver.service
Requires=kube-apiserver.service

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/controller-manager
ExecStart=/usr/local/bin/kube-controller-manager \
   $KUBE_LOGTOSTDERR \
   $KUBE_LOG_LEVEL \
   $KUBE_LOG_DIR \
   $KUBE_MASTER \
   $KUBE_CONTROLLER_MANAGER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
[root@k8s-master rpms]# 

1.5.2配置kube-controller-manager启动参数
[root@k8s-master rpms]# cat /etc/kubernetes/controller-manager
KUBE_CONTROLLER_MANAGER_ARGS=""

1.5.3启动kube-controller-manager服务
[root@k8s-master ~]# systemctl daemon-reload
[root@k8s-master ~]# systemctl enable kube-controller-manager
Created symlink from /etc/systemd/system/multi-user.target.wants/kube-controller-manager.service to /usr/lib/systemd/system/kube-controller-manager.service.
[root@k8s-master ~]# systemctl start kube-controller-manager
[root@k8s-master ~]# systemctl status kube-controller-manager

1.6部署kube-scheduler服务
1.6.1设置systemd服务文件
[root@k8s-master rpms]# cat /usr/lib/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler Plugin
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=kube-apiserver.service
Requires=kube-apiserver.service

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/scheduler
ExecStart=/usr/local/bin/kube-scheduler \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_LOG_DIR \
            $KUBE_MASTER \
            $KUBE_SCHEDULER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
[root@k8s-master rpms]# 

1.6.2配置kube-schedulerr启动参数
[root@k8s-master rpms]#  cat /etc/kubernetes/schedulerr
KUBE_SCHEDULER_ARGS=""
[root@k8s-master rpms]# 

1.6.3启动kube-scheduler服务
[root@k8s-master ~]# systemctl daemon-reload
[root@k8s-master ~]# systemctl enable kube-scheduler
Created symlink from /etc/systemd/system/multi-user.target.wants/kube-scheduler.service to /usr/lib/systemd/system/kube-scheduler.service.
[root@k8s-master ~]# systemctl start kube-scheduler
[root@k8s-master ~]# systemctl status kube-scheduler

1.7两个NODE节点安装docker-engine
[root@test yum.repos.d]# vi  /etc/yum.repos.d/docker.repo
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/$releasever/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
 
[root@test yum.repos.d]#yum install -y docker-engine
 
[root@zhuyrdocker ~]# service docker start
Redirecting to /bin/systemctl start  docker.service
[root@zhuyrdocker ~]# docker -v
Docker version 1.12.6, build 78d1802
[root@zhuyrdocker ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
[root@zhuyrdocker ~]#

1.8安装kubernetes客户端
两台NODE节点虚拟机都要安装kubelet，kube-proxy
[root@k8s-node1 rpms]# tar zxvf kubernetes-client-linux-amd64.tar.gz -C /usr/local/
kubernetes/
kubernetes/client/
kubernetes/client/bin/
kubernetes/client/bin/kubefed
kubernetes/client/bin/kubectl
[root@k8s-node1 rpms]# ln -s /usr/local/kubernetes/client/bin/kubectl /usr/local/bin/kubectl
[root@k8s-node1 rpms]# ln -s /usr/local/kubernetes/client/bin/kubefed /usr/local/bin/kubefed
[root@k8s-node1 rpms]# ln -s /usr/local/kubernetes/client/bin/kube-proxy /usr/local/bin/kube-proxy
[root@k8s-node1 rpms]# ln -s /usr/local/kubernetes/client/bin/kubelet /usr/local/bin/kubelet

kube-proxy包默认client没有可以从server拷贝过来
[root@k8s-master bin]# scp -P 22 /usr/local/kubernetes/server/bin/kubelet root@172.16.160.207:/usr/local/kubernetes/client/bin/   
[root@k8s-master bin]# scp -P 22 /usr/local/kubernetes/server/bin/kube-proxy root@172.16.160.207:/usr/local/kubernetes/client/bin/  
[root@k8s-master bin]# scp -P 22 /usr/local/kubernetes/server/bin/kubelet root@172.16.160.206:/usr/local/kubernetes/client/bin/  
^[[A[root@k8s-master bin]# scp -P 22 /usr/local/kubernetes/server/bin/kube-proxy root@172.16.160.206:/usr/local/kubernetes/client/bin/   
[root@k8s-master bin]# 

部署kubelet服务
[root@k8s-node1 bin]# cd /etc/
[root@k8s-node1 etc]# mkdir kubernetes
[root@k8s-node1 etc]# vi /etc/kubernetes/config

KUBE_LOGTOSTDERR="--logtostderr=false"
KUBE_LOG_DIR="--log-dir=/data/logs/kubernetes"
KUBE_LOGTOSTDERR="--logtostderr=false"
KUBE_LOG_DIR="--log-dir=/data/logs/kubernetes"
KUBE_LOG_LEVEL="--v=2"
KUBE_ALLOW_PRIV="--allow-privileged=false"
KUBE_MASTER="--master=http://172.16.160.205:8080"

[root@k8s-node1 bin]# vi /usr/lib/systemd/system/kubelet.service

[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/data/kubelet
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/local/bin/kubelet \
   $KUBE_LOGTOSTDERR \
   $KUBE_LOG_LEVEL \
   $KUBE_LOG_DIR \
   $KUBELET_API_SERVER \
   $KUBELET_ADDRESS \
   $KUBELET_PORT \
   $KUBELET_HOSTNAME \
   $KUBE_ALLOW_PRIV \
   $KUBELET_POD_INFRA_CONTAINER \
   $KUBELET_ARGS
Restart=on-failure

[Install]
WantedBy=multi-user.target
[root@k8s-node1 bin]# 

配置kubelet启动参数
[root@k8s-node1 etc]# vi /etc/kubernetes/kubelet

KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_PORT="--port=10250"
KUBELET_HOSTNAME="--hostname-override=k8s-node1"
KUBELET_API_SERVER="--api-servers=http://172.16.160.205:8080"
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"
KUBELET_ARGS=""

启动kubelet服务
[root@k8s-node1 ~]# systemctl daemon-reload
[root@k8s-node1 ~]# systemctl enable kubelet.service
Created symlink from /etc/systemd/system/multi-user.target.wants/kubelet.service to /usr/lib/systemd/system/kubelet.service.
[root@k8s-node1 ~]# systemctl start kubelet.service

部署kube-proxy服务
设置systemd服务文件
[root@k8s-node1 bin]# vi /usr/lib/systemd/system/kube-proxy.service

[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/proxy
ExecStart=/usr/local/bin/kube-proxy \
   $KUBE_LOGTOSTDERR \
   $KUBE_LOG_LEVEL \
   $KUBE_LOG_DIR \
   $KUBE_MASTER \
   $KUBE_PROXY_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target


配置kubelet启动参数
[root@k8s-node2 bin]# vi /etc/kubernetes/proxy

KUBE_PROXY_ARGS=""

启动kubelet服务
[root@k8s-node1 ~]# systemctl daemon-reload
[root@k8s-node1 ~]# systemctl enable kube-proxy.service
Created symlink from /etc/systemd/system/multi-user.target.wants/kube-proxy.service to /usr/lib/systemd/system/kube-proxy.service.
[root@k8s-node1 ~]# systemctl start kube-proxy.service

验证节点是否启动
[root@k8s-node1 ~]# kubectl get nodes
NAME        STATUS    AGE
k8s-node1   Ready     9m
