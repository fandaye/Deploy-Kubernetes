## 部署环境

### 主机节点清单

| 服务器名 |    ip地址     | etcd | K8S server | K8s node |
| :------: | :-----------: | :--: | :--------: | :------: |
|  node01  | 172.16.50.111 |  Y   |     Y      |          |
|  node02  | 172.16.50.113 |  Y   |     Y      |          |
|  node03  | 172.16.50.115 |  Y   |     Y      |          |
|  node04  | 172.16.50.116 |      |            |    Y     |
|  node05  | 172.16.50.118 |      |            |    Y     |
|  node06  | 172.16.50.120 |      |            |    Y     |
|  node07  | 172.16.50.128 |      |            |    Y     |

### 版本信息

- Linux版本：CentOS 7.6.1810

- 内核版本：3.10.0-957.1.3.el7.x86_64

  ```
  # cat /etc/redhat-release 
  CentOS Linux release 7.6.1810 (Core) 
  
  # uname -r
  3.10.0-957.1.3.el7.x86_64
  ```

- docker 版本

  ```
  Server Version: 1.13.1
  ```

- 网络组件

  weave

### 安装前准备

- 关闭selinux

  ```
  # vim /etc/sysconfig/selinux 
  SELINUX=disabled
  ```

- 关闭防火墙

  ```
  # systemctl stop firewalld && \
  systemctl disable firewalld
  ```

- 修改内核参数

  ```
  # cat  > /etc/sysctl.d/k8s.conf << EOF
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  net.ipv4.ip_forward = 1
  EOF
  
  sysctl -p /etc/sysctl.d/k8s.conf
  ```

- 拉取项目

  ```
  # yum install git -y && \
  mkdir /data && \
  cd /data && \
  git clone https://github.com/fandaye/Deploy-Kubernetes.git
  
  # cd /data/Deploy-Kubernetes
  # git checkout v1.13.4  # 切换到v1.13.4分支
  ```

- 导入镜像

  ```
  # cd /data/Deploy-Kubernetes/image
  
  # for i in `ls *.zip` ; do \
  unzip $i ; \
  done
  
  # for i in `ls *tar` ; do \
  docker load -i $i ; \
  done
  ```

  

## Docker 部署

```
# yum install docker -y 
# cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://8ph3bzw4.mirror.aliyuncs.com"],
  "graph": "/data/docker"
}
EOF

# mkdir /data/docker -p && \
systemctl start docker && systemctl enable docker 
```



##kube组件安装

**添加kube源**

```
#　cat > /etc/yum.repos.d/kube.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# yum install  /data/Deploy-Kubernetes/pkg/*.rpm -y
# systemctl enable kubelet
```



## Etcd集群部署

**安装包**

```
cd /data/Deploy-Kubernetes/pkg && \
tar -zxf etcd-v3.3.11-linux-amd64.tar.gz && \
cp etcd-v3.3.11-linux-amd64/{etcd,etcdctl} /usr/bin/ && \
rm -rf etcd-v3.3.11-linux-amd64
```

**使用 `kubeadm` 生成`Etcd`所需证书**

```
[root@node01 ~]# mkdir /etc/kubernetes/pki/etcd -p
# kubeadm init phase certs etcd-ca && \
kubeadm init phase certs apiserver-etcd-client && \
kubeadm init phase certs etcd-healthcheck-client && \
kubeadm init phase certs etcd-peer && \
kubeadm init phase certs etcd-server

[root@node01 ~]# for node in 113 115 ; do \
scp /etc/kubernetes/pki/etcd/{ca.crt,ca.key} \
root@172.16.50.${node}:/etc/kubernetes/pki/etcd/ && \
scp /etc/kubernetes/pki/{apiserver-etcd-client.crt,apiserver-etcd-client.key} \
root@172.16.50.${node}:/etc/kubernetes/pki ; \
done

[root@node02 ~]# kubeadm init phase certs etcd-healthcheck-client && \
kubeadm init phase certs etcd-peer && \
kubeadm init phase certs etcd-server

[root@node03 ~]# kubeadm init phase certs etcd-healthcheck-client && \
kubeadm init phase certs etcd-peer && \
kubeadm init phase certs etcd-server

```

**Etcd 启动脚本**

`/usr/lib/systemd/system/etcd.service`

