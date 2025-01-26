---
layout: post
collection: kubernetes
permalink: /kubernetes/etcd-manager
title:  "Explorando Etcd"
author: Paulo Rogério
date:   2025-01-22 20:59:13 -0300
categories: [kubernetes]
published: true
---

## 🚀 Etcd Explorando Api

####  Interagindo com Etcd

A conexão foi estabelecida, por falta dos certificado. Assim como consumimos a **API** do kubernetes por meio de um certificado, para o etcd é preciso fazer a mesma coisa.

Mas onde ficam localizados os certificados?

```bash
root@prgs-control-plane:/# ls -la /etc/kubernetes/pki/etcd/
total 40
drwxr-xr-x 2 root root 4096 Jan 24 00:40 .
drwxr-xr-x 3 root root 4096 Jan 24 00:40 ..
-rw-r--r-- 1 root root 1094 Jan 24 00:40 ca.crt
-rw------- 1 root root 1679 Jan 24 00:40 ca.key
-rw-r--r-- 1 root root 1123 Jan 24 00:40 healthcheck-client.crt
-rw------- 1 root root 1679 Jan 24 00:40 healthcheck-client.key
-rw-r--r-- 1 root root 1224 Jan 24 00:40 peer.crt
-rw------- 1 root root 1675 Jan 24 00:40 peer.key
-rw-r--r-- 1 root root 1224 Jan 24 00:40 server.crt
-rw------- 1 root root 1675 Jan 24 00:40 server.key
```

Como interagir com o cluster etcd?

```bash
root@prgs-control-plane:/# apt install etcd-client
root@prgs-control-plane:/# openssl x509 -in /etc/kubernetes/pki/etcd/ca.crt -text
```

Assim como CA do Kubernetes o ***ETCD*** possui sua própria Unidade Certificadora.

```bash
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 7578045755675238448 (0x692aa076eeade030)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = etcd-ca
        Validity
            Not Before: Jan 24 00:35:16 2025 GMT
            Not After : Jan 22 00:40:16 2035 GMT
        Subject: CN = etcd-ca
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:b1:25:90:58:ca:26:4b:59:ae:aa:b4:0e:1a:81:
                    17:d6:8d:ed:e6:45:af:46:06:e9:2b:ce:c1:d4:22:
```

```bash
root@prgs-control-plane:/# export certs="/etc/kubernetes/pki/etcd"
root@prgs-control-plane:/# ETCDCTL_API=3 etcdctl --cacert=${certs}/ca.crt --cert=${certs}/server.crt --key=${certs}/server.key member list --write-out=table

+------------------+---------+--------------------+-------------------------+-------------------------+------------+
|        ID        | STATUS  |        NAME        |       PEER ADDRS        |      CLIENT ADDRS       | IS LEARNER |
+------------------+---------+--------------------+-------------------------+-------------------------+------------+
| 23da9c3f2594532a | started | prgs-control-plane | https://172.18.0.3:2380 | https://172.18.0.3:2379 |      false |
+------------------+---------+--------------------+-------------------------+-------------------------+------------+
```

Outra forma de conectar-se.

```bash
root@prgs-control-plane:/# cat > /root/etcdctl.env <<EOF
export ETCDCTL_API=3
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key
EOF
root@prgs-control-plane:/# source /root/etcdctl.env
root@prgs-control-plane:/# etcdctl member list --write-out=table

+------------------+---------+--------------------+-------------------------+-------------------------+------------+
|        ID        | STATUS  |        NAME        |       PEER ADDRS        |      CLIENT ADDRS       | IS LEARNER |
+------------------+---------+--------------------+-------------------------+-------------------------+------------+
| 23da9c3f2594532a | started | prgs-control-plane | https://172.18.0.3:2380 | https://172.18.0.3:2379 |      false |
+------------------+---------+--------------------+-------------------------+-------------------------+------------+
```

#### Health Check

```bash
root@prgs-control-plane:/# etcdctl endpoint health --endpoints="https://172.18.0.3:2379"
https://172.18.0.3:2379 is healthy: successfully committed proposal: took = 11.292646ms
```


#### Endpoint Status

```bash
root@prgs-control-plane:/# etcdctl endpoint status --write-out=table --endpoints="https://172.18.0.3:2379"
+-------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|        ENDPOINT         |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://172.18.0.3:2379 | 23da9c3f2594532a |  3.5.16 |  5.4 MB |      true |      false |         2 |      11482 |              11482 |        |
+-------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

#### Read Data

```bash
root@prgs-control-plane:/# etcdctl --endpoints=https://172.18.0.3:2379 get / --prefix

/registry/apiextensions.k8s.io/customresourcedefinitions/bgpadvertisements.metallb.io
{"kind":"CustomResourceDefinition","apiVersion":"apiextensions.k8s.io/v1beta1","metadata":{"name":"bgpadvertisements.metallb.io","uid":"c282d792-d32f-41ab-be49-5080c899ed25","generation":1,"creationTimest
amp":"2025-01-24T00:41:54Z","labels":{"app.kubernetes.io/managed-by":"Helm"},"annotations":{"controller-gen.kubebuilder.io/version":"v0.16.3","meta.helm.sh/release-name":"metallb","meta.helm.sh/release-na
mespace":"metallb-system"},"managedFields":[{"manager":
```