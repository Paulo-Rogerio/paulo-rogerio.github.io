---
layout: post
collection: kubernetes
permalink: /kubernetes/deployment-maxsurge
title:  "Kubernetes Maxsurge / Maxunavailable"
author: Paulo Rogério
date:   2025-02-01 15:00:13 -0300
categories: [kubernetes]
published: true
---

## 🚀 Kubernetes Deployment Maxsurge / Maxunavailable

- [1) Deployment Maxsurge / Maxunavailable](#1-deployment-maxsurge--maxunavailable)
- [2) Maxsurge / Maxunavailable Mão na Massa](#2-maxsurge--maxunavailable-mão-na-massa)

#### 1) Deployment Maxsurge / Maxunavailable

Por padrao o rollout sobe **25% dos pods novo ( nova release )** e a medida que esse pods ficam health, ele vai matando proporcionalmente a mesma quantidade **( 25% ) do replicaset antigo**.

🔸  Termina 25% dos pods velhos.

🔸  Comeca 25% dos pods novos.

Suponhamos que queira personalizar esse rollout para que tenha a seguinte caracteística quando uma nova release entrar no ar:

Imagine que temos um deployment configurado com 12 réplicas. Nesse cenário essa configuração garante que teria que subir 25% a mais, ou seja, eu teria 15 Pod rodando. E quando esses novos pods estirem health aí sim ele mataria 25% dos Pods do **Replicaset** antigo.


🔸 maxSurge: Numero máximo de pods que podem ser agendados para rollout acima do desejado, ou seja, se desejado e 100 essa opçao subiriria 125 pods, pode ser especificado em porcentagem.

🔸 maxUnavailable: Numero de pods que podem ficar indisponiveis durant um rollout. Se definir para **0** , ele não derruba nenhum pod até que os novos fiquem realth.

#### Obs.: Definir maxUnavailable é uma boa estratégia em ambiente produtivo.

Buscando Doc...

```bash
➜  kind git:(main) k explain deployment.spec.strategy
➜  kind git:(main) k explain deployment.spec.strategy.rollingUpdate
```

#### 2) Maxsurge / Maxunavailable Mão na Massa

```bash
➜  kind git:(main) k neat <<< $(k create deployment --image=nginx nginx --dry-run=client -o yaml)
```

Subindo Deployment com imagem nginx...

```bash
➜  kind git:(main) cat <<EOF | kaf -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  replicas: 12
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
EOF
deployment.apps/nginx created        
```


```bash
➜  kind git:(main) kgp
NAME                     READY   STATUS    RESTARTS   AGE
nginx-5869d7778c-48ln6   1/1     Running   0          95s
nginx-5869d7778c-5m5g8   1/1     Running   0          95s
nginx-5869d7778c-5xqp4   1/1     Running   0          95s
nginx-5869d7778c-9nghf   1/1     Running   0          95s
nginx-5869d7778c-cv9s8   1/1     Running   0          95s
nginx-5869d7778c-ffvc6   1/1     Running   0          95s
nginx-5869d7778c-qrvtf   1/1     Running   0          95s
nginx-5869d7778c-r6bw4   1/1     Running   0          95s
nginx-5869d7778c-sflt2   1/1     Running   0          95s
nginx-5869d7778c-vf862   1/1     Running   0          95s
nginx-5869d7778c-vqj6b   1/1     Running   0          95s
nginx-5869d7778c-zn2mf   1/1     Running   0          95s
```

Alterando imagem do Deployment para httpd, isso irá gerar um novo replicaset...

```bash
➜  kind git:(main) cat <<EOF | kaf -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  replicas: 12
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
EOF
deployment.apps/nginx configured
```

Em outra aba execute um get para monitorarmos esse rollout...

Observer que primeiramente ele cria os **25% a mais de Pods** para então **terminar 25% dos Pods do replicaset antigo**.

```bash
➜  kind git:(main) watch kubectl get pod
NAME                     READY   STATUS              RESTARTS   AGE
nginx-5869d7778c-48ln6   1/1     Running             0          3m58s
nginx-5869d7778c-5m5g8   1/1     Running             0          3m58s
nginx-5869d7778c-5xqp4   1/1     Running             0          3m58s
nginx-5869d7778c-9nghf   1/1     Running             0          3m58s
nginx-5869d7778c-cv9s8   1/1     Running             0          3m58s
nginx-5869d7778c-ffvc6   1/1     Running             0          3m58s
nginx-5869d7778c-qrvtf   1/1     Running             0          3m58s
nginx-5869d7778c-r6bw4   1/1     Running             0          3m58s
nginx-5869d7778c-sflt2   1/1     Running             0          3m58s
nginx-5869d7778c-vf862   1/1     Running             0          3m58s
nginx-5869d7778c-vqj6b   1/1     Running             0          3m58s
nginx-5869d7778c-zn2mf   1/1     Running             0          3m58s
nginx-69bf56d45f-7fbcg   0/1     ContainerCreating   0          12s
nginx-69bf56d45f-8p566   0/1     ContainerCreating   0          12s
nginx-69bf56d45f-sr4bk   0/1     ContainerCreating   0          12s
```
