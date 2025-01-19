---
layout: post
collection: kubernetes
permalink: /kubernetes/provisioner-kubernetes-vagrant
title:  "Provisionar Cluster Kubernetes Usando Vagrant"
author: Paulo Rogério
date:   2025-01-19 19:24:00 -0300
categories: [kubernetes]
published: true
---

# 🚀 Kubernetes Kubadm

- [1) Pre-Requisitos](#1-pre-requisitos)
  - [1.1) VirtualBox](#11-virtualbox)
  - [1.2) Vagrant](#12-vagrant)
- [2) Repositório](#2-repositório)
- [3) Criando VMs](#3-criando-vms)
- [4) Check Cluster](#4-check-cluster)
- [5) Destroy Cluster](#5-destroy-cluster)

#### 1) Pre-Requisitos

O **Virtualbox** é um virtualizador que emula uma **Virtual Machine**.

O **Vagrant** é uma ferramenta para provisionar conteúdo dentro de um Host, todas as instruções é realizada dentro do **Vagrantfile**.

## 1.1) VirtualBox

O projeto ***VirtualBox*** é distribuiddo pela **Oracle**, podendo ser executado em qualquer plataforma. Baixe e execute em seu Host. 

[VirtualBox](https://www.virtualbox.org/){:target="_blank"}

## 1.1) Vagrant

O projeto ***Vagrant*** é distribuiddo pela **HashCorp**. Baixe o binário para seu Host.

[Vagrant](https://www.vagrantup.com/){:target="_blank"}

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/hashicorp-vagrant
```

#### 2) Repositório

Após garantir que todos os executáveis estejam presente no hosts, clone o repositório.

[Repositório](https://github.com/Paulo-Rogerio/provisioner-kubernetes){:target="_blank"}

```bash
git clone https://github.com/Paulo-Rogerio/provisioner-kubernetes.git
```

#### 3) Criando VMs

```bash
cd provisioner-kubernetes/vagrant
ls -la
-rw-r--r--@ 1 paulo  staff   2,6K 18 Jan 11:16 Vagrantfile
-rwxr-xr-x  1 paulo  staff    21B 17 Jan 08:24 connect-master.sh
-rwxr-xr-x  1 paulo  staff    21B 17 Jan 08:24 connect-worker-1.sh
-rwxr-xr-x  1 paulo  staff    21B 17 Jan 08:24 connect-worker-2.sh
-rwxr-xr-x  1 paulo  staff    53B 17 Jan 08:24 create.sh
-rwxr-xr-x  1 paulo  staff    74B 17 Jan 08:24 destroy.sh
drwxr-xr-x  7 paulo  staff   224B 17 Jan 08:24 scripts
-rwxr-xr-x  1 paulo  staff    35B 17 Jan 08:24 start.sh
-rwxr-xr-x  1 paulo  staff    33B 17 Jan 08:24 stop.sh
drwxr-xr-x  5 paulo  staff   160B 17 Jan 08:24 vagrant-files
```

Ao iniciar o script de ***create.sh*** é iniciado o processo de Deploy.

```bash
➜  vagrant git:(main) sh create.sh
Bringing machine 'master-1' up with 'virtualbox' provider...
Bringing machine 'worker-1' up with 'virtualbox' provider...
Bringing machine 'worker-2' up with 'virtualbox' provider...
==> master-1: Importing base box 'generic/ubuntu2204'...
...
...
...
```

Como mostrado nas imagem abaixo será provisionado as instâncias e o cluster ***Kubernetes***.

![alt text](/images/kubernetes/provisioner-kubernetes-vagrant/01-vagrant.png)

![alt text](/images/kubernetes/provisioner-kubernetes-vagrant/02-vagrant.png)

![alt text](/images/kubernetes/provisioner-kubernetes-vagrant/03-vagrant.png)

![alt text](/images/kubernetes/provisioner-kubernetes-vagrant/04-vagrant.png)


#### 4) Check Cluster

Validando o cluster, passando o arquivo **config** como parâmetro.

```bash
➜  vagrant git:(main) ls vagrant-files/configs/
config  join.sh

➜  vagrant git:(main) k get nodes --kubeconfig=./vagrant-files/configs/config
NAME       STATUS   ROLES           AGE     VERSION
master-1   Ready    control-plane   15m     v1.32.1
worker-1   Ready    worker          6m54s   v1.32.1
worker-2   Ready    worker          81s     v1.32.1
```

Validando o cluster, passando o **config** como variável de ambiente.

```bash
➜  vagrant git:(main) KUBECONFIG="./vagrant-files/configs/config" k get nodes
NAME       STATUS   ROLES           AGE     VERSION
master-1   Ready    control-plane   16m     v1.32.1
worker-1   Ready    worker          7m30s   v1.32.1
worker-2   Ready    worker          117s    v1.32.1 
```

```bash
➜  vagrant git:(main) k get pods -A --kubeconfig=./vagrant-files/configs/config
NAMESPACE        NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx    ingress-nginx-admission-create-gk2s6        0/1     Completed   0          14m
ingress-nginx    ingress-nginx-admission-patch-j27w4         0/1     Completed   0          14m
ingress-nginx    ingress-nginx-controller-6d59d95796-tjd95   1/1     Running     0          14m
kube-flannel     kube-flannel-ds-66h6x                       1/1     Running     0          2m35s
kube-flannel     kube-flannel-ds-8h8q6                       1/1     Running     0          8m8s
kube-flannel     kube-flannel-ds-gp9jv                       1/1     Running     0          16m
kube-system      coredns-668d6bf9bc-6xg4v                    1/1     Running     0          16m
kube-system      coredns-668d6bf9bc-strh9                    1/1     Running     0          16m
kube-system      etcd-master-1                               1/1     Running     0          16m
kube-system      kube-apiserver-master-1                     1/1     Running     0          16m
kube-system      kube-controller-manager-master-1            1/1     Running     0          16m
kube-system      kube-proxy-5lxgg                            1/1     Running     0          2m35s
kube-system      kube-proxy-khs2l                            1/1     Running     0          8m8s
kube-system      kube-proxy-rxvrl                            1/1     Running     0          16m
kube-system      kube-scheduler-master-1                     1/1     Running     0          16m
kube-system      metrics-server-5bb596c8b8-7fzzg             1/1     Running     0          16m
metallb-system   metallb-controller-8474b54bc4-r84tr         1/1     Running     0          15m
metallb-system   metallb-speaker-pp9dr                       4/4     Running     0          15m
metallb-system   metallb-speaker-sgst6                       4/4     Running     0          7m26s
metallb-system   metallb-speaker-xfqxc                       4/4     Running     0          112s
```

#### 5) Destroy Cluster

Caso queira apenas desligar as VMs, execute o **stop.sh**.

```bash
➜  vagrant git:(main) ✗ sh stop.sh
==> worker-2: Attempting graceful shutdown of VM...
==> worker-1: Attempting graceful shutdown of VM...
==> master-1: Attempting graceful shutdown of VM...
```

Para destruir as VMs...

```bash
➜  vagrant git:(main) ✗ sh destroy.sh
==> worker-2: Destroying VM and associated drives...
==> worker-1: Destroying VM and associated drives...
==> master-1: Destroying VM and associated drives...
```
