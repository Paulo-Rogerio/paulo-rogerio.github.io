---
layout: post
collection: kubernetes
permalink: /kubernetes/provisioner-kubernetes-kind
title:  "Provisionar Cluster Kubernetes Usando Kind"
author: Paulo Rogério
date:   2025-01-19 11:39:00 -0300
categories: [kubernetes]
published: true
---


# 🚀 Kubernetes Kind

- [1) Pre-Requisitos](#1-pre-requisitos)
  - [1.1) Colima](#11-colima)
  - [1.2) Kind](#12-kind)
  - [1.3) Kubectl](#13-kubectl)
  - [1.4) Helm](#14-helm)    
- [2) Repositório](#2-repositório)
- [3) Criando Cluster](#3-criando-cluster)
- [4) Check Cluster](#4-check-cluster)
- [5) Manifestos](#5-manifestos)
- [6) Destroy Cluster](#6-destroy-cluster)

#### 1) Pre-Requisitos

O Kind é uma ferramenta para rodar Kubernetes Local ***Kubernetes in Docker***, para executá-lo é necessário apenas do **docker** 
rodando em sua máquina. Se você roda seu Host ***Windows ou MacOS*** verás com o tempo que o Docker Desktop contém limitações. 

Algumas das limitações que enfrentei foi ao usar o ***Metallb***, onde os pacotes oriundos do LoadBalancer não conseguiam ser roteados para dentro do services do Kubernetes e consequententemente não entregue aos pods.

O que de fato me fez migrar para o **Colima** foi por ser uma plataforma bem enxuta e leve para rodar.


## 1.1) Colima

O projeto lima-vm ***Linux Virtual Machines*** usa máquinas virtuais Linux para fazer compartilhamento automático de arquivo e encaminhamento de portas.

[Colima](https://github.com/abiosoft/colima){:target="_blank"}

```bash
brew install colima
```

## 1.2) Kind

Executável que roda em sua máquina para provisionar Kubernetes dentro do docker.

[Kind](https://kind.sigs.k8s.io/){:target="_blank"}

```bash
brew install kind
```

## 1.3) Kubectl

Instale o binário ***kubectl*** para gerenciar o cluster kubernetes.

```bash
brew install kubernetes-cli
```

## 1.4) Helm

Instale o binário responsável por gerenciar pacotes dentro de um cluster Kubernetes. 

[Helm](https://helm.sh/){:target="_blank"}

```bash
brew install helm
```

#### 2) Repositório

Após garantir que todos os executáveis estejam presente no hosts, clone o repositório.

[Repositório](https://github.com/Paulo-Rogerio/provisioner-kubernetes){:target="_blank"}

```bash
git clone https://github.com/Paulo-Rogerio/provisioner-kubernetes.git
```

#### 3) Criando Cluster

```bash
cd provisioner-kubernetes/kind
ls -la
drwxr-xr-x  9 paulo  staff   288B 18 Jan 07:44 k8s
-rwxr-xr-x  1 paulo  staff   831B 18 Jan 09:33 start.sh
-rwxr-xr-x  1 paulo  staff   192B 17 Jan 08:25 stop.sh
```


```bash
sh start.sh
```

Ao executar o script **start.sh**, será necessário informar o **NOME DO CLUSTER**, como mostrado na imagem abaixo. Nesse exemplo o nome do cluster chama-se: ***prgs*** 

![alt text](/images/kubernetes/provisioner-kubernetes-kind/01-kind.png)

Como o colima é um Linux rodando como se fosse uma VM, ele cria uma Rede isolada com um range de IP diferente. Sendo assim,é necessário criar rotas no seu Host **MacOS** para permitir trafegar dados nesse Range de Ips gerido pelo **Colima**. 

O script requer ***interação do usuário*** ao adicionar a rota, como isso requer previlégios de root será solicitado a **SENHA DO USUÁRIO**, conforme mostra a imagem abaixo.

![alt text](/images/kubernetes/provisioner-kubernetes-kind/02-kind.png)

Ao término da execução do script inicia-se um ***Deploy de Teste** para checar se **Metallb** está funcional e após o teste remove-se o **deployment**.

![alt text](/images/kubernetes/provisioner-kubernetes-kind/03-kind.png)

#### 4) Check Cluster

```bash
➜  kind git:(main) k get nodes
NAME                 STATUS   ROLES             AGE   VERSION
prgs-control-plane   Ready    control-plane     22m   v1.31.2
prgs-worker          Ready    worker-apps       22m   v1.31.2
prgs-worker2         Ready    worker-postgres   22m   v1.31.2
```

```bash
➜  kind git:(main) k get pods -A
NAMESPACE            NAME                                         READY   STATUS      RESTARTS   AGE
ingress-nginx        ingress-nginx-admission-create-mr6mh         0/1     Completed   0          19m
ingress-nginx        ingress-nginx-admission-patch-mp2x8          0/1     Completed   1          19m
ingress-nginx        ingress-nginx-controller-5f4f4d9787-rb5jx    1/1     Running     0          19m
kube-system          coredns-7c65d6cfc9-cvkqp                     1/1     Running     0          22m
kube-system          coredns-7c65d6cfc9-xhtm6                     1/1     Running     0          22m
kube-system          etcd-prgs-control-plane                      1/1     Running     0          22m
kube-system          kindnet-9s7t5                                1/1     Running     0          22m
kube-system          kindnet-fhbdm                                1/1     Running     0          22m
kube-system          kindnet-p4tl5                                1/1     Running     0          22m
kube-system          kube-apiserver-prgs-control-plane            1/1     Running     0          22m
kube-system          kube-controller-manager-prgs-control-plane   1/1     Running     0          22m
kube-system          kube-proxy-55cxr                             1/1     Running     0          22m
kube-system          kube-proxy-7zqds                             1/1     Running     0          22m
kube-system          kube-proxy-vpvb6                             1/1     Running     0          22m
kube-system          kube-scheduler-prgs-control-plane            1/1     Running     0          22m
kube-system          metrics-server-7bb58f4dcb-mdgzl              1/1     Running     0          17m
local-path-storage   local-path-provisioner-57c5987fd4-f7pf2      1/1     Running     0          22m
metallb-system       metallb-controller-dc88974b6-6ks69           1/1     Running     0          21m
metallb-system       metallb-speaker-6qpnd                        4/4     Running     0          21m
metallb-system       metallb-speaker-tpgg8                        4/4     Running     0          21m
metallb-system       metallb-speaker-z48g2                        4/4     Running     0          21m
```

#### 5) Manifestos

O **metrics-server** é instalado por meio de um **Helm Chart**, por padrão ele é configurado para interagir com a **API do Kubernetes** por meio de conexão segura **TLS**, sendo assim , salvamos o values para modificar o comportamento do **Chart Metrics Server** para permiti-lo interagir com API do Kubernetes por meio de um certificado auto assinado. 

[Helm Values - Metrics ](https://artifacthub.io/packages/helm/metrics-server/metrics-server?modal=values){:target="_blank"}

Se precisar atualizar o **Chart Values** do **Metrics Server**, lembre-se de ajustar o parâmetro abaixo para garantir essa comunicação.

```bash
vi values/metric-server.yaml
```

Adicione o parâmetro **--kubelet-insecure-tls**

```bash
defaultArgs:
  - --cert-dir=/tmp
  - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
  - --kubelet-use-node-status-port
  - --kubelet-insecure-tls
  - --metric-resolution=15s
```

#### 6) Destroy Cluster

Após terminar seus estudos, pode desligar o cluster juntamente com o colima executando o comando **stop.sh**, conforme mostra a imagem abaixo:

![alt text](/images/kubernetes/provisioner-kubernetes-kind/04-kind.png)