```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
  --advertise-client-urls=https://{{NodeIP}}:2379 \
  --cert-file=/etc/kubernetes/pki/etcd/server.crt \
  --client-cert-auth=true \
  --data-dir=/var/lib/etcd \
  --initial-advertise-peer-urls=https://{{NodeIP}}:2380 \
  --key-file=/etc/kubernetes/pki/etcd/server.key \
  --listen-client-urls=https://127.0.0.1:2379,https://{{NodeIP}}:2379 \
  --listen-peer-urls=https://{{NodeIP}}:2380 \
  --name={{NodeName}} \
  --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt \
  --peer-client-cert-auth=true \
  --peer-key-file=/etc/kubernetes/pki/etcd/peer.key \
  --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt \
  --snapshot-count=10000 \
  --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt \
  --initial-cluster-token=etcd-cluster-0 \
  --initial-cluster=node01=https://172.16.50.111:2380,node02=https://172.16.50.113:2380,node03=https://172.16.50.115:2380 \
  --initial-cluster-state=new
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
> 替换 {{NodeIP}} / {{NodeName}}

**启动etcd 集群**

```
systemctl enable etcd && systemctl start etcd
```

**检查集群运行**

```
# for i in 111 113 115 ; do \
etcdctl \
--endpoints=https://172.16.50.$i:2379 \
--ca-file=/etc/kubernetes/pki/etcd/ca.crt \
--cert-file=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
--key-file=/etc/kubernetes/pki/etcd/healthcheck-client.key \
member list \
; done

输出: 
... peerURLs=https://172.16.50.115:2380 clientURLs=https://172.16.50.115:2379 isLeader=true
... peerURLs=https://172.16.50.113:2380 clientURLs=https://172.16.50.113:2379 isLeader=false
... peerURLs=https://172.16.50.111:2380 clientURLs=https://172.16.50.111:2379 isLeader=false
```



## 初始化Kubernetes集群

**编辑配置文件**`/data/Deploy-Kubernetes/config/config.yaml`

```
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: stable
apiServer:
  certSANs:
  - "node01"
  - "node02"
  - "node03"
  - "172.16.50.111"
  - "172.16.50.113"
  - "172.16.50.115"
  - "172.16.50.190"
controlPlaneEndpoint: "172.16.50.190:6443"
etcd:
    external:
        endpoints:
        - https://172.16.50.111:2379
        - https://172.16.50.113:2379
        - https://172.16.50.115:2379
        caFile: /etc/kubernetes/pki/etcd/ca.crt
        certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
        keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
```

> 172.16.50.190 作为负载IP

**添加IP地址**

```
[root@node01 ~]# ifconfig eth0:0 172.16.50.190 netmask 255.255.255.0 up
```

> 比较懒，可以使用  keepalived 来实现故障自动转移

**初始化**

```
[root@node01 ~]# kubeadm init --config /data/Deploy-Kubernetes/config/config.yaml 
```

**查看集群状态**

```
[root@node01 ~]# mkdir -p $HOME/.kube && cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@node01 ~]# kubectl get node
NAME     STATUS     ROLES    AGE    VERSION
node01   NotReady   master   6m4s   v1.13.4

[root@node01 ~]# kubectl get pod --all-namespaces # 加上 -o wide 参数显示更详细信息
NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE
kube-system   coredns-86c58d9df4-ckc8f         0/1     Pending   0          8m3s
kube-system   coredns-86c58d9df4-wt2dp         0/1     Pending   0          8m3s
kube-system   kube-apiserver-node01            1/1     Running   0          7m5s
kube-system   kube-controller-manager-node01   1/1     Running   0          7m4s
kube-system   kube-proxy-q2449                 1/1     Running   0          8m3s
kube-system   kube-scheduler-node01            1/1     Running   0          7m1s
```

> 网络安装之后 coredns 才会启动

**安装网络**

```
[root@node01 ～]# kubectl create -f /data/Deploy-Kubernetes/config/weave-net.yaml 
serviceaccount/weave-net created
clusterrole.rbac.authorization.k8s.io/weave-net created
clusterrolebinding.rbac.authorization.k8s.io/weave-net created
role.rbac.authorization.k8s.io/weave-net created
rolebinding.rbac.authorization.k8s.io/weave-net created
daemonset.extensions/weave-net created

再次查看集群信息

[root@node01 ~]# kubectl get node
NAME     STATUS   ROLES    AGE   VERSION
node01   Ready    master   10m   v1.13.4
[root@node01 ~]# kubectl get pod --all-namespaces 
NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE
kube-system   coredns-86c58d9df4-ckc8f         1/1     Running   0          10m
kube-system   coredns-86c58d9df4-wt2dp         1/1     Running   0          10m
kube-system   kube-apiserver-node01            1/1     Running   0          9m36s
kube-system   kube-controller-manager-node01   1/1     Running   0          9m35s
kube-system   kube-proxy-q2449                 1/1     Running   0          10m
kube-system   kube-scheduler-node01            1/1     Running   0          9m32s
kube-system   weave-net-6znrt                  2/2     Running   0          91s
```

**拷贝证书到 node02 node03 节点**

```
[root@node01 ~]# for node in 113 115 ; do \
scp /etc/kubernetes/pki/{ca.crt,ca.key,sa.key,sa.pub,front-proxy-ca.crt,front-proxy-ca.key} \
root@172.16.50.$node:/etc/kubernetes/pki/ && \
scp /etc/kubernetes/admin.conf root@172.16.50.$node:/etc/kubernetes \
; done
```

**node02 节点加入集群**

```
[root@node02 pkg]# kubeadm join 172.16.50.190:6443 \
--token xoy1bv.tniobqdvl7r70f3j \
--discovery-token-ca-cert-hash sha256:cc5d9b58a0482dc77bde9656946e04b0eb40ca8522b752839fa8bb449fb21a3f \
--experimental-control-plane
```

**node03 节点加入集群**

```
[root@node03 pkg]# kubeadm join 172.16.50.190:6443 \
--token xoy1bv.tniobqdvl7r70f3j \
--discovery-token-ca-cert-hash sha256:cc5d9b58a0482dc77bde9656946e04b0eb40ca8522b752839fa8bb449fb21a3f \
--experimental-control-plane
```

>  **--experimental-control-plane**  参数表示已管理节点加入集群，如果是work节点不加此参数

**再次查看集群信息**

```
[root@node01 ~]# kubectl get node
NAME     STATUS   ROLES    AGE     VERSION
node01   Ready    master   16m     v1.13.4
node02   Ready    master   2m50s   v1.13.4
node03   Ready    master   72s     v1.13.4

