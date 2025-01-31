---
layout: post
collection: kubernetes
permalink: /kubernetes/deployment
title:  "Kubernetes Deployment"
author: Paulo Rogério
date:   2025-01-30 15:00:13 -0300
categories: [kubernetes]
published: true
---

## 🚀 Kubernetes Deployment

- [1) O que é um Deployment](#1-o-que-é-um-deployment)
- [2) Deployment Mão na Massa](#2-deployment-mão-na-massa)

#### 1) O que é um Deployment

Um deployment é utilizado para aplicações ***stateless ( Aplicação escaláveis )***. O deployments orquestra os **replicaset**, e são os **replicaset** que cria os pods.

Para que esse entendimento fique claro, imagine um **manifesto ( Deployment )**, ele criará **1 replicaset**, se eu precisar fazer um ***Rolling Update ( Termo usado quando vou atualizar meus pods )***, atualizando a imagem, o kubernetes criará outro **replicaset**. Na grande maioria dos casos o deployment ocorrerá da seguinte forma:
- **Replicaset Old** ( Encerra um Pod ), a medida que o **Replicaset New** ( Cria um novo Pod ).

![](/images/kubernetes/deployment-replicaset/deployment.png)

#### 2) Deployment Mão na Massa

Executando via linha de comando...

```bash
➜  kind git:(main) k create deployment --image=nginx nginx
deployment.apps/nginx created
```

Gerar o manifesto yaml caso não se recorde a sintexe...

```bash
➜  kind git:(main) k neat <<< $(k create deployment --image=nginx nginx --dry-run=client -o yaml)
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

O Deployment garante que os Pods sempre estejam Running, mesmo que o pod seja encerrado, um novo será criado.

```bash
➜  kind git:(main) kgp
NAME                     READY   STATUS    RESTARTS   AGE
nginx-5869d7778c-5j97l   1/1     Running   0          70s
```

```bash
➜  kind git:(main) kdp nginx-5869d7778c-5j97l
pod "nginx-5869d7778c-5j97l" deleted
```

Observe que o **ID** do Pod foi alterado, isso porque o Pod antigo morreu e o novo foi criado.

```bash
➜  kind git:(main) kgp
NAME                     READY   STATUS    RESTARTS   AGE
nginx-5869d7778c-gwpcn   1/1     Running   0          23s
```

```bash
➜  kind git:(main) k scale deployment nginx --replicas 3
deployment.apps/nginx scaled
```

```bash
➜  kind git:(main) kgp
NAME                     READY   STATUS    RESTARTS   AGE
nginx-5869d7778c-gwpcn   3/3     Running   0          2m49s
nginx-5869d7778c-t89w9   3/3     Running   0          28s
nginx-5869d7778c-z4vbz   3/3     Running   0          38s
```

Ao encerrar todos os Pods o replicaset fica zerado...

```bash
➜  kind git:(main) k scale deployment nginx --replicas 0
deployment.apps/nginx scaled
```

```bash
➜  kind git:(main) k get rs
NAME               DESIRED   CURRENT   READY   AGE
nginx-5869d7778c   0         0         0       21m
```

```bash
➜  kind git:(main) k delete deployment nginx
deployment.apps "nginx" deleted
```