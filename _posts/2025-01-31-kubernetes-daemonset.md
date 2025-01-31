---
layout: post
collection: kubernetes
permalink: /kubernetes/daemonset
title:  "Kubernetes Daemonset"
author: Paulo Rogério
date:   2025-01-30 15:00:13 -0300
categories: [kubernetes]
published: true
---

## 🚀 Kubernetes Daemonset

- [1) Características Daemonset](#1-características-daemonset)
- [2) Caso de Usos](#2-caso-de-usos)
- [3) Como é escalado o Daemonset](#3-como-é-escalado-o-daemonset)
- [4) Criando Daemonset](#4-criando-daemonset)
- [5) Usando Explain para Daemonset](#5-usando-explain-para-daemonset)
- [6) Daemonset Mão na Massa](#6-daemonset-mão-na-massa)

#### 1) Características Daemonset

🔸 Daemonset não se define o número de replicas. 
🔸 O Daemonset será igual ao número de nodes de um cluster. 
🔸 Um pod por cluster.
🔸 O Daemonset não passa pelo kube-scheduler

#### 2) Caso de Usos

🔸 Coletor de Logs
🔸 CNI

Esses **Pods** rodam a nivel de host. Para Pods que gerenciam os logs geralmente é montado a pasta **/var/log** do host dentro do DaemonSet ( pod ).

#### 3) Como é escalado o Daemonset

🔸 Daemonset não se define o número de replicas. 
🔸 Daemonset não passa pelo kube-scheduler.

Suponhamos que temos 2 worker

```bash
➜  kind git:(main) k get nodes
NAME                 STATUS   ROLES             AGE   VERSION
prgs-control-plane   Ready    control-plane     86m   v1.31.2
prgs-worker          Ready    worker-apps       86m   v1.31.2
prgs-worker2         Ready    worker-postgres   86m   v1.31.2
```

Se precisar adicionar um novo **worker** o ***daemonSet controller*** vai detectar que o novo host não tem o daemonset e irá instala-lo.

É comum eu especifica uma label para rodar algum tipo de DaemonSet, **ex: Um worker que roda Machine Learning** que faz uso de CPU diferenciado, as vezes é necessário instalar algum daemon set apenas nesse worker.

#### 4) Criando Daemonset

O comando ***kubectl create*** não cria um **daemonset**, então uma forma fácil de fazer isso é gerar um manifesto **yaml de um Deployment** e após gerado alterar apenas o **Kind**.

```bash
➜  kind git:(main) k get nodes --show-labels
prgs-worker          Ready    worker-apps       97m   v1.31.2   kubernetes.io/role=worker-apps,prgs/apps=true
```

```bash
➜  kind git:(main) k neat <<< $(k create deployment --image=nginx nginx --dry-run=client -o yaml)
```

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: nginx
  name: nginx
spec:
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
      nodeSelector: 
        prgs/apps: "true"
```

#### 5) Usando Explain para Daemonset

No dia a dia , poderá usar a documentação oficial para construir melhor seu **Daemonset**. Essa mesma documentação pode ser acessada pelo **kubectl explain**

Quais campos usar?

```bash
➜  kind git:(main) k explain daemonset
➜  kind git:(main) k explain daemonset.spec
➜  kind git:(main) k explain daemonset.spec.template
➜  kind git:(main) k explain daemonset.spec.template.spec ( Declaração dos pods )
```

```bash
nodeSelector	<map[string]string>
    NodeSelector is a selector which must be true for the pod to fit on a node.
    Selector which must match a node's labels for the pod to be scheduled on
    that node. More info:
    https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
```

#### 6) Daemonset Mão na Massa

Vamos gerar um manifesto dinamicamente com **kubectl create deployment**, substituindo o **Kind** e removendo a linha referente ao numero de replicas.

```bash
➜  kind git:(main) k neat <<< $(k create deployment --image=nginx nginx --dry-run=client -o yaml) | sed 's/kind: Deployment/kind: DaemonSet/;/replicas: 1/d' <<EOF | kaf -
EOF
daemonset.apps/nginx created
```

Como no meu lab tenho 2 **worker nodes**, terei 2 deploy...

```bash
➜  kind git:(main) kgp
NAME          READY   STATUS              RESTARTS   AGE
nginx-gktpx   0/1     ContainerCreating   0          25s
nginx-tkzjf   0/1     ContainerCreating   0          25s
```

```bash
➜  kind git:(main) kgp -o wide
NAME          READY   STATUS    RESTARTS   AGE    IP           NODE           NOMINATED NODE   READINESS GATES
nginx-gktpx   1/1     Running   0          4m4s   10.244.2.5   prgs-worker2   <none>           <none>
nginx-tkzjf   1/1     Running   0          4m4s   10.244.1.6   prgs-worker    <none>           <none>
```

```bash
➜  kind git:(main) k get daemonsets
NAME    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
nginx   2         2         2       2            2           <none>          4m38s
```

Apesar de ser um daemonset faço o trobleshouting como se fosse um pod...

Fazendo um describe no pod...

```bash
➜  kind git:(main) kdesp nginx-gktpx
``` 

```bash
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  5m25s  default-scheduler  Successfully assigned default/nginx-gktpx to prgs-worker2
  Normal  Pulling    5m24s  kubelet            Pulling image "nginx"
  Normal  Pulled     4m59s  kubelet            Successfully pulled image "nginx" in 24.713s (24.713s including waiting). Image size: 72080558 bytes.
  Normal  Created    4m59s  kubelet            Created container: nginx
  Normal  Started    4m59s  kubelet            Started container nginx
```
