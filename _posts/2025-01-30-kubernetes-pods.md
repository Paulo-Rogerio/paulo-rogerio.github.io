---
layout: post
collection: kubernetes
permalink: /kubernetes/pods
title:  "Kubernetes Pods"
author: Paulo Rogério
date:   2025-01-30 15:00:13 -0300
categories: [kubernetes]
published: true
---

## 🚀 Gerenciando Pods

- [1) Conceito Pod](#1-conceito-pod)
- [2) Tipos de Pod](#2-tipos-de-pod)
- [3) Pods](#3-pods)

#### 1) Kubernetes Pod

Os pods são as menores unidades de computação implantáveis ​​que você pode criar e gerenciar no Kubernetes.

#### 2) Tipos de Pod

#### Static Pods 

[Mais detalhes sobre Static Pods](https://paulo-rogerio.github.io/kubernetes/etcd-deploy#1-deploy-etcd-static-pod){:target="_blank"}

#### Sidecar

Recurso disponível onde consigo subir 2 ou mais **containers**, no mesmo Pod. Nesse formato de deploy, os **containers** compatilham recursos de rede e arquivos, mas são processos isolados.

#### Init Containers

Container init, não fazem parte do processo principal do pod, geralmente são ações que fazem determinadas tarefas pre-requistos antes do Pod realmente iniciar. Posso ter varios init containers, em meu manifesto **yaml** e somente quando eles finalizarem ***( Init / Passar )**, é que o container **( Pod da aplicação )** será executado.

Ex: 
- clonar um repositório
- Um pod que roda PostgreSQL, poderia ter um Init que faria um **pg_basebackup**


#### 3) Pods

Caso não esteje familiarizado com os aliases usado nesse documento, [Click Aqui](https://paulo-rogerio.github.io/kubernetes/aliases){:target="_blank"} para ser redirecionado a documentação sobre aliases.

Se eu não lembrar a sintexe do manifesto yaml para criar uma Pod?

```bash
➜  kind git:(main) k run --image busybox demo --dry-run=client -o yaml
```

Se preferir pode-se usar o **plugin** **neat** do krew para limpar alguns metadados.

```bash
➜  kind git:(main) k neat <<< $(k run --image busybox demo --dry-run=client -o yaml)
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: demo
  name: demo
spec:
  containers:
  - image: busybox
    name: demo
```

Vamos criar o Pod Init que tenha a seguinte característica:
- Será criado um initContainers que ficará esperando até que um service chamado **psql**, seja criado.


```bash
➜  ~ cat <<EOF | kaf -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
  initContainers:
  - name: waitfordns
    image: busybox
    command: [ "/bin/sh", "-c", "--" ]
    args: [ "until ping -c 10 psql; do echo 'Trying to resolve...'; echo; sleep 1; done" ]
EOF
pod/nginx created
```

Nesse contexto é criado 2 containers no mesmo pod ( **nginx** => aplicação e **waitfordns** que é meu pre-deploy ).

```bash
➜  kind git:(main) kgp
NAME    READY   STATUS     RESTARTS   AGE
nginx   0/1     Init:0/1   0          12s
```

```bash
➜  kind git:(main) k describe pod nginx
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  2m26s  default-scheduler  Successfully assigned default/nginx to prgs-worker
  Normal  Pulling    2m26s  kubelet            Pulling image "busybox"
  Normal  Pulled     2m15s  kubelet            Successfully pulled image "busybox" in 10.33s (10.33s including waiting). Image size: 2167089 bytes.
  Normal  Created    2m15s  kubelet            Created container: waitfordns
  Normal  Started    2m15s  kubelet            Started container waitfordns
```

Para mim ler os logs desse **"pre-deploy"** chamado waitfordns, posso fazer isso especificando o container.

```bash
➜  kind git:(main) k logs nginx
Defaulted container "nginx" out of: nginx, waitfordns (init)
Error from server (BadRequest): container "nginx" in pod "nginx" is waiting to start: PodInitializing
```

```bash
➜  kind git:(main) k logs nginx -c waitfordns
ping: bad address 'psql'
Trying to resolve...

ping: bad address 'psql'
Trying to resolve...

ping: bad address 'psql'
Trying to resolve...

ping: bad address 'psql'
Trying to resolve...
```

O container só irá proceguir sua execução após o service **psql** estiver criado.

```bash
➜  kind git:(main) k create service clusterip psql --tcp=5432:5432
service/psql created 
```

Check se o Pod foi iniciado..

```bash
➜  kind git:(main) kgp
NAME    READY   STATUS            RESTARTS   AGE
nginx   0/1     PodInitializing   0          6m58s
```

```
➜  kind git:(main) kgp
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          7m36s
```
