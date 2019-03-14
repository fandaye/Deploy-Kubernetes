## 如何使用cephfs

### 前提

```
# ceph fs new cephfs cephfs_metadata cephfs_data
# ceph fs ls
name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
# ceph mds stat
e29: 1/1/1 up {0=docker01-morepay-cn=up:active}
```



### 使用Ceph身份验证密钥 

```
# ceph auth get-key client.admin |base64
QVFETHVBcGEzeGxVS2hBQTIvcWxvbkFHcUZFMThSMHFCYllUSWc9PQ==
```

示例yaml 

```
kubectl create -f ceph/yaml/ceph-secret.yaml
```



### 创建`PersistentVolume`

示例yaml 

```
kubectl create -f ceph/yaml/cephfs-pv.yaml
```



### 创建`PersistentVolumeClaim`

示例yaml 

```
kubectl create -f ceph/yaml/cephfs-pvc.yaml
```



### 测试

示例yaml 

```
kubectl create -f ceph/cephfs-test.yaml
```

