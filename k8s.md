简介
```none
k8s是谷歌开源的一个基于容器的分布式架构方案，支持自动负载均衡、故障处理和部署实施等。
k8s集群机器由一组Master和一群Node节点组成
在Master上运行着与集群管理相关的一组进程kube-apiserver、kube-controller-manager、kube-scheduler
在Node上运行着docker、kubelet、kube-proxy进程，他们负责Pod的创建、启动、重启、停止、销毁以及实现软件层面的负载均衡
另外Node上还运行着一组虚拟网桥服务进程（如fannel），负责实现跨机器间容器的网络通信；
在集群节点上还运行着一组ETCD集群服务，用以保存集群资源的所有状态信息。
etcd：集群数据库，存储集群资源的各种信息；
kube-apiserver：集群控制的入口进程，提供HTTP REST API接口，通过kube-apiserver可以实现集群资源的增、删、查、改等各项功能；
kube-controller-manager：资源总控制中心，通过kube-apiserver监控集群状态，确保集群处于预期工作状态；
kube-scheduler：通过kube-apiserver查询各个Node状态，负责将Pod分配到符合状态的Node上，是集群资源的调度室。
docker：负责容器的创建和管理；
fannel：负责集群的网络通信；
kubelet：负责Pod对应容器的创建、启动和停止等任务，与master一起负责集群管理的基本功能；
kube-proxy：负责k8s通信与负载均衡机制的实现。
```
各节点配置
```none
192.168.88.11 node01 etcd、flanneld、docker、kube-apiserver、kube-controller-manager、kube-scheduler
192.168.88.12 node02 etcd、flanneld、docker、kubelet、kube-proxy
192.168.88.13 node03 etcd、flanneld、docker、kubelet、kube-proxy

cat >> /etc/hosts << EOF
192.168.88.11 node01
192.168.88.12 node02
192.168.88.13 node03
EOF
echo node01 > /etc/hostname
```

---

etcd集群-node01配置

