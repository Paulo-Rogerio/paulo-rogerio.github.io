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
