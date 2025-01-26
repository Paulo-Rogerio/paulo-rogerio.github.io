---
layout: post
collection: kubernetes
permalink: /kubernetes/etcd-backup-restore
title:  "Backup Restore Etcd"
author: Paulo Rogério
date:   2025-01-22 20:59:13 -0300
categories: [kubernetes]
published: true
---

## 🚀 Etcd Manutenção

- [1) Check Service Etcd](#1-check-service-etcd)
- [2) Start Pod](#2-start-pod)
- [3) Backup Etcd](#3-backup-etcd)
- [4) Check Backup Etcd](#4-check-backup-etcd)
- [5) Restore Backup Etcd](#5-restore-backup-etcd)
- [6) Check Status Cluster](#6-check-status-cluster)

#### 1) Check Service Etcd

```bash
kgn --kubeconfig=./vagrant-files/configs/config
NAME       STATUS   ROLES           AGE   VERSION
master-1   Ready    control-plane   22m   v1.31.0
worker-1   Ready    worker          14m   v1.31.0
```

#### 2) Start Pod

```bash
kubectl run --image nginx nginx --kubeconfig=./vagrant-files/configs/config
```

```bash
kgp --kubeconfig=./vagrant-files/configs/config
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          4s
```

#### 3) Backup Etcd

Conecte no ***control-plane*** e instale os pacotes a seguir.

```bash
vagrant ssh master-1
vagrant@master-1:/home/vagrant$ sudo su 
root@master-1:/home/vagrant# apt update && apt install etcd-client vim -y
```

Check o status do serviço etcd.

```bash
root@master-1:/home/vagrant# cat > /root/etcdctl.env <<EOF
export ETCDCTL_API=3
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key
EOF
root@master-1:/home/vagrant# 
root@master-1:/home/vagrant# 
root@master-1:/home/vagrant# source /root/etcdctl.env
root@master-1:/home/vagrant# etcdctl member list --write-out=table

+-----------------+---------+----------+----------------------------+----------------------------+
|       ID        | STATUS  |   NAME   |         PEER ADDRS         |        CLIENT ADDRS        |
+-----------------+---------+----------+----------------------------+----------------------------+
| f303f204881938b | started | master-1 | https://192.168.56.56:2380 | https://192.168.56.56:2379 |
+-----------------+---------+----------+----------------------------+----------------------------+
```

Iniciando Backup...

```bash
root@master-1:/home/vagrant# mkdir -p /opt/backup
root@master-1:/home/vagrant# source /root/etcdctl.env
root@master-1:/home/vagrant# etcdctl --endpoints=https://192.168.56.56:2379 snapshot save /opt/backup/snapshot-$(date +%Y%m%d).db

2025-01-26 15:13:03.709939 I | clientv3: opened snapshot stream; downloading
2025-01-26 15:13:03.873847 I | clientv3: completed snapshot read; closing
Snapshot saved at /opt/backup/snapshot-20250126.db
root@master-1:/home/vagrant#
root@master-1:/home/vagrant#
root@master-1:/home/vagrant# du -hs /opt/backup/snapshot-20250126.db
4.5M	/opt/backup/snapshot-20250126.db
```

#### 4) Check Backup Etcd

```bash
root@master-1:/home/vagrant# etcdctl snapshot status /opt/backup/snapshot-$(date +%Y%m%d).db -w table
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 4002ab87 |     3506 |       1307 |     4.6 MB |
+----------+----------+------------+------------+
```

#### 5) Restore Backup Etcd

Iniciando o processo de restore...

```bash
root@master-1:/home/vagrant# mkdir -p /opt/restore
root@master-1:/home/vagrant# source /root/etcdctl.env
root@master-1:/home/vagrant# etcdctl --endpoints=https://192.168.56.56:2379 --data-dir="/opt/restore/etcd" snapshot restore /opt/backup/snapshot-$(date +%Y%m%d).db
2025-01-26 15:19:36.749138 I | mvcc: restore compact to 2787
2025-01-26 15:19:36.781174 I | etcdserver/membership: added member 8e9e05c52164694d [http://localhost:2380] to cluster cdf818194e3a8c32
```


Após restaurar devemos ajustar o manufesto yaml que o pod do etcd lê. Como ele é um ***Static Pod***, suas configuraçõe ficam em ***/etc/kubernetes/manifests/etcd.yaml***.

Localize as entradas:

```bash
--data-dir=/var/lib/etcd
- mountPath: /var/lib/etcd
path: /var/lib/etcd
```

Segue o exemplo do arquivo original...

```yaml
spec:
  containers:
  - command:
    - etcd
    - --data-dir=/var/lib/etcd
...
...
...
    volumeMounts:
    - mountPath: /var/lib/etcd
      name: etcd-data
...
...
...
  volumes:
  - hostPath:
      path: /var/lib/etcd
      type: DirectoryOrCreate
    name: etcd-data
status: {}
```

Substitua as entradas acima por:

```bash
--data-dir=/opt/restore/etcd
- mountPath: /opt/restore/etcd
path: /opt/restore/etcd
```

Segue o exemplo ....

```yaml
spec:
  containers:
  - command:
    - etcd
    - --data-dir=/opt/restore/etcd
...
...
...
    volumeMounts:
    - mountPath: /opt/restore/etcd
      name: etcd-data
...
...
...
  volumes:
  - hostPath:
      path: /opt/restore/etcd/
      type: DirectoryOrCreate
    name: etcd-data
status: {}
```

Restart Kubelet...

```bash
root@master-1:/home/vagrant# systemctl restart kubelet
root@master-1:/home/vagrant# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Sun 2025-01-26 15:30:43 -03; 13s ago
       Docs: https://kubernetes.io/docs/
   Main PID: 25950 (kubelet)
      Tasks: 12 (limit: 2220)
     Memory: 51.2M
        CPU: 3.095s
```

#### 6) Check Status Cluster

```bash
root@master-1:/home/vagrant# etcdctl member list --write-out=table
+------------------+---------+----------+-----------------------+----------------------------+
|        ID        | STATUS  |   NAME   |      PEER ADDRS       |        CLIENT ADDRS        |
+------------------+---------+----------+-----------------------+----------------------------+
| 8e9e05c52164694d | started | master-1 | http://localhost:2380 | https://192.168.56.56:2379 |
+------------------+---------+----------+-----------------------+----------------------------+     
```

Check o status dos nodes...

```bash
root@master-1:/home/vagrant# kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
master-1   Ready    control-plane   45m   v1.31.0
worker-1   Ready    worker          37m   v1.31.0
```

Check status dos Pods

```bash
root@master-1:/home/vagrant# kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          22m
```

Check os manifestos lido pelo Static Pod ( etcd )...


```bash
root@master-1:/home/vagrant# cat /etc/kubernetes/manifests/etcd.yaml | grep restore
    - --data-dir=/opt/restore/etcd
    - mountPath: /opt/restore/etcd
      path: /opt/restore/etcd/
```