证书部署
```none
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.0/cfssl_1.6.0_linux_amd64 -O /usr/local/bin/cfssl
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.0/cfssljson_1.6.0_linux_amd64 -O /usr/local/bin/cfssljson
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.0/cfssl-certinfo_1.6.0_linux_amd64 -O /usr/local/bin/cfssl-certinfo

chmod +x /usr/local/bin/cfssl*
mkdir -p /k8s/package/
mkdir -p /k8s/etcd/{bin,cert,cfg}
cd /k8s/etcd/cert

cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "k8s": {
        "expiry": "87600h",
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ]
      }
    }
  }
}
EOF

cat > ca-csr.json << EOF
{
  "CN": "etcd ca",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "beijing",
      "ST": "beijing"
    }
  ]
}
EOF

cat > etcd-server-csr.json << EOF
{
  "CN": "etcd",
  "hosts": [
    "192.168.88.11",
    "192.168.88.12",
    "192.168.88.13"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "beijing",
      "ST": "beijing"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=k8s etcd-server-csr.json | cfssljson -bare server
```
etcd配置
```none
######## flannel不支持etcd3，etcd要启动v2 ########
https://github.com/etcd-io/etcd/releases

wget https://github.com/etcd-io/etcd/releases/download/v3.5.0/etcd-v3.5.0-linux-amd64.tar.gz -P /k8s/package/
tar -zxvf /k8s/package/etcd-v3.5.0-linux-amd64.tar.gz -C /k8s/package/
cp /k8s/package/etcd-v3.5.0-linux-amd64/{etcd,etcdctl,etcdutl} /k8s/etcd/bin/

cat > /k8s/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd01"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.88.11:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.88.11:2379,http://127.0.0.1:2379"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.88.11:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.88.11:2379"
ETCD_INITIAL_CLUSTER="etcd01=https://192.168.88.11:2380,etcd02=https://192.168.88.12:2380,etcd03=https://192.168.88.13:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
#[Security]
ETCD_CERT_FILE="/k8s/etcd/cert/server.pem"
ETCD_KEY_FILE="/k8s/etcd/cert/server-key.pem"
ETCD_TRUSTED_CA_FILE="/k8s/etcd/cert/ca.pem"
ETCD_PEER_CERT_FILE="/k8s/etcd/cert/server.pem"
ETCD_PEER_KEY_FILE="/k8s/etcd/cert/server-key.pem"
ETCD_PEER_TRUSTED_CA_FILE="/k8s/etcd/cert/ca.pem"
EOF

cat > /k8s/etcd/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
[Service]
Type=notify
EnvironmentFile=/k8s/etcd/cfg/etcd.conf
ExecStart=/k8s/etcd/bin/etcd --enable-v2
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF

ln -sf /k8s/etcd/etcd.service /usr/lib/systemd/system/
systemctl daemon-reload && systemctl enable --now etcd
rsync -av --delete /k8s/etcd node02:/k8s/
rsync -av --delete /k8s/etcd node03:/k8s/
```
etcd集群-node02配置
```none
#node02
sed -i '/ETCD_INITIAL_CLUSTER/!s/11/12/;/ETCD_NAME/s/etcd01/etcd02/' /k8s/etcd/cfg/etcd.conf
ln -sf /k8s/etcd/etcd.service /usr/lib/systemd/system/
systemctl daemon-reload && systemctl enable --now etcd
```
etcd集群-node03配置
```none
#node03
sed -i '/ETCD_INITIAL_CLUSTER/!s/11/13/;/ETCD_NAME/s/etcd01/etcd03/' /k8s/etcd/cfg/etcd.conf
ln -sf /k8s/etcd/etcd.service /usr/lib/systemd/system/
systemctl daemon-reload && systemctl enable --now etcd
```
状态检查
```none
node01启动时会启动失败，待三节点都启动后正常
ETCDCTL_API=2 /k8s/etcd/bin/etcdctl --ca-file=/k8s/etcd/cert/ca.pem --cert-file=/k8s/etcd/cert/server.pem --key-file=/k8s/etcd/cert/server-key.pem --endpoints="https://192.168.88.11:2379,https://192.168.88.12:2379,https://192.168.88.13:2379" cluster-health
member 2a3100fb6c7ba717 is healthy: got healthy result from https://192.168.88.13:2379
member 5ef82fbc38d7f023 is healthy: got healthy result from https://192.168.88.11:2379
member bd76b69252666cb6 is healthy: got healthy result from https://192.168.88.12:2379
```
---

docker安装
```none
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum clean all && yum makecache fast
yum install docker-ce docker-ce-cli containerd.io -y
systemctl enable --now docker.service
docker version
```
---
fannel安装配置
```none
Falnnel要用etcd存储自身一个子网信息，所以要保证能成功连接Etcd，写入预定义子网段
ETCDCTL_API=2 /k8s/etcd/bin/etcdctl --ca-file=/k8s/etcd/cert/ca.pem --cert-file=/k8s/etcd/cert/server.pem --key-file=/k8s/etcd/cert/server-key.pem --endpoints="https://192.168.88.11:2379,https://192.168.88.12:2379,https://192.168.88.13:2379" set /coreos.com/network/config '{"Network":"172.17.0.0/16","Backend":{"Type":"vxlan"}}'
验证
ETCDCTL_API=2 /k8s/etcd/bin/etcdctl --ca-file=/k8s/etcd/cert/ca.pem --cert-file=/k8s/etcd/cert/server.pem --key-file=/k8s/etcd/cert/server-key.pem --endpoints="https://192.168.88.11:2379,https://192.168.88.12:2379,https://192.168.88.13:2379" get /coreos.com/network/config

https://github.com/flannel-io/flannel/releases
wget https://github.com/flannel-io/flannel/releases/download/v0.14.0/flannel-v0.14.0-linux-amd64.tar.gz -P /k8s/package/
tar -zxvf /k8s/package/flannel-v0.14.0-linux-amd64.tar.gz -C /k8s/package/ 
mkdir -p /k8s/flanneld/{bin,cfg}
cp /k8s/package/{flanneld,mk-docker-opts.sh} /k8s/flanneld/bin/

cat > /k8s/flanneld/cfg/flanneld << EOF
FLANNEL_OPTIONS="-etcd-endpoints=https://192.168.88.11:2379,https://192.168.88.12:2379,https://192.168.88.13:2379 -etcd-cafile=/k8s/etcd/cert/ca.pem -etcd-certfile=/k8s/etcd/cert/server.pem -etcd-keyfile=/k8s/etcd/cert/server-key.pem"
EOF

cat > /k8s/flanneld/flanneld.service << "EOF"
[Unit]
Description=Flanneld overlay address etcd agent
After=network-online.target network.target
Before=docker.service
[Service]
Type=notify
EnvironmentFile=/k8s/flanneld/cfg/flanneld
ExecStart=/k8s/flanneld/bin/flanneld -ip-masq $FLANNEL_OPTIONS
ExecStartPost=/k8s/flanneld/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/subnet.env
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF

ln -sf /k8s/flanneld/flanneld.service /usr/lib/systemd/system/
systemctl daemon-reload && systemctl enable --now flanneld.service

sed -i '/Type/a EnvironmentFile=/run/flannel/subnet.env' /usr/lib/systemd/system/docker.service
sed -i '/ExecStart/c ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS ' /usr/lib/systemd/system/docker.service
systemctl daemon-reload && systemctl restart docker

rsync -av --delete /k8s/flanneld node02:/k8s/
rsync -av --delete /k8s/flanneld node02:/k8s/
```

