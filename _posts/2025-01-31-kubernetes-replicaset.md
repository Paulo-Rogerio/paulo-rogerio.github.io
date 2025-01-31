---
layout: post
collection: kubernetes
permalink: /kubernetes/replicaset
title:  "Kubernetes Replicaset"
author: Paulo Rogério
date:   2025-01-30 15:00:13 -0300
categories: [kubernetes]
published: true
---

## 🚀 Kubernetes Replicaset

- [1) Replicaset](#1-replicaset)
- [2) Como Replicaset Sabe Qual Pod Gerenciar?](#2-como-replicaset-sabe-qual-pod-gerenciar)
- [3) Replicaset Mão na Massa](#3-replicaset-mão-na-massa)

#### 1) Replicaset

Vimos no conteúdo de **Deployments** que as réplicas são gerenciadas pelo **Replicaset**, que ao fazer um ***Rolling Update*** um novo **Replicaset** é criado. Na grande maioria dos casos o deployment ocorrerá da seguinte forma:

- **Replicaset Old** ( Encerra um Pod ), a medida que o **Replicaset New** ( Cria um novo Pod ).

- Existem outras formas que esse rolout ocorre, ex: canary que veremos logo logo.

![](/images/kubernetes/deployment-replicaset/deployment.png)

#### 2) Como Replicaset Sabe Qual Pod Gerenciar?

As label tem o proposito de nortear o **replicaset** para que ele identifique qual **Pod** ele irá gerenciar. Ela tambem é usada para direcionar que um pod possa deployar em determinado worker. O objeto responsável por fazer esse match é o ***matchLabels***.

Mostrar labels...

```bash
➜  kind git:(main) k get deployment --show-labels
```

Filtrando por label...

```bash
➜  kind git:(main) k get pod -l app=nginx
```

Exemplo manifesto yaml deployment...

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
    environment: development
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
      - image: httpd
        name: nginx
```

Labels válidas apenas no escopo de deployment e não para pods...

```bash
➜  kind git:(main) k get deployment --show-labels
NAME          READY   UP-TO-DATE   AVAILABLE   AGE   LABELS
nginx          1/1        1            1       27m   app=nginx,environment=development
```

Observer que no escopo dos Pods , essa label não é acessível...

```bash
➜  kind git:(main) k get pod --show-labels
NAME                           READY   STATUS    RESTARTS   AGE   LABELS
nginx-58d5f87bb5-pcznl   1/1     Running   0          20m   app=nginx,pod-template-hash=58d5f87bb5
```

Essas labels podem ser utilizadas nas condições de filtros...

```bash
➜  kind git:(main) k delete pod -n kube-system -l k8s-app=kube-dns
```

#### 3) Replicaset Mão na Massa

```bash
➜  kind git:(main) k create deployment --image=nginx --replicas=3 nginx
deployment.apps/nginx created
```

List Replicaset ...

```bash
➜  kind git:(main) k get replicasets
NAME               DESIRED   CURRENT   READY   AGE
nginx-5869d7778c   3         3         3       36s
```

Observer que o **ID** do Pod é herdado do **ID** do Replicaset.

```bash
➜  kind git:(main) kgp
NAME                     READY   STATUS    RESTARTS   AGE
nginx-5869d7778c-k2dnr   1/1     Running   0          53s
nginx-5869d7778c-t64x7   1/1     Running   0          53s
nginx-5869d7778c-zj2nt   1/1     Running   0          53s
```

O que acontece quando troco a imagem de um deployment? Vamos baixar o manifesto **yaml** e substituir a imagem do **nginx** por **httpd**.

```bash
➜  kind git:(main) k neat <<< $(k get deployment nginx -o yaml) | sed 's/image: nginx/image: httpd/' <<EOF | kaf -
EOF
deployment.apps/nginx configured
```

O replicaset zerados é mantido para fins de restore..

```bash
➜  kind git:(main) k get rs
NAME               DESIRED   CURRENT   READY   AGE
nginx-5869d7778c   0         0         0       8m51s
nginx-69bf56d45f   3         3         3       6m55s
```

Check o histórico de Deployments....

```bash
➜  kind git:(main) k rollout history deployment/nginx
deployment.apps/nginx
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

Esse Replicaset **"nginx-5869d7778c"** representa a Revision **"1"**.

Esse Replicaset **"nginx-69bf56d45f"** representa a Revision **"2"**.

Como fazer o Rollout?

```bash
➜  kind git:(main) k rollout undo deployment/nginx --to-revision=1
deployment.apps/nginx rolled back
```

```bash
➜  kind git:(main) k get rs
NAME               DESIRED   CURRENT   READY   AGE
nginx-5869d7778c   3         3         3       14m
nginx-69bf56d45f   0         0         0       12m
```

É uma boa prática acompanhar o processo de **Rollout**, geralmente isso pode ser feito em outro terminal...

```bash
➜  kind git:(main) watch kubectl rollout status deployment/nginx
```

Caso tenha algum problemas com os Pod, pode-se reinicia-los...

Em outro terminal monitore as atividades das criações dos Pods...

```bash
➜  kind git:(main) k get pods -w
```

Reiniciando os Pods...

```bash
➜  kind git:(main) k rollout restart deployment/nginx
deployment.apps/nginx restarted
```
