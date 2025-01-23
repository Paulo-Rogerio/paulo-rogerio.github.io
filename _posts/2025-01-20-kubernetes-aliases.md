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
alias k8s-alias='
                echo "k = kubectl" && \
                echo "kgctx = kubectl config get-contexts" && \
                echo "kuctx = kubectl config use-context" && \
                echo "kictx = kubectl cluster-info --context" && \
                echo "kg = kubectl get" && \
                echo "kgpo = kubectl get pod" && \
                echo "kgpoall = kubectl get pod --all-namespaces"
                '

alias k='kubectl'
alias kgctx='kubectl config get-contexts'
alias kuctx="kubectl config use-context"
alias kictx="kubectl cluster-info --context"
alias kg='kubectl get \$1'
alias kgpo='kubectl get pod'
alias kgpoall='kubectl get pod --all-namespaces'
alias krestpo='kubectl rollout restart deploy'
EOF
```

[Aliases Github](https://github.com/wuestkamp/Kubernetes-Certified-Administrator?tab=readme-ov-file){:target="_blank"}

[BookMarks Kubernetes](https://github.com/reetasingh/CKAD-Bookmarks?tab=readme-ov-file){:target="_blank"}

### Exemple Usage

```bash
krestpo -n kube-system kube-proxy
```

