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
- [5) Como funciona o Scale do Statefullset](#5-como-funciona-o-scale-do-statefullset)
- [6) Como funciona o DNS](#6-como-funciona-o-dns)

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

Como o kuberenetes garante que teremos **1 volume** por Pod? Qual objeto no kubernetes responsável por fazer essa mágica?

###### R: VolumeClaimTemplate

```bash
➜ kind git:(main) k explain statefulsets.spec | grep required
  selector	<LabelSelector> -required-
  serviceName	<string> -required-
  template	<PodTemplateSpec> -required-
```

```bash
➜ kind git:(main) k explain statefulsets.spec.volumeClaimTemplates
➜ kind git:(main) k explain statefulsets.spec.volumeClaimTemplates.spec ( Aqui que mora o segrego )
➜ kind git:(main) k explain statefulsets.spec.volumeClaimTemplates.metadata ( Aqui detalhes desse metadata como nome por ex )
➜ kind git:(main) k explain statefulsets.spec.volumeClaimTemplates.spec.accessModes
➜ kind git:(main) k explain statefulsets.spec.volumeClaimTemplates.spec.resources
➜ kind git:(main) k explain statefulsets.spec.template.spec.containers
➜ kind git:(main) k explain statefulsets.spec.template.spec.containers.volumeMounts ( Arrary )
```

Check se o StorageClass deployado no cluster. StorageClass será explicado melhor no módulo de Persistencia de Dados. Por padrão o **kind** instala o StorageClass **Local-Path**.

```bash
➜ kind git:(main) k get sc
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  73m
```

No manifesto da aplicação **Statefullset** será necessário adicinar as seguintes entradas..

* Ponto de Montagem.

* Tipo de Acesso e o StorageClass usado.

```yaml
...
...
    spec:
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: nginx-html
          mountPath: "/usr/share/nginx/html"
...
...
...
  volumeClaimTemplates:
  - metadata:
      name: nginx-html
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1G

```



Primeiramente vamos criar um service, esse service como explicado acima, não contém **IP Address** pois ele é apenas um resolvedor de DNS.

Gerando Service ....

```bash
➜  kind git:(main) k neat <<< $(k create service clusterip nginx --clusterip="None" --dry-run=client -o yaml)
```

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  clusterIP: None
  ports:
  - name: nginx
    port: 80
  selector:
    app: nginx
```

Gerando Manifesto do Statefullset...

```bash
➜  kind git:(main) k neat <<< $(k create deployment --image=nginx nginx --dry-run=client -o yaml) | sed 's/kind: Deployment/kind: StatefulSet/' 
```

```yaml
apiVersion: apps/v1
kind: StatefulSet
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
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

Aplicando o manifesto...

```bash
➜  ~ cat <<EOF | kaf -
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  clusterIP: None
  ports:
  - name: nginx
    port: 80
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  serviceName: "nginx"
  replicas: 2
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
        volumeMounts:
        - name: nginx-html
          mountPath: "/usr/share/nginx/html"
  volumeClaimTemplates:
  - metadata:
      name: nginx-html
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1G
EOF
service/nginx created
statefulset.apps/nginx created
```

```bash
➜  kind git:(main) kgp
NAME      READY   STATUS    RESTARTS   AGE
nginx-0   1/1     Running   0          59s
nginx-1   1/1     Running   0          31s
```

Inspecionando o Pv...

```bash
➜  kind git:(main) k get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-02309071-a6dc-4208-b10a-aeaff8f94e79   1G         RWO            Delete           Bound    default/nginx-html-nginx-0   standard       <unset>                          2m19s
pvc-7f9056df-083d-4a0d-8c8b-199a8b6de798   1G         RWO            Delete           Bound    default/nginx-html-nginx-1   standard       <unset>                          110s
```

Inspecionando o Pvc...

```bash
➜  kind git:(main) k get pvc
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
nginx-html-nginx-0   Bound    pvc-02309071-a6dc-4208-b10a-aeaff8f94e79   1G         RWO            standard       <unset>                 2m43s
nginx-html-nginx-1   Bound    pvc-7f9056df-083d-4a0d-8c8b-199a8b6de798   1G         RWO            standard       <unset>                 2m14s
```

#### No kind o StoraClass Local-Path armazena os dados no seguinte lugar (/var/local-path-provisioner/)

Como tenho 2 worker nodes, vamos criar 1 arquivo **index.html** em cada worker.

```bash
➜  kind git:(main) ✗ docker exec prgs-worker bash -c "ls /var/local-path-provisioner/"
pvc-02309071-a6dc-4208-b10a-aeaff8f94e79_default_nginx-html-nginx-0
➜  kind git:(main) ✗
➜  kind git:(main) ✗
➜  kind git:(main) ✗ docker exec prgs-worker bash -c "echo Nginx-0 > /var/local-path-provisioner/pvc-02309071-a6dc-4208-b10a-aeaff8f94e79_default_nginx-html-nginx-0/index.html"
```

```bash
➜  kind git:(main) ✗ docker exec prgs-worker2 bash -c "ls /var/local-path-provisioner/"
pvc-7f9056df-083d-4a0d-8c8b-199a8b6de798_default_nginx-html-nginx-1
➜  kind git:(main) ✗ 
➜  kind git:(main) ✗
➜  kind git:(main) ✗ docker exec prgs-worker2 bash -c "echo Nginx-1 > /var/local-path-provisioner/pvc-7f9056df-083d-4a0d-8c8b-199a8b6de798_default_nginx-html-nginx-1/index.html"
```

Criando Port-Forward para acesso ao Pod...

```bash
➜  kind git:(main) k port-forward pod/nginx-0 8181:80
Forwarding from 127.0.0.1:8181 -> 80
Forwarding from [::1]:8181 -> 80
```

```bash
➜  kind git:(main) k port-forward pod/nginx-1 8282:80
Forwarding from 127.0.0.1:8282 -> 80
Forwarding from [::1]:8282 -> 80
```

```bash
➜  kind git:(main) ✗ for _ in {1..10}; do curl localhost:8181; done
Nginx-0
Nginx-0
Nginx-0
Nginx-0
Nginx-0
Nginx-0
Nginx-0
Nginx-0
Nginx-0
Nginx-0
```

```bash
➜  kind git:(main) ✗ for _ in {1..10}; do curl localhost:8282; done
Nginx-1
Nginx-1
Nginx-1
Nginx-1
Nginx-1
Nginx-1
Nginx-1
Nginx-1
Nginx-1
Nginx-1
```

#### 5) Como funciona o Scale do Statefullset

🔸 O scale desse cara é mais lento pois ele é feito um a um.

```bash
➜  kind git:(main) ✗ k scale statefulset nginx --replicas 4
statefulset.apps/nginx scaled
```

```bash
➜  kind git:(main) kgp
NAME      READY   STATUS    RESTARTS   AGE
nginx-0   1/1     Running   0          22m
nginx-1   1/1     Running   0          21m
nginx-2   1/1     Running   0          116s
nginx-3   1/1     Running   0          107s
```

🔸 Ao baixar todos os Pods, **OS PVC NÃO SÃO ALTERADOS**.

```bash
➜  kind git:(main) k scale statefulset nginx --replicas 0
statefulset.apps/nginx scaled
```

```bash
➜  kind git:(main) kgp
No resources found in default namespace.
```

List Pv...

```bash
➜  kind git:(main) k get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-02309071-a6dc-4208-b10a-aeaff8f94e79   1G         RWO            Delete           Bound    default/nginx-html-nginx-0   standard       <unset>                          23m
pvc-344ea339-60cf-44e2-87bb-04e4892e2c23   1G         RWO            Delete           Bound    default/nginx-html-nginx-3   standard       <unset>                          2m52s
pvc-7f9056df-083d-4a0d-8c8b-199a8b6de798   1G         RWO            Delete           Bound    default/nginx-html-nginx-1   standard       <unset>                          22m
pvc-c24b55c5-291a-4294-9ad9-2284185772c3   1G         RWO            Delete           Bound    default/nginx-html-nginx-2   standard       <unset>                          3m
```

List Pvc...

```bash
➜  kind git:(main) k get pvc
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
nginx-html-nginx-0   Bound    pvc-02309071-a6dc-4208-b10a-aeaff8f94e79   1G         RWO            standard       <unset>                 23m
nginx-html-nginx-1   Bound    pvc-7f9056df-083d-4a0d-8c8b-199a8b6de798   1G         RWO            standard       <unset>                 22m
nginx-html-nginx-2   Bound    pvc-c24b55c5-291a-4294-9ad9-2284185772c3   1G         RWO            standard       <unset>                 3m6s
nginx-html-nginx-3   Bound    pvc-344ea339-60cf-44e2-87bb-04e4892e2c23   1G         RWO            standard       <unset>                 2m57s
```

#### 6) Como funciona o DNS

Por padrao a consulta de Stateful set nao é feita por IP , porque o service do tipo Cluster-IP está definido como "None", por isso nao tem IP definido.

```bash
➜  kind git:(main) k get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   50m
nginx        ClusterIP   None         <none>        80/TCP    27m
```

A consulta é feita por DNS...

```bash
➜  kind git:(main) for i in 0 1 2 3; do kubectl exec "nginx-$i" -- sh -c 'hostname'; done
nginx-0
nginx-1
nginx-2
nginx-3
```

Executando Teste...

Em outra aba execute um Pod temporáriamente para efetuar os testes.

```bash
➜  kind git:(main) ✗ kubectl run -i --tty --image alpine dns-test --restart=Never --rm
If you don't see a command prompt, try pressing enter.
/ #
/ #
/ # apk add bind-tools
/ # host kubernetes
kubernetes.default.svc.cluster.local has address 10.96.0.1

/ #
/ #
/ # host nginx
nginx.default.svc.cluster.local has address 10.244.2.7
nginx.default.svc.cluster.local has address 10.244.1.7
/ #
/ #
/ # host nginx-0.nginx.default.svc.cluster.local
nginx-0.nginx.default.svc.cluster.local has address 10.244.1.7
/ #
/ #
/ # host nginx-1.nginx.default.svc.cluster.local
nginx-1.nginx.default.svc.cluster.local has address 10.244.2.7
/ #
/ #
/ # host nginx-0.nginx
nginx-0.nginx.default.svc.cluster.local has address 10.244.1.7
/ #
/ #
/ # host nginx-1.nginx
nginx-1.nginx.default.svc.cluster.local has address 10.244.2.7
/ #
/ #
/ # ping -c 1 nginx-0.nginx
PING nginx-0.nginx (10.244.1.7): 56 data bytes
64 bytes from 10.244.1.7: seq=0 ttl=63 time=0.130 ms
/ #
/ #
/ # ping -c 1 nginx-1.nginx
PING nginx-1.nginx (10.244.2.7): 56 data bytes
64 bytes from 10.244.2.7: seq=0 ttl=62 time=0.329 ms
/ #
/ #
/ # nslookup nginx-0.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx-0.nginx
Address 1: 10.244.1.19 nginx-0.nginx.default.svc.cluster.local
/ #
/ #
/ # nslookup nginx-1.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx-1.nginx
Address 1: 10.244.2.14 nginx-1.nginx.default.svc.cluster.local
/ #
/ #
/ # nslookup nginx-0.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx-0.nginx
Address 1: 10.244.1.9 nginx-0.nginx.default.svc.cluster.local
/ #
/ #
/ # nslookup nginx-1.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx-1.nginx
Address 1: 10.244.2.10 nginx-1.nginx.default.svc.cluster.local
/ #
/ #
/ # nslookup nginx-2.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx-2.nginx
Address 1: 10.244.1.10 nginx-2.nginx.default.svc.cluster.local
/ #
/ #
/ # nslookup nginx-3.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx-3.nginx
Address 1: 10.244.2.11 nginx-3.nginx.default.svc.cluster.local

/ #
/ #
/ # nslookup nginx-3.nginx.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx-3.nginx.default.svc.cluster.local
Address 1: 10.244.2.11 nginx-3.nginx.default.svc.cluster.local
```
