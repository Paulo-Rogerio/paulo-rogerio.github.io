---
layout: post
collection: kubernetes
permalink: /kubernetes/argocd-infraestrura
title:  "Argocd InfraEstrutura"
author: Paulo Rogério
date:   2025-08-18 22:00:00 -0300
categories: [kubernetes]
published: true
---

## 🚀 Deploy ArgoCD 

- [1) Deploy Infraestrurura](#1-deploy-infraestrurura)
- [2) Repositórios](#2-repositórios)
- [3) External DNS](#3-external-dns)
- [4) Credencias](#4-credencias)
- [5) Destruindo Tudo](#5-destruindo-tudo)
- [6) Youtube Demo](#6-youtube-demo)


#### 1) Deploy Infraestrurura

Nessa documentação iremos falar sobre a infraestrutura necessária para deployar o ArgoCD em um ambiente **EKS**. Foi definido um **Makefile** que ajuda no deploy da infra.

Para que esse código possa ser adpado à sua realidade, será necessário:

- Criar uma conta e definir um aliases. O aliases usado para esse ambiente chama-se **estudos-paulo**.
- Cada **aliases**, deve estar definido no **root da estrutura principal** que contempla a implementação do terraform.

Ex: 

```bash
.
├── Makefile
├── README.md
├── _bin
│   ├── clean-all.sh
│   ├── create-all.sh
├── estudos-paulo
│   └── production
│       ├── _global
│       │   └── route53
├── outro-aliases
│   └── production
│       ├── _global
│       │   └── route53
```

- Para facilitar a gestão das versões de **terraform / terragrunt**, foi usado um gerenciador de versão para manipular e facilitar a transição de versão.

[Tgenv](https://github.com/tgenv/tgenv)

[Tfenv](https://github.com/tfutils/tfenv)

- Na raiz do projeto encontrará um **Makefile**, que foi instrumentado para invocar os códigos **terragrunt** em suas respectivas pastas.

```bash
DIR=estudos-paulo/production/us-east-1/shared/vpc make terragrunt-init
DIR=estudos-paulo/production/us-east-1/shared/vpc make terragrunt-plan
DIR=estudos-paulo/production/us-east-1/shared/vpc make terragrunt-apply

DIR=estudos-paulo/production/us-east-1/services/eks/cluster make terragrunt-init
DIR=estudos-paulo/production/us-east-1/services/eks/cluster make terragrunt-plan
DIR=estudos-paulo/production/us-east-1/services/eks/cluster make terragrunt-apply

DIR=estudos-paulo/production/us-east-1/services/eks/deployments make terragrunt-init
DIR=estudos-paulo/production/us-east-1/services/eks/deployments make terragrunt-plan
DIR=estudos-paulo/production/us-east-1/services/eks/deployments make terragrunt-apply

DIR=estudos-paulo/production/us-east-1/services/eks/helm make terragrunt-init
DIR=estudos-paulo/production/us-east-1/services/eks/helm make terragrunt-plan
DIR=estudos-paulo/production/us-east-1/services/eks/helm make terragrunt-apply
```

#### 2) Repositórios

🔸 **Blueprints** Modulos necessários que servem os arquivos **terragrunt**.

🔸 **Terrafgrunt** é a estrutura que defini e invoca o terraform  de forma centralizada por meio de **modulos**, cujo propósito e permitir o reaproveitamento de código.  

[Blueprint](https://gitlab.com/prgs-estudos/sre/cloud-infra/terraform-blueprints/blueprints)

[Terragrunt](https://gitlab.com/prgs-estudos/sre/cloud-infra/terragrunt/aws-infrastructure-terragrunt)

#### 3) External DNS

Para esse **laboratório** estou usando um **DNS** externo. A integração com o **Certificate Manager da AWS**  por meio de **DNS** só acontecerá se inserido as seguintes entradas nas configurações do DNS.

```bash

nome    Conteúdo                  prio
@       0 issue "amazon.com"       0
@       0 issuewild "amazon.com"   0
@       0 issue "awstrust.com"     0
@       0 issue "amazontrust.com"  0
@       0 issue "amazonaws.com"    0
@       0 issue "SomeCA.com"       0
```

#### 4) Credencias

Após o término da execução do **terraform** , atualize seu kubeconfig para consumir o cluster.

```bash
aws eks update-kubeconfig --region us-east-1 --name <meu-cluster>
kubectl config use-context arn:aws:eks:us-east-1:<minha-conta>:cluster/<meu-cluster>
```

Obtendo as credencias do ArgoCD.

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d | xargs
```

#### 5) Destruindo Tudo

Finalizado o laboratório, para evitar custos, recomendo a remoção de toda a implementação. O processo de deploy dura em média **25 minutos**. Para remover basta executar o codigo abaixo:

```bash
helm uninstall argo-cd -n argocd

DIR=estudos-paulo/production/us-east-1/services/eks/helm make terragrunt-init
DIR=estudos-paulo/production/us-east-1/services/eks/helm make terragrunt-destroy
DIR=estudos-paulo/production/us-east-1/services/eks/helm make terragrunt-clean

DIR=estudos-paulo/production/us-east-1/services/eks/deployments make terragrunt-init
DIR=estudos-paulo/production/us-east-1/services/eks/deployments make terragrunt-destroy
DIR=estudos-paulo/production/us-east-1/services/eks/deployments make terragrunt-clean

DIR=estudos-paulo/production/us-east-1/services/eks/cluster make terragrunt-init
DIR=estudos-paulo/production/us-east-1/services/eks/cluster make terragrunt-destroy
DIR=estudos-paulo/production/us-east-1/services/eks/cluster make terragrunt-clean

DIR=estudos-paulo/production/us-east-1/shared/vpc make terragrunt-init
DIR=estudos-paulo/production/us-east-1/shared/vpc make terragrunt-destroy
DIR=estudos-paulo/production/us-east-1/shared/vpc make terragrunt-clean
```

#### 6) Youtube Demo

Você pode acompanhar todo esse processo de implementação no link abaixo.

[Demo](https://github.com/tfutils/tfenv)