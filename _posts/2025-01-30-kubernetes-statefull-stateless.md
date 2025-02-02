---
layout: post
collection: kubernetes
permalink: /kubernetes/statefull-stateless
title:  "Kubernetes Statefull Vs Stateless"
author: Paulo Rogério
date:   2025-01-30 15:00:13 -0300
categories: [kubernetes]
published: true
---

## 🚀 Kubernetes Statefull Vs Stateless

- [1) Statefull Vs Stateless](#1-statefull-vs-stateless)
- [2) Deployment Vs Statefullset Vs Daemonset Vs Replicaset](#2-deployment-vs-statefullset-vs-daemonset-vs-replicaset)

#### 1) Statefull Vs Stateless

O que vai determinar se a aplicação é ***Statless*** é a capacidade que ela tem de ser escalável.

Já uma aplicação ***StateFull*** depende totalmente do ***estado da aplicação ( sessão )***. 

Ex: Imagina uma aplicação que autentica usuários, e determinado usuário logado em uma seção conectado em um **Pod A**, ao ser redirecionado ao **Pod B**, essa sessão autenticada pode não funcionar. A aplicação depende totalmente de estado.

Claro que esse cenário poderia facilmente ser contornado se a sessão do usuário estiver armazenada em um Redis.

Em regras gerais quanto mais a aplicação **usa/depende** do sistema de arquivos, mais ***Statefull*** ela é.

#### 2) Deployment Vs Statefullset Vs Daemonset Vs Replicaset

🔸 **Deployment** responsável por gerenciar os **Replicaset**.

🔸 **Statefullset** é gerido pelo **kube-scheduler**, tem os nomes do pods com um prefixo **ex: jenkins-0**. É um deployment controlado. Ele sempre segue a ordem de subir um e matar um Pod.  

🔸 **Daemonset** não se define o numero de replicas. O Daemonset será igual ao numero de nodes de um cluster. Daemonset não passa pelo **kube-scheduler**.
