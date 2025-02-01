---
layout: post
collection: kubernetes
permalink: /kubernetes/pod-lifecycle
title:  "Kubernetes Pod Lifecycle"
author: Paulo Rogério
date:   2025-01-31 15:00:13 -0300
categories: [kubernetes]
published: true
---

## 🚀 Kubernetes Pod Lifecycle

- [1) Pod Lifecycle](#1-pod-lifecycle)
- [2) Pod Lifecycle Mão na Massa](#2-pod-lifecycle-mão-na-massa)

#### 1) Pod Lifecycle

Um Pod recebe um prazo para terminar graciosamente, que é de 30 segundos por padrão. Ou seja a aplicação deve subir nesse intervalo de tempo e sair com status code 0.
Apos esse 30 segundos o kubelet envia um sinal de sigkill para aplicaçao, então mata o processo imediatamente.

Podemos manipular esse ciclo de vida, configurar um yaml para sempre que receber um sigterm, seja realizado uma ação.
Para isso é necessário ajusta o **terminationGracePeriodSeconds** com um pouco mais do padrao, por exemplo 60 segundos

#### 2) Pod Lifecycle Mão na Massa

```bash
➜  kind git:(main) cat <<EOF | kaf -
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pod-lifecycle
  name: pod-lifecycle
spec:
  terminationGracePeriodSeconds: 60
  containers:
  - image: alpine
    name: pod-lifecycle
    command:
      - "sleep"
      - "9999999999"
    lifecycle:
       preStop: 
          exec:
            command: 
              - sh 
              - -c
              - curl 10.244.1.6 
EOF
```

Se o commando (curl / script ) demorar mais que 30 secundos para executar , posso ajustar isso definindo
**terminationGracePeriodSeconds: 60** com um valor que satisfaça minha necessidade. 

```bash
➜  kind git:(main) k get pods -o wide
pod-lifecycle     1/1     Running   0          82s   10.244.2.6   prgs-worker
```

Essa imagem não tem o curl, entao para simular o cenário vamos instalar o binário no Pod.

```bash
➜  kind git:(main) k exec -it pod-lifecycle -- sh
apk add curl
```

Ele mandará um curl para o service que está escutando em 10.244.1.6, então vamos monitorar os logs do nginx. Tem que chegar uma requisição do IP 10.244.2.6

```bash
➜  kind git:(main) k logs multicontainers -c nginx -f
```

Ao encerrar esse Pod deverá chegar uma notificação...

```bash
➜  kind git:(main) k delete pod pod-lifecycle
```

```bash
10.244.2.8 - - [07/Jan/2025:12:28:05 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.11.1" "-"
```