node02 & node03
```none
ln -sf /k8s/flanneld/flanneld.service /usr/lib/systemd/system/
systemctl daemon-reload && systemctl enable --now flanneld.service

sed -i '/Type/a EnvironmentFile=/run/flannel/subnet.env' /usr/lib/systemd/system/docker.service
sed -i '/ExecStart/c ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS' /usr/lib/systemd/system/docker.service
systemctl daemon-reload && systemctl restart docker
```
各个节点会分配一个不同的c段网络，本机docker0为网关地址，在不同节点相互测试网络连通性。在部署Kubernetes之前一定要确保etcd、flannel、docker是正常工作
```none
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: ens32: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:2a:e7:ed brd ff:ff:ff:ff:ff:ff
    inet 192.168.88.11/24 brd 192.168.88.255 scope global noprefixroute dynamic ens32
       valid_lft 38sec preferred_lft 38sec
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:cd:12:04:91 brd ff:ff:ff:ff:ff:ff
    inet 172.17.24.1/24 brd 172.17.24.255 scope global docker0
       valid_lft forever preferred_lft forever
4: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default 
    link/ether 56:d7:7a:5b:ee:5b brd ff:ff:ff:ff:ff:ff
    inet 172.17.24.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
```
---
------master节点------

kube-apiserver部署
```none
https://github.com/kubernetes/kubernetes
https://kubernetes.io/zh/docs/setup/release/notes/

mkdir -p /k8s/kubernetes/{bin,cert,cfg}
wget https://dl.k8s.io/v1.18.0/kubernetes-server-linux-amd64.tar.gz -P /k8s/package/
tar -zxf /k8s/package/kubernetes-server-linux-amd64.tar.gz -C /k8s/package/
cp /k8s/package/kubernetes/server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl} /k8s/kubernetes/bin/
echo 'source <(/k8s/kubernetes/bin/kubectl completion bash)' >> /etc/profile
echo 'PATH=$PATH:/k8s/kubernetes/bin/' >> /etc/profile && source /etc/profile
```
证书制作-ca
```none
cd /k8s/kubernetes/cert/
cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "expiry": "87600h",
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ]
      }
    }
  }
}
EOF

cat > ca-csr.json << EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "beijing",
      "ST": "beijing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
```
生成apiserver证书
```none
cat > api-server-csr.json << EOF
{
  "CN": "kubernetes",
  "hosts": [
    "10.0.0.1",
    "127.0.0.1",
    "192.168.88.11",
    "192.168.88.12",
    "192.168.88.13",
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
      "L": "beijing",
      "ST": "beijing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
注：host中的最后几个IP为apiserver的IP，一般为master集群的所有IP

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes api-server-csr.json | cfssljson -bare server
```
生成kube-proxy证书
```none
cat > kube-proxy-csr.json << EOF
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
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
```
生成kube-admin证书
```none
cat > admin-csr.json << EOF
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
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```
生成token文件。第一列：随机字符串自己可生成。第二列:用户名。第三列:UID。第四列:用户组
```none
cat > /k8s/kubernetes/cfg/token.csv << EOF
674c457d4dcf2eefe4920d7dbb6b0ddc,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```
创建apiserver配置文件、启动文件并启动服务
```none
cat > /k8s/kubernetes/cfg/kube-apiserver << EOF
KUBE_APISERVER_OPTS="--bind-address=192.168.88.11 \
--advertise-address=192.168.88.11 \
--secure-port=6443 --v=4 \
--allow-privileged=true \
--service-cluster-ip-range=10.0.0.0/24 \
--etcd-servers=https://192.168.88.11:2379,https://192.168.88.12:2379,https://192.168.88.13:2379 \
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
--authorization-mode=RBAC,Node \
--kubelet-https=true \
--enable-bootstrap-token-auth \
--token-auth-file=/k8s/kubernetes/cfg/token.csv \
--service-node-port-range=30000-50000 \
--tls-cert-file=/k8s/kubernetes/cert/server.pem \
--tls-private-key-file=/k8s/kubernetes/cert/server-key.pem \
--client-ca-file=/k8s/kubernetes/cert/ca.pem \
--service-account-key-file=/k8s/kubernetes/cert/ca-key.pem \
--etcd-cafile=/k8s/etcd/cert/ca.pem \
--etcd-certfile=/k8s/etcd/cert/server.pem \
--etcd-keyfile=/k8s/etcd/cert/server-key.pem"
EOF

参数说明：
* --logtostderr 启用日志
* --v 日志等级
* --etcd-servers etcd集群地址
* --bind-address 监听地址
* --secure-port https安全端口
* --advertise-address 集群通告地址
* --allow-privileged 启用授权
* --service-cluster-ip-range Service虚拟IP地址段
* --enable-admission-plugins 准入控制模块
* --authorization-mode 认证授权，启用RBAC授权和节点自管理
* --enable-bootstrap-token-auth 启用TLS bootstrap功能，后面会讲到
* --token-auth-file token文件
* --service-node-port-range Service Node类型默认分配端口范围

cat > /k8s/kubernetes/kube-apiserver.service << "EOF" 
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=etcd.service
Wants=etcd.service
[Service]
Type=notify
EnvironmentFile=-/k8s/kubernetes/cfg/kube-apiserver
ExecStart=/k8s/kubernetes/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF

ln -sf /k8s/kubernetes/kube-apiserver.service /usr/lib/systemd/system/
systemctl daemon-reload && systemctl enable --now kube-apiserver.service
```
创建controller-manager配置文件、启动文件并启动服务
```none
cat > /k8s/kubernetes/cfg/kube-controller-manager << EOF
KUBE_CONTROLLER_MANAGER_OPTS="--v=4 --master=127.0.0.1:8080 \
--leader-elect=true \
--address=127.0.0.1 \
--service-cluster-ip-range=10.0.0.0/24 \
--cluster-name=kubernetes \
--cluster-signing-cert-file=/k8s/kubernetes/cert/ca.pem \
--cluster-signing-key-file=/k8s/kubernetes/cert/ca-key.pem  \
--root-ca-file=/k8s/kubernetes/cert/ca.pem \
--service-account-private-key-file=/k8s/kubernetes/cert/ca-key.pem \
--experimental-cluster-signing-duration=87600h0m0s"
EOF

cat > /k8s/kubernetes/kube-controller-manager.service << "EOF"
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
After=kube-apiserver.service
Wants=kube-apiserver.service
[Service]
EnvironmentFile=-/k8s/kubernetes/cfg/kube-controller-manager
ExecStart=/k8s/kubernetes/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF

ln -sf /k8s/kubernetes/kube-controller-manager.service /usr/lib/systemd/system/
systemctl daemon-reload && systemctl enable --now kube-controller-manager.service
```
创建scheduler配置文件、启动文件并启动服务
```none
cat > /k8s/kubernetes/cfg/kube-scheduler << EOF
KUBE_SCHEDULER_OPTS="--v=4 --master=127.0.0.1:8080 --leader-elect"
EOF
参数详解：
--master         #连接本地的apiserver
--leader-elect   #当该组件启动多个时,自动选举(HA)

cat > /k8s/kubernetes/kube-scheduler.service << "EOF"
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
After=kube-apiserver.service
Wants=kube-apiserver.service 
[Service]
EnvironmentFile=-/k8s/kubernetes/cfg/kube-scheduler
ExecStart=/k8s/kubernetes/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF

ln -sf /k8s/kubernetes/kube-scheduler.service /usr/lib/systemd/system/
systemctl daemon-reload && systemctl enable --now kube-scheduler.service
```
验证
```none
echo 'export KUBERNETES_MASTER="127.0.0.1:8080"' >> /etc/profile && source /etc/profile
kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"} 
```
将kubelet-bootstrap用户绑定到系统集群角色,使用cluster-admin角色给用户system:anonymous做clusterrolebinding
```none
/k8s/kubernetes/bin/kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
/k8s/kubernetes/bin/kubectl create clusterrolebinding system:anonymous --clusterrole=cluster-admin --user=system:anonymous
```
创建kubeconfig文件，在生成kubernetes证书的目录下执行以下命令生成kubeconfig文件
```none
vim /k8s/kubernetes/kubeconfig.sh

cd /k8s/kubernetes/cert/
KUBE_APISERVER="https://192.168.88.11:6443"
BOOTSTRAP_TOKEN=674c457d4dcf2eefe4920d7dbb6b0ddc
# 设置集群参数
kubectl config set-cluster kubernetes --certificate-authority=./ca.pem --embed-certs=true --server=${KUBE_APISERVER}  --kubeconfig=bootstrap.kubeconfig
# 设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap --token=${BOOTSTRAP_TOKEN} --kubeconfig=bootstrap.kubeconfig
# 设置上下文参数
kubectl config set-context default --cluster=kubernetes --user=kubelet-bootstrap --kubeconfig=bootstrap.kubeconfig
# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
# 创建kube-proxy.kubeconfig文件
kubectl config set-cluster kubernetes --certificate-authority=./ca.pem --embed-certs=true --server=${KUBE_APISERVER} --kubeconfig=kube-proxy.kubeconfig
kubectl config set-credentials kube-proxy --client-certificate=./kube-proxy.pem --client-key=./kube-proxy-key.pem --embed-certs=true --kubeconfig=kube-proxy.kubeconfig
kubectl config set-context default --cluster=kubernetes --user=kube-proxy --kubeconfig=kube-proxy.kubeconfig
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

bash -x /k8s/kubernetes/kubeconfig.sh
ls /k8s/kubernetes/cert/{bootstrap.kubeconfig,kube-proxy.kubeconfig}  -l
-rw------- 1 root root 2167 Sep 18 11:07 /k8s/kubernetes/cert/bootstrap.kubeconfig
-rw------- 1 root root 6273 Sep 18 11:07 /k8s/kubernetes/cert/kube-proxy.kubeconfig

node2、node3
mkdir -p /k8s/kubernetes/{cfg,bin}

将这两个文件拷贝到Node节点/k8s/kubernetes/cfg
scp /k8s/kubernetes/cert/{bootstrap.kubeconfig,kube-proxy.kubeconfig} node02:/k8s/kubernetes/cfg/
scp /k8s/kubernetes/cert/{bootstrap.kubeconfig,kube-proxy.kubeconfig} node03:/k8s/kubernetes/cfg/

scp /k8s/package/kubernetes/server/bin/{kubelet,kube-proxy} node02:/k8s/kubernetes/bin/
scp /k8s/package/kubernetes/server/bin/{kubelet,kube-proxy} node03:/k8s/kubernetes/bin/
```

