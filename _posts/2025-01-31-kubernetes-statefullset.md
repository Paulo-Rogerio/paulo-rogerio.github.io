---
layout: post
collection: kubernetes
permalink: /kubernetes/statefullset
title:  "Kubernetes Statefullset"
author: Paulo Rogério
date:   2025-01-30 15:00:13 -0300
categories: [kubernetes]
published: true
---

## 🚀 Kubernetes Statfullset

- [1) Statefull Vs Stateless](https://paulo-rogerio.github.io/kubernetes/statefull-stateless){:target="_blank"}
- [2) Características Statefull](#2-características-statefull)
- [3) Como um Statefull é Exposto por um Serviço](#3-como-um-statefull-é-exposto-por-um-serviço)
- [4) Statefullset Mão na Massa](#4-statefullset-mão-na-massa)

#### 2) Características Statefull

🔸 Statefullset sempre um ***Volume*** por pod. 

🔸 Statefullset um ***Volume*** não é compartilhado entre outros Pods. 

🔸 Quando um pod morre, o pvc garante que os dados ainda estaram lá.

🔸 Os nomes sao sempre previsíveis, se meu deployment chama nginx, os nomes seriam: ( nginx-0, nginx-1 )

🔸 Mesmo que o Pod **( nginx-2 )** morra, quando ele subir novamente, somente ele irá acessar esse dados.

#### 3) Como um Statefull é Exposto por um Serviço

Um detalhe importante é quando exponho um ***Statefullset***, diferentemente de um deploymente que cria-se um service **( ClusterIP / NodePort )**, esse cara trabalha diferente.

Ele cria um ***Headless Service*** **( Diferente de um Service comum não tem IP )**, esse cara é um resolvedor de nomes que conhece todas as replicas **( Ex: nginx-1, nginx-2, nginx-3 )**, ele não tem IP. 

Esse simplesmente retorna o DNS de todos os IPs dos statefull **( Ex: nginx-1, nginx-2, nginx-3 )** e o cliente que requisitou escolhe em qual ele quer se conectar.

#### 4) Statefullset Mão na Massa