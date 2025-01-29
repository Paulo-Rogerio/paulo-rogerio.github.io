---
layout: post
collection: kubernetes
permalink: /kubernetes/componentes
title:  "Componentes Kubernetes"
author: Paulo Rogério
date:   2025-01-22 20:59:13 -0300
categories: [kubernetes]
published: true
---

## 🚀 Estrutura Kubernetes

- [1) Control Plane](#1-control-plane)
- [2) Data Plane](#2-data-plane)
- [3) Componetes Externo](#3-componetes-externo)
- [4) Modo Roteamento Cluster](#4-modo-roteamento-cluster)


# 1) Control Plane

São os nodes que roda o coração do kubernets com os serviços críticos. Geralmente não se executa Pods de apllicação nesses workers.

## Api Server
🔹 Ponto de Entreda ( Api ). Todas as chamadas feitas por um usuario ( kubectl ), dentro ou fora do cluster irão passar pelo api-server. Se passa pelo ***Api Server*** é uma chamada de Api, então requer autenticação, é escalável e comunica-se com **ETCD**

## Cloud Controller
🔹 Integra-se a APIs de nuvem **( AWS, Azure, GCP )**, gerencia os **Nodes**, define rota e serviço, oferece suporte a versões de recursos independentes específicos do cloud provider.

## Etcd
🔹 Armazenamento de chave-valor distribuído, armazena o estado, a configuração e os metadados do cluster. 

## Kube-Controller ( Controller Manager )
🔹 Serviço do cluster Kubernetes que controla o que será criado ou removido e onde está o recurso ou para qual node será schedulado o Pod. Além de lidar com eventos de ciclo de vida, como coleta de lixo, monitora o estado do cluster.

## Kube-Scheduler
🔹 Agendador !!! Decide o posicionamento do pod com base em recursos, regras de afinidade, taints, tolerações. Esse objeto que decide em qual **worker node** será colocado o pod, ele avalia por exemplo quanto de recurso o pod vai precisar e qual worker tem os recursos para atende-lo, caso não tenha recurso nos workers o pod ficarão como pending. Esse objeto do kubernetes baseia-se em regras.

![alt text](/images/kubernetes/componentes/estrutura-kubernetes.png)

# 2) Data Plane

São os nodes que as aplicações deployadas irão rodar.

## Kubelet
🔸 Gerencia contêineres no node worker, garante a integridade, é o objeto que se comunica com o servidor de API.

## Kube-Proxy
🔸 Criar as regras de **iptables** em cada node. Geralmente cluster gerenciados ( EKS , AKS) usam esse modo de reotamento. É o objeto que roteia o tráfego para os Pods.

## CNI  
🔸 Rede para comunicação dos Pods (Ex: Kindnet / Flannel / Calico )

## Container Runtime Interface (CRI)
🔸 Gerencia o ciclo de vida do conteiner, pull de imagens, suporta executores como Docker e containerd.

## Pods
🔸 As menores unidades implantáveis, contêm contêineres, compartilham recursos de rede e armazenamento.

# 3) Componetes Externo

## Ingress
✅ Acesso externo para aplicações ( rotas / paths / certificados ).

## Local Path
✅ Armazenamento de dados local ( workers )

## Metallb
✅ Simula um Loadbalance ( entrega um Ip no Range da Interface para um service do tipo Loadbalance ).


# 4) Modo Roteamento Cluster

### Como checar qual modo de roteamento que meu cluster está operando?

```bash
k neat <<< $(k get cm -n kube-system kube-proxy -o yaml) | grep mode
mode: iptables
```

O ***neat*** é um plugin que é instalado no seu Host usando **krew**. Esse plugin "limpa" o manifesto yaml, removendo alguns campos.

[Krew](https://krew.sigs.k8s.io/){:target="_blank"}

Outra forma de roteamento é o ***ipvs***, para checar pode-se usar o comando acima, ou conectar-se a um node e checar se a interface tem o prefixo ipvs.

### Como alterar o modo de roteamento?

***Obs.:** Antes de realiazar as mudança deve-se checar na documentação do cloud provider o suporte ao roteamento desejado.

Altere o mode para ***ipvs***

```bash
k edit cm -n kube-system kube-proxy -o yaml
```

#### Será necessário reciclar os Pods.

```bash
k rollout restart deploy -n kube-system kube-proxy
```