[root@node01 ~]# kubectl get pod --all-namespaces 
NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE
kube-system   coredns-86c58d9df4-ckc8f         1/1     Running   0          16m
kube-system   coredns-86c58d9df4-wt2dp         1/1     Running   0          16m
kube-system   kube-apiserver-node01            1/1     Running   0          15m
kube-system   kube-apiserver-node02            1/1     Running   0          3m20s
kube-system   kube-apiserver-node03            1/1     Running   0          103s
kube-system   kube-controller-manager-node01   1/1     Running   0          15m
kube-system   kube-controller-manager-node02   1/1     Running   0          3m20s
kube-system   kube-controller-manager-node03   1/1     Running   0          103s
kube-system   kube-proxy-ls95l                 1/1     Running   0          103s
kube-system   kube-proxy-q2449                 1/1     Running   0          16m
kube-system   kube-proxy-rk4rf                 1/1     Running   0          3m20s
kube-system   kube-scheduler-node01            1/1     Running   0          15m
kube-system   kube-scheduler-node02            1/1     Running   0          3m20s
kube-system   kube-scheduler-node03            1/1     Running   0          103s
kube-system   weave-net-6znrt                  2/2     Running   0          7m49s
kube-system   weave-net-r5299                  2/2     Running   1          3m20s
kube-system   weave-net-xctmb                  2/2     Running   1          103s
```



**node04 node05 node06 node07 节点加入集群**

```
kubeadm join 172.16.50.190:6443 --token xoy1bv.tniobqdvl7r70f3j --discovery-token-ca-cert-hash sha256:cc5d9b58a0482dc77bde9656946e04b0eb40ca8522b752839fa8bb449fb21a3f
```

**再次查看集群信息**

```
[root@node01 ~]# kubectl get node
NAME     STATUS     ROLES    AGE   VERSION
node01   Ready      master   29m   v1.13.4
node02   Ready      master   15m   v1.13.4
node03   Ready      master   13m   v1.13.4
node04   Ready      <none>   19s   v1.13.4
node05   NotReady   <none>   24s   v1.13.4
node06   NotReady   <none>   30s   v1.13.4
node07   NotReady   <none>   36s   v1.13.4
```

![https://github.com/fandaye/Deploy-Kubernetes/blob/v1.13.4/png/status.png]()

**token 查看方法**

```
[root@node01 ~]# kubeadm token list
TOKEN                     TTL       EXPIRES                     USAGES   
xoy1bv.tniobqdvl7r70f3j   23h       2019-03-14T02:37:38-04:00  .....
```

**ca证书sha256编码hash值查看方法**

```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
openssl dgst -sha256 -hex | sed 's/^.* //'
```



## **dashboard 部署**

**创建**

```
[root@node01 config]# kubectl create -f /data/Deploy-Kubernetes/config/kubernetes-dashboard.yaml      
secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard-admin created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-admin created
```

**记录Token 用于登录认证**

```
[root@node01 ～]# secrets=`kubectl get secrets -n kube-system | grep kubernetes-dashboard-admin | awk '{print $1}'` 
[root@node01 ~]# kubectl describe secrets/${secrets}  -n kube-system   | grep "token:"
```

**导出证书**

```
[root@node01 ～]# cat /etc/kubernetes/admin.conf | grep client-certificate-data | awk -F ': ' '{print $2}' | base64 -d > /etc/kubernetes/pki/client.crt && \
> cat /etc/kubernetes/admin.conf | grep client-key-data | awk -F ': ' '{print $2}' | base64 -d > /etc/kubernetes/pki/client.key

[root@node01 ～]# openssl pkcs12 -export -inkey /etc/kubernetes/pki/client.key -in /etc/kubernetes/pki/client.crt -out /etc/kubernetes/pki/client.pfx
```

**导入证书到浏览器**

建议使用火狐浏览器