---
------node节点------

两个node节点执行，创建kubelet配置文件、启动文件并启动服务
```none
cat > /k8s/kubernetes/cfg/kubelet.config << EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 192.168.88.12
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS: ["10.0.0.2"]
clusterDomain: cluster.local.
failSwapOn: false
authentication:
  anonymous:
    enabled: true 
  webhook:
    enabled: false
EOF

cat > /k8s/kubernetes/cfg/kubelet << EOF
KUBELET_OPTS="--hostname-override=192.168.88.12 \
--kubeconfig=/k8s/kubernetes/cfg/kubelet.kubeconfig \
--bootstrap-kubeconfig=/k8s/kubernetes/cfg/bootstrap.kubeconfig \
--config=/k8s/kubernetes/cfg/kubelet.config \
--cert-dir=/k8s/kubernetes/cert --v=4 \
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0"
EOF
参数说明：
* --hostname-override 在集群中显示的主机名
* --kubeconfig 指定kubeconfig文件位置，会自动生成
* --bootstrap-kubeconfig 指定刚才生成的bootstrap.kubeconfig文件
* --cert-dir 颁发证书存放位置
* --pod-infra-container-image 管理Pod网络的镜像

cat > /k8s/kubernetes/kubelet.service << "EOF"
[Unit]
Description=Kubernetes Kubelet
After=docker.service
Requires=docker.service
[Service]
EnvironmentFile=/k8s/kubernetes/cfg/kubelet
ExecStart=/k8s/kubernetes/bin/kubelet $KUBELET_OPTS
Restart=on-failure
KillMode=process
[Install]
WantedBy=multi-user.target
EOF

ln -sf /k8s/kubernetes/kubelet.service /usr/lib/systemd/system/
systemctl daemon-reload && systemctl enable --now kubelet.service

# master节点审批Node加入集群，启动后还没加入到集群中，需要手动允许该节点才可以。
kubectl get csr
kubectl certificate approve node-csr-a5Gcvu23cU_FbUKATy2PekJ7uGhPNHqo_7l314-WYCs
kubectl get node
```
两个node节点执行，创建kube-proxy配置文件、启动文件并启动服务
```none
cat > /k8s/kubernetes/cfg/kube-proxy << EOF
KUBE_PROXY_OPTS="--hostname-override=192.168.88.12 \
--kubeconfig=/k8s/kubernetes/cfg/kube-proxy.kubeconfig \
--cluster-cidr=10.0.0.0/24 --proxy-mode=ipvs --v=4"
EOF

cat > /k8s/kubernetes/kube-proxy.service << "EOF" 
[Unit]
Description=Kubernetes Proxy
After=network.target
[Service]
EnvironmentFile=-/k8s/kubernetes/cfg/kube-proxy
ExecStart=/k8s/kubernetes/bin/kube-proxy $KUBE_PROXY_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF

ln -sf /k8s/kubernetes/kube-proxy.service /usr/lib/systemd/system/
systemctl daemon-reload && systemctl enable --now kube-proxy.service
```
coredns
```none
git clone https://github.com/coredns/deployment.git
cd deployment/kubernetes/
./deploy.sh -r 10.0.0.0/24 -i 10.0.0.2 -d cluster.local > coredns.yaml
kubectl apply -f coredns.yaml
kubectl get pods -n kube-system -o wide
```
yaml
```none
apiVersion: apps/v1
kind: Deployment
metadata:
  name: websv
spec:
  replicas: 1
  selector:
    matchLabels:
      app: websv
  template:
    metadata:
      labels:
        app: websv
    spec:
      containers:
      - name: nginx
        image: nginx:1.20-alpine
        volumeMounts:
        - name: htmldir
          mountPath: /usr/share/nginx/html/
      volumes:
      - name: htmldir
        hostPath:
          path: /data/
      nodeSelector:
        kubernetes.io/hostname: 192.168.88.11
---
apiVersion: v1
kind: Service
metadata:
  name: websv
  labels:
    app: websv
spec:
  type: NodePort
  clusterIP: 10.0.0.20
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    nodePort: 30080
  selector:
    app: websv
```
匹配取反
```none
apiVersion: apps/v1
kind: Deployment
metadata:
  name: websv1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: websv1
  template:
    metadata:
      labels:
        app: websv1
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: NotIn
                values:
                - 192.168.88.11
                - 192.168.88.12
      containers:
      - name: nginx
        image: nginx:alpine
---
apiVersion: v1
kind: Service
metadata:
  name: websv1
  labels:
    app: websv1
spec:
  clusterIP: 10.0.0.10
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
  selector:
    app: websv1
```








