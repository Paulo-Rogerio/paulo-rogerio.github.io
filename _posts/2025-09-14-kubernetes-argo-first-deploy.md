---
layout: post
collection: kubernetes
permalink: /kubernetes/argocd-infraestrura
title:  "Argocd First Deploy"
author: Paulo Rogério
date:   2025-09-14 22:00:00 -0300
categories: [kubernetes]
published: true
---

## 🚀 Deploy ArgoCD 

- [1) Bump Modules Terraform](#1-bump-modules-terraform)
- [2) Repositórios](#2-repositórios)
- [3) Deploy](#3-Deploy)
- [4) Youtube Demo](#4-youtube-demo)

#### 1) Bump Modules Terraform

Foi refatorado os [Modules](https://gitlab.com/prgs-estudos/sre/cloud-infra/terraform-blueprints/blueprints), para que usem a ultima versão dos módulos oficias da **AWS**. 

Juntamente com esse upgrade dos módulos, foi ajustado o **terragrunt** para que os **Node Groups** do Eks possa ser passado como parâmetro, isso permite gerenciar múltiplos node groups no mesmo cluster Kubernetes. 

```hcl
inputs = {

  cluster_name            = "prgs"
  cluster_version         = "1.32"
  environment             = local.account_vars.environment    
  vpc_id                  = dependency.vpc.outputs.vpc_id
  public_subnets_ids      = dependency.vpc.outputs.public_subnets
  private_subnets_ids     = dependency.vpc.outputs.private_subnets
  tags                    = local.service_vars.tags
  enable_logs             = false
  eks_managed_node_groups = { 

    # Cluster PostgreSQL Vault 
    "postgres-vault-v2" = { 
      ami_type       = "AL2023_x86_64_STANDARD" 
      instance_types = ["t3.medium"] 
      min_size       = 3 
      max_size       = 4 
      desired_size   = 3 
      size_disk      = 50 
      labels         = { 
        role    = "postgres-vault" 
        cluster = "prgs-production"
      }
    }

    # Cluster PostgreSQL App Rails 
    "postgres-app-rails-v1" = { 
      ami_type       = "AL2023_x86_64_STANDARD" 
      instance_types = ["t3.medium"] 
      min_size       = 3 
      max_size       = 4 
      desired_size   = 3 
      size_disk      = 50 
      labels         = { 
        role    = "postgres-app-rails" 
        cluster = "prgs-production"
      }
    }

    # Cluster PostgreSQL PowerDNS 
    "postgres-powerdns-v1" = { 
      ami_type       = "AL2023_x86_64_STANDARD" 
      instance_types = ["t3.medium"] 
      min_size       = 3 
      max_size       = 4 
      desired_size   = 3 
      size_disk      = 50 
      labels         = { 
        role    = "postgres-powerdns" 
        cluster = "prgs-production"
      }
    }

  }
}
```

Você pode acompanhar isso detalhadamente [Aqui](https://www.youtube.com/watch?v=uzuxW8H0Mg4).

#### 2) Repositórios

Para esse estudos usando o **ArgoCD** iremos trabalhar com 2 repositórios:

🔸 **Operator** Deploy do Operator **cloudnative-pg** usando **ArgoCD**.

🔸 **Meus Cluster PostgreSQL** Cada repositório representará seu respectivo cluster **Postgres**, para esse estudo, teremos apenas o cluster **postgres-vault**.  

[Operator](https://gitlab.com/prgs-estudos/dbre/cloudnative-pg/operator)

[Cluster - PostgreSQL Vault](https://gitlab.com/prgs-estudos/dbre/cloudnative-pg/postgres-vault)

O primeiro deploy que deve ser realizado é o operator, para isso clone o repositório e navegue até diretório **argocd-projects/plain-yaml**, aplique o manifesto.

```bash
kubectl apply -f .
```

#### 3) Deploy

Existe várias forma de personalizar como o **ArgoCD** irá usar para manipular os yamls para atender um determinado **enviroment**, iremos atuar iniciamente com **kustomize** e futuramente com **helm**.

Para testar o comportamento do kustomize, ou seja, para entender como que o kustomize irá manipular os manifestos yamls, execute os procedimentos abaixo.

**Obs.:** É necessário que tenha o **kustomize** instalado em seu host.

Navegue no repositório **postgres-vault** no path **argocd-manifest/plain/overlays/production**, dentro desse diretório execute: 

```bash
kustomize build .
```

Será mostrado uma prévia do que será deployado.

```bash
apiVersion: v1
kind: Namespace
metadata:
  name: pg-vault-production
---
apiVersion: v1
data:
  password: cG9zdGdyZXM=
  username: cG9zdGdyZXM=
kind: Secret
metadata:
  name: pg-vault
  namespace: pg-vault-production
type: kubernetes.io/basic-auth
---
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-vault
  namespace: pg-vault-production
spec:
  affinity:
    enablePodAntiAffinity: true
    nodeSelector:
      role: postgres-vault
    podAntiAffinityType: required
    tolerations:
    - effect: NoSchedule
      key: role
      operator: Equal
      value: postgres-vault
    topologyKey: kubernetes.io/hostname
  enableSuperuserAccess: true
  imageName: ghcr.io/cloudnative-pg/postgresql:15
  instances: 3
  primaryUpdateStrategy: unsupervised
  storage:
    size: 1Gi
    storageClass: gp3
```    

O resultado do build, é o esperado? Entao, vamos aplica-lo.

Navegue no repositório **postgres-vault** no path **argocd-maniargocd-projects/plain/production**, dentro desse diretório execute: 

```bash
kubectl apply -f .
```

#### 6) Youtube Demo

Você pode acompanhar todo esse processo de implementação no link abaixo.

[Demo](https://www.youtube.com/watch?v=dm9s3XLGzf8)
