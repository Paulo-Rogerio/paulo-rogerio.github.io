---
layout: post
collection: kubernetes
permalink: /kubernetes/namespaces
title:  "Kubernetes Namespaces"
author: Paulo Rogério
date:   2025-01-28 09:59:13 -0300
categories: [kubernetes]
published: true
---

## 🚀 Gerenciando Namespaces

- [1) Conceito Namespace](#1-conceito-namespace)
- [2) Porque Deveria Trabalhar com Namespace](#2-porque-deveria-trabalhar-com-namespace)
- [3) Criando Namespace](#3-criando-namespace)
- [4) Syntaxe Namespace](#4-syntaxe-namespace)
- [5) Como Definir uma Namespace como Default](#5-como-definir-uma-namespace-como-default)

#### 1) Conceito Namespace

Os namespaces do Kubernetes são partições lógicas em um cluster que fornecem um escopo para recursos, como pods, serviços e implantações. Eles ajudam a organizar e gerenciar recursos em um ambiente multilocatário, melhorando a segurança, a escalabilidade e a utilização de recursos.

#### 2) Porque Deveria Trabalhar com Namespace

✨ ***Isolamento de recursos:*** Os namespaces fornecem uma maneira de isolar recursos, como pods e serviços, de outros namespaces 🔒🔑.

✨ ***Organização:*** Os namespaces ajudam a organizar os recursos de maneira lógica e estruturada, facilitando o gerenciamento e o dimensionamento do cluster 📈📊.

✨ ***Segurança:*** Os namespaces fornecem uma maneira de implementar o RBAC (Controle de Acesso Baseado em Função) e limitar o acesso aos recursos 🔒🔒.

✨ ***Escalabilidade:*** Os namespaces facilitam o dimensionamento do cluster, fornecendo uma maneira de gerenciar recursos de forma independente 🔁📈.

#### 3) Criando Namespace

#### [Aliases Kubectl](https://paulo-rogerio.github.io/kubernetes/aliases){:target="_blank"}

Criando Namespace usando manifesto yaml

```bash
➜  ~ cat <<EOF | kaf -
apiVersion: v1
kind: Namespace
metadata:
  name: prgs
EOF
namespace/prgs created
```

Check os namespace criados

```bash
➜  ~ kgns
NAME                 STATUS   AGE
default              Active   7m4s
ingress-nginx        Active   4m20s
kube-node-lease      Active   7m4s
kube-public          Active   7m4s
kube-system          Active   7m4s
local-path-storage   Active   6m48s
metallb-system       Active   5m51s
prgs                 Active   2s
```

Deletando namespace 

```bash
➜  ~ kdns prgs
namespace "prgs" deleted
```

#### 4) Syntaxe Namespace

Se eu não lembrar a sintexe do manifesto yaml para criar uma namespace?

#### [Aliases Kubectl](https://paulo-rogerio.github.io/kubernetes/aliases){:target="_blank"}

Gerar o yaml 

```bash
➜  ~ k neat <<< $(kcns prgs --dry-run=client -o yaml)
apiVersion: v1
kind: Namespace
metadata:
  name: prgs
```

Gerar o Yaml e aplica o manifesto.

```bash
k neat <<< $(kcns prgs --dry-run=client -o yaml) <<EOF | kaf -
EOF
namespace/prgs created
```

#### 5) Como Definir uma Namespace como Default

Set default Conexts

```bash
kubectl config set-context --current --namespace=prgs
```

```bash
kubectl config get-contexts
CURRENT   NAME        CLUSTER     AUTHINFO    NAMESPACE
*         kind-prgs   kind-prgs   kind-prgs    prgs
```