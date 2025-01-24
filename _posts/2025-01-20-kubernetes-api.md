---
layout: post
collection: kubernetes
permalink: /kubernetes/api
title:  "Explorando API Kubernetes"
author: Paulo Rogério
date:   2025-01-22 20:59:13 -0300
categories: [kubernetes]
published: true
---

## 🚀 Api Kubernetes

- [1) KubeConfig](#1-kubeconfig)
  - [1.1) CA](#11-ca)
  - [1.2) Certificate](#12-certificado)
  - [1.3) Key](#13-certificado)
- [2) Api Rotas Debug](#2-api-rotas-debug)
- [3) Consumindo API](#3-consumindo-api)
  - [3.1) Anonimo](#31-anonimo)
  - [3.2) Autenticando com CA](#32-autenticando-com-ca)
  - [3.3) Autenticando com Usuário e Senha](#33-autenticando-com-usuário-e-senha) 

#### 1) KubeConfig

O kubeconfig é o arquivo que contém as credencias para autenticar em um cluster **Kubernetes**. Para que essa autenticação possa ocorrer é necessário 3 conteúdos presente dentro do arquivo.

- CA ( Autoridade Certificadora ) 
- Certificado ( Certificado do Usuário )
- Chave ( Chave do Usuário )

## 1.1) CA

É a Certificado Raiz **entidade certificadora** criado pelo kubeadm na criação do cluster , responsável por assinar os outros certificados de usuário.

```bash
grep certificate-authority-data < ~/.kube/config | cut -f2 -d : | tr -d ' ' | base64 -d | openssl x509 -text -out -
```

Certificado assinado pelo dono da CA **kubernetes**.

```bash
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 5039711052060191681 (0x45f0a68014a86bc1)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=kubernetes
        Validity
            Not Before: Jan 24 00:35:16 2025 GMT
            Not After : Jan 22 00:40:16 2035 GMT
        Subject: CN=kubernetes
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:97:23:c9:92:a0:24:51:5c:5d:01:d9:bc:57:74:
                    c0:aa:e8:04:00:37:de:18:7c:19:40:f7:aa:b5:b0
```                    

## 1.2) Certificate 

O certificado do usuário e a sua assinatura digital, e a forma como usuário interagi com a API do kubernetes.

```bash
grep client-certificate-data < ~/.kube/config | cut -f2 -d : | tr -d ' ' | base64 -d | openssl x509 -text -out -
```

Certificado do usuário

```bash
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 3044125561371326043 (0x2a3ee811d139765b)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=kubernetes
        Validity
            Not Before: Jan 24 00:35:16 2025 GMT
            Not After : Jan 24 00:40:16 2026 GMT
        Subject: O=kubeadm:cluster-admins, CN=kubernetes-admin
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c9:cb:bc:83:da:e9:a3:55:59:c5:a9:b8:ec:08:
                    4d:c4:2c:2a:3b:d2:ea:c6:ab:60:f6:66:5b:5c:85:
```

## 1.2) Key 

A chave do usuário usada para interagi com a API do kubernetes.

```bash
grep client-key-data < ~/.kube/config | cut -f2 -d : | tr -d ' ' | base64 -d
```

```bash
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAycu8g9rpo1VZxam47AhNxCwqO9Lqxqtg9mZbXIWJVb0YA9Vd
ao3O/701rw3cL5ZuFH6au/LFxRXh/j+stNjLWUfQYiS4Z9WlpUQCm6iVmox0N8GN
7ypAX0GdSstXvr/VjgpLqSlKzODx+VLuNhdcLOML7mfKrCPM1CWne++LQJYQUGk0
```

#### 2) Api Rotas Debug

Para descobrir as chamadas de API que o kubectl executa por detras dos panos, é necessário rodar o comando no modo debug, com o maior nível de verbosidade possível.

```bash
k get deployments -A -v9
```

```bash
I0123 21:55:11.652620   99409 loader.go:395] Config loaded from file:  /Users/PauloRogerio/.kube/config
I0123 21:55:11.671903   99409 round_trippers.go:466] curl -v -XGET  -H "User-Agent: kubectl/v1.31.1 (darwin/amd64) kubernetes/948afe5" -H "Accept: application/json;as=Table;v=v1;g=meta.k8s.io,application/json;as=Table;v=v1beta1;g=meta.k8s.io,application/json" 'https://127.0.0.1:51913/apis/apps/v1/deployments?limit=500'
I0123 21:55:11.678725   99409 round_trippers.go:510] HTTP Trace: Dial to tcp:127.0.0.1:51913 succeed
I0123 21:55:11.710002   99409 round_trippers.go:553] GET https://127.0.0.1:51913/apis/apps/v1/deployments?limit=500 200 OK in 35 milliseconds
I0123 21:55:11.710069   99409 round_trippers.go:570] HTTP Statistics: DNSLookup 0 ms Dial 1 ms TLSHandshake 14 ms ServerProcessing 14 ms Duration 35 ms
I0123 21:55:11.710142   99409 round_trippers.go:577] Response Headers:
I0123 21:55:11.710211   99409 round_trippers.go:580]     Audit-Id: 2be78f03-2d54-46e6-96ae-03e05748568d
I0123 21:55:11.710245   99409 round_trippers.go:580]     Cache-Control: no-cache, private
I0123 21:55:11.710315   99409 round_trippers.go:580]     Content-Type: application/json
```

#### 3) Consumindo API

Agora que temos os endpoints, vamos interagir com o cluster via **curl** para enteder o status code de retorno.

## 3.1) Anonimo

## Interagindo com cluster no modo anonimo

Observer que API retorna um 403 pois não foi autenticado

```bash
curl -k https://127.0.0.1:51913/apis/apps/v1/deployments?limit=500
```

```bash
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "deployments.apps is forbidden: User \"system:anonymous\" cannot list resource \"deployments\" in API group \"apps\" at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "group": "apps",
    "kind": "deployments"
  },
  "code": 403
}
```

```bash
curl -k -I https://127.0.0.1:51913/apis/apps/v1/deployments?limit=500
```

```bash
HTTP/2 403
audit-id: 1407b6ff-d7d6-4ea4-a910-6c1e6c781a22
cache-control: no-cache, private
content-type: application/json
x-content-type-options: nosniff
x-kubernetes-pf-flowschema-uid: b927b57b-07ab-4a7e-9d11-3cc7ac3be3cd
x-kubernetes-pf-prioritylevel-uid: a2b3cdbb-813e-4c08-af23-a01c71793a40
content-length: 345
date: Fri, 24 Jan 2025 01:10:52 GMT
```

## 3.2) Autenticando com CA

Extraindo o certificado

```bash
grep certificate-authority-data < ~/.kube/config | cut -f2 -d : | tr -d ' ' | base64 -d > /tmp/ca.crt
```

Apenas informando a **CA** também não autorizado.

```bash
curl -k -I --cacert /tmp/ca.crt https://127.0.0.1:51913/apis/apps/v1/deployments?limit=500
```

```bash
HTTP/2 403
audit-id: f05ca465-3b24-4792-a13e-fc8b898774d9
cache-control: no-cache, private
content-type: application/json
x-content-type-options: nosniff
x-kubernetes-pf-flowschema-uid: b927b57b-07ab-4a7e-9d11-3cc7ac3be3cd
x-kubernetes-pf-prioritylevel-uid: a2b3cdbb-813e-4c08-af23-a01c71793a40
content-length: 345
date: Fri, 24 Jan 2025 01:11:17 GMT
```

## 3.3) Autenticando com Usuário e Senha

```bash
grep client-certificate-data < ~/.kube/config | cut -f2 -d : | tr -d ' ' | base64 -d > /tmp/user.crt
```

```bash
grep client-key-data < ~/.kube/config | cut -f2 -d : | tr -d ' ' | base64 -d > /tmp/user.key
```

```bash
curl --cacert /tmp/ca.crt --cert /tmp/user.crt --key /tmp/user.key  https://127.0.0.1:51913/apis/apps/v1/deployments?limit=500
```

```bash
...
...
...
      "metadata": {
        "name": "metrics-server",
        "namespace": "kube-system",
        "uid": "ef592d57-00af-4d47-a9a9-0babf1d602a0",
        "resourceVersion": "1415",
        "generation": 1,
        "creationTimestamp": "2025-01-24T00:45:14Z",
        "labels": {
          "app.kubernetes.io/instance": "metrics-server",
          "app.kubernetes.io/managed-by": "Helm",
          "app.kubernetes.io/name": "metrics-server",
          "app.kubernetes.io/version": "0.7.2",
          "helm.sh/chart": "metrics-server-3.12.2"
        },
        "annotations": {
          "deployment.kubernetes.io/revision": "1",
          "meta.helm.sh/release-name": "metrics-server",
          "meta.helm.sh/release-namespace": "kube-system"
        },
...
...
...
```