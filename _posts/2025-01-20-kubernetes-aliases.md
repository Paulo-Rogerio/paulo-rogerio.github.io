---
layout: post
collection: kubernetes
permalink: /kubernetes/aliases
title:  "Kubernetes Aliases Command"
author: Paulo Rogério
date:   2025-01-22 20:59:13 -0300
categories: [kubernetes]
published: true
---

## 🚀 Aliases Command

## Append file

```bash
echo "source ~/.zshrc_aliases" >> ~/.zshrc
```

## Create file with content the aliases

```bash
touch ~/.zshrc_aliases
```

```bash
#========== k8s ===============
alias k8s-alias='
                printf "%s \n" "k .............. kubectl"
                printf "%s \n" "kgctx .......... kubectl config get-contexts"
                printf "%s \n" "kuctx .......... kubectl config use-context"
                printf "%s \n" "kictx .......... kubectl cluster-info --context"
                printf "%s \n" "kg ............. kubectl get"
                printf "%s \n" "kgp ............ kubectl get pod"
                printf "%s \n" "kdesp .......... kubectl describe pod"
                printf "%s \n" "kdp ............ kubectl delete pod"
                printf "%s \n" "kgpw ........... kubectl get pod -o wide"
                printf "%s \n" "kgpall ......... kubectl get pod --all-namespaces"
                printf "%s \n" "kgd ............ kubectl get deployment"
                printf "%s \n" "kgdall ......... kubectl get deployment --all-namespaces"
                printf "%s \n" "kdd ............ kubectl delete deployment"
                printf "%s \n" "kddall ......... kubectl delete deployment --all"
                printf "%s \n" "kgrs ........... kubectl get replicaset"
                printf "%s \n" "kgn ............ kubectl get nodes"
                printf "%s \n" "kgnl ........... kubectl get nodes --show-labels"
                printf "%s \n" "kaf ............ kubectl apply -f"
                printf "%s \n" "kdf ............ kubectl delete -f"
                printf "%s \n" "kl ............. kubectl log"
                printf "%s \n" "krp ............ kubectl run -it --image alpine myspec -- sh"
                printf "%s \n" "krpyaml ........ kubectl run --image nginx nginx --dry-run=client -o yaml"
                printf "%s \n" "krest .......... krest deployment/nginx"
                printf "%s \n" "krest .......... krest -n default deployment frontend"
                printf "%s \n" "kcdyaml ........ kubectl create deployment nginx --image nginx --dry-run=client -o yaml"
                '

alias k='kubectl'

alias kgctx='kubectl config get-contexts'
alias kuctx="kubectl config use-context"
alias kictx="kubectl cluster-info --context"

alias kg='kubectl get'
alias kgp='kubectl get pod'
alias kdesp='kubectl describe pod'
alias kdp='kubectl delete pod'
alias kgpw='kubectl get pod -o wide'
alias kgpall='kubectl get pod --all-namespaces'

alias kgd='kubectl get deployment'
alias kgdall='kubectl get deployment --all-namespaces'
alias kdd='kubectl delete deployment'
alias kddall='kubectl delete deployment --all'

alias kgrs='kubectl get replicaset'
alias kgn='kubectl get nodes'
alias kgnl='kubectl get nodes --show-labels'

alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'

alias kl='kubectl logs'
alias krp='kubectl run -it --image alpine myspec -- sh'
alias krpyaml='kubectl run --image nginx nginx --dry-run=client -o yaml'

alias krest='kubectl rollout restart'
alias kcdyaml='kubectl create deployment nginx --image nginx --dry-run=client -o yaml'
```

[Aliases Github](https://github.com/wuestkamp/Kubernetes-Certified-Administrator?tab=readme-ov-file){:target="_blank"}

[BookMarks Kubernetes](https://github.com/reetasingh/CKAD-Bookmarks?tab=readme-ov-file){:target="_blank"}

### Exemple Usage

```bash
k8s-alias
```

```bash
k .............. kubectl
kgctx .......... kubectl config get-contexts
kuctx .......... kubectl config use-context
kictx .......... kubectl cluster-info --context
kg ............. kubectl get
kgp ............ kubectl get pod
kdesp .......... kubectl describe pod
kdp ............ kubectl delete pod
kgpw ........... kubectl get pod -o wide
kgpall ......... kubectl get pod --all-namespaces
kgd ............ kubectl get deployment
kgdall ......... kubectl get deployment --all-namespaces
kdd ............ kubectl delete deployment
kddall ......... kubectl delete deployment --all
kgrs ........... kubectl get replicaset
kgn ............ kubectl get nodes
kgnl ........... kubectl get nodes --show-labels
kaf ............ kubectl apply -f
kdf ............ kubectl delete -f
kl ............. kubectl log
krp ............ kubectl run -it --image alpine myspec -- sh
krpyaml ........ kubectl run --image nginx nginx --dry-run=client -o yaml
krest .......... krest deployment/nginx
krest .......... krest -n default deployment frontend
kcdyaml ........ kubectl create deployment nginx --image nginx --dry-run=client -o yaml
```

## Ex:
```bash
krest -n kube-system kube-proxy
```

