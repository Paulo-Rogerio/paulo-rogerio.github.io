---
layout: post
collection: kubernetes
permalink: /kubernetes/etcd-deploy
title:  "Explorando Deploy Etcd"
author: Paulo Rogério
date:   2025-01-22 20:59:13 -0300
categories: [kubernetes]
published: true
---

## 🚀 Etcd Static Pod

- [1) Deploy Etcd Static Pod](#1-deploy-etcd-static-pod)
- [2) Investigando Manifesto Yaml Etcd](#2-investigando-manifesto-yaml-etcd)
- [3) Interagindo com Etcd](#2-interagindo-com-etcd)

#### 1) Deploy Etcd Static Pod

Static Pods são **pods** gerenciados pelo próprio Kubernetes, sendo mais específico, gerenciados pelo **kubelet**. O **kubelet** foi programado para ler qualquer manifesto injetado no diretório ***/etc/kubernetes/manifests/***. Os manifestos presentes alí são geridos pelo kubernetes.

Todos os Pods que são schedulados pelo api-server é gerido por um componente chamado **scheduler**, porém os static Pods não são gerenciados por esse componete. É o próprio **kubelet** que gerência.

Um ponto importante a lembrar é que esse característica só se aplica aos **Control Plane / Master** , os workers não tem pods geridos pelo kubelet.

Outra característica é que por ser estático, não é escalável.

```bash
root@prgs-control-plane:/# ls /etc/kubernetes/manifests/
etcd.yaml  kube-apiserver.yaml	kube-controller-manager.yaml  kube-scheduler.yaml
```

```bash
root@prgs-control-plane:/# systemctl list-units --type=service --state=active
root@prgs-control-plane:/# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; preset: enabled)
    Drop-In: /etc/systemd/system/kubelet.service.d
             └─10-kubeadm.conf, 11-kind.conf
     Active: active (running) since Tue 2025-01-07 09:15:33 UTC; 1h 19min ago
```

#### 2) Investigando Manifesto Yaml Etcd

```bash
kgp -n kube-system etcd-prgs-control-plane -o yaml
```

Comando que inicia o serviço

```bash
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://172.18.0.3:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --experimental-initial-corrupt-check=true
    - --experimental-watch-progress-notify-interval=5s
    - --initial-advertise-peer-urls=https://172.18.0.3:2380
    - --initial-cluster=prgs-control-plane=https://172.18.0.3:2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://172.18.0.3:2379
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://172.18.0.3:2380
    - --name=prgs-control-plane
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

Conectado no **Control Plane** check conectividade com a porta do etcd.

```bash
root@prgs-control-plane:/# ss -lnt | grep 2379
LISTEN 0      4096       127.0.0.1:2379       0.0.0.0:*
LISTEN 0      4096      172.18.0.3:2379       0.0.0.0:*
```

```bash
root@prgs-control-plane:/# nc -v 172.18.0.3 2379
prgs-control-plane [172.18.0.3] 2379 (?) open
```

A porta está **Listen**, mas não consigo me conectar. Porque?

```bash
curl -k https://172.18.0.3:2379
curl: (56) OpenSSL SSL_read: OpenSSL/3.0.15: error:0A00045C:SSL routines::tlsv13 alert certificate required, errno 0
```

#### 3) Interagindo com Etcd

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