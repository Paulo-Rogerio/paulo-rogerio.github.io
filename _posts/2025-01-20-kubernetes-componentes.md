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

![alt text](/images/kubernetes/componentes/estrutura-kubernetes.png)

## Ingress
Acesso externo para aplicações ( rotas / paths / certificados ).

## CoreDNS
Realiza a resolução de Nomes.

## ETCD
Banco de Dados ( Chave : Valor )

## CNI  
Rede para comunicação dos Pods (Ex: Kindnet / Flannel / Calico )

## Api Server
Ponto de Entrega ( Api ). Todas as chamadas sejam feitas por um usuario ( kubectl ), dentro ou fora do cluster irão passar pelo api-server. Se passa pelo ***Api Server*** é uma chamada de Api, então requer autenticação.

## Kube-Controller
Serviço do cluster Kubernetes que controla o que será criado ou removido e onde está o recurso ou para qual node será schedulado o Pod.

## Kube-Proxy
Criar as regras de **iptables** em cada node. Geralmente cluster gerenciados ( EKS , AKS) usam esse modo de reotamento.

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

Será necessário reciclar os Pods.

```bash
k rollout restart deploy -n kube-system kube-proxy
```

## Kube-Scheduller
Objeto que decide em qual **worker** será colocado o pod, ele avalia por exemplo quanto de recurso o pod vai precisar
e qual worker tem os recursos para atende-lo, caso não tenha recurso nos workers o pod fica como pending.

Esse objeto do kubernetes baseia-se em regras.

## Local Path
Armazenamento de dados local ( workers )

## Metallb
Simula um Loadbalance ( entrega um Ip no Range da Interface para um service do tipo Loadbalance ).