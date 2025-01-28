---
layout: post
collection: etcd
permalink: /etcd/etcd-manager
title:  "Manager Etcd"
author: Paulo Rogério
date:   2025-01-26 16:07:13 -0300
categories: [etcd]
published: true
---

# 🚀 Gerenciando Cluster Etcd

- [1) Objetivo](#1-objetivo)
- [2) Repositório](#2-repositório)
- [3) Docker ](#3-docker)
  - [3.1) Gerando Imagem](#31-gerando-imagem)
  - [3.2) Docker Compose](#32-docker-compose)
- [4) Criando Cluster Etcd ](#4-criando-cluster-etcd)
  - [4.1) Gerando Certificados](#41-gerando-certificados)
  - [4.2) Iniciando Serviço Etcd](#42-iniciando-serviço-etcd) 
- [5) Manipulando Etcd](#5-manipulando-etcd)     
  - [5.1) Check Status](#51-check-status)
  - [5.2) Endpoint Status](#52-endpoint-status)
  - [5.3) Inserindo Registros](#53-inserindo-registros)
  - [5.4) Lendo Registros](#54-lendo-registros) 
  - [5.5) Deletando Registros](#54-deletando-registros)
- [6) Gerenciando os Membros do Cluster](#6-gerenciando-os-membros-do-cluster)
  - [6.1) Inserindo Novo Membro](#61-inserindo-novo-membro)
  - [6.2) Deletando um Membro](#62-deletando-um-membro)  

#### 1) Objetivo

O objetivo dessa doc, é garantir que você possa exercitar a manipulação de um cluster **etcd**, entender como criar os certificados, iniciar o cluster, adicionar membros e remove-los.

Para execução desse documento é necessário ter o **docker** instalado em seu Host.

#### 1) Repositório

Após garantir que o **docker** esteja presente no hosts, clone o repositório.

[Repositório](https://github.com/Paulo-Rogerio/etcd){:target="_blank"}

```bash
git clone https://github.com/Paulo-Rogerio/etcd.git
```

#### 3) Manipulando Etcd

Uma vez clonado execute será necessário ***buildar*** a imagem com os pacotes necessário para o **etcd**.

##### 3.1) Gerando Imagem

Entre no diretório **images** e execute o script **build.sh**.

```bash
➜  docker git:(main) cd images
➜  docker git:(main)
➜  docker git:(main)
➜  images git:(main) ll
total 16
-rw-r--r--  1 paulo  staff   1,3K 28 Jan 08:39 Dockerfile
-rwxr-xr-x  1 paulo  staff   101B 28 Jan 13:42 build.sh
```

Build...

```bash
➜  images git:(main) sh build.sh
Sending build context to Docker daemon  4.096kB
Step 1/11 : FROM alpine
 ---> b0c9d60fc5e3
Step 2/11 : MAINTAINER psilva.gomes.rogerio@gmail.com
 ---> Running in 908ea3998c54
 ---> Removed intermediate container 908ea3998c54
 ---> eeadeddcd00f
...
...
```

Liste as imagens compiladas...

```bash
➜  docker git:(main) docker images
REPOSITORY              TAG       IMAGE ID       CREATED         SIZE
prgs/etcd               3.5.18    817935ca1793   3 minutes ago   145MB
...
...
...
```

##### 3.2) Docker Compose

Uma vez que a imagms está presente em seu Host, vamos inicar os containers.

O compose abaixo, iniciará 4 **containers** etcd, sendo que o entrypoint das imagens é uma while infinito para evitar
que o container morra.

```yaml
x-etcd-common:
  &etcd-common
  image: prgs/etcd:3.5.18
  volumes:
    - ./shared:/shared
  command: |
    bash -c "while true; do echo running...; sleep 10; done"

services:
  etcd1:
    <<: *etcd-common
    container_name: etcd1
    hostname: etcd1

  etcd2:
    <<: *etcd-common
    container_name: etcd2
    hostname: etcd2

  etcd3:
    <<: *etcd-common
    container_name: etcd3
    hostname: etcd3

  etcd4:
    <<: *etcd-common
    container_name: etcd4
    hostname: etcd4

networks:
  etcd:
    driver: bridge
```

Para iniciar os containers , execute o script **restart.sh**

```bash
➜  docker git:(main) ll
total 40
-rw-r--r--  1 paulo  staff    83B 28 Jan 14:32 README.md
-rw-r--r--@ 1 paulo  staff   520B 28 Jan 14:03 docker-compose.yml
drwxr-xr-x  4 paulo  staff   128B 28 Jan 13:41 images
-rwxr-xr-x  1 paulo  staff    45B 28 Jan 08:39 restart.sh
drwxr-xr-x  6 paulo  staff   192B 28 Jan 13:57 shared
-rwxr-xr-x  1 paulo  staff    18B 28 Jan 08:39 status.sh
-rwxr-xr-x  1 paulo  staff    20B 28 Jan 08:39 stop.sh
➜  docker git:(main)
➜  docker git:(main)
➜  docker git:(main)
➜  docker git:(main) sh restart.sh
[+] Running 5/5
 ✔ Network docker_default  Created                     0.2s
 ✔ Container etcd4         Started                     1.3s
 ✔ Container etcd1         Started                     1.5s
 ✔ Container etcd2         Started                     1.5s
 ✔ Container etcd3         Started                     1.5s
```

Para checar se os **containers** foram criados com sucesso , execute o **status.sh**

```bash
➜  docker git:(main) sh status.sh
NAME      IMAGE              COMMAND                  SERVICE   CREATED         STATUS         PORTS
etcd1     prgs/etcd:3.5.18   "bash -c 'while true…"   etcd1     3 minutes ago   Up 3 minutes
etcd2     prgs/etcd:3.5.18   "bash -c 'while true…"   etcd2     3 minutes ago   Up 3 minutes
etcd3     prgs/etcd:3.5.18   "bash -c 'while true…"   etcd3     3 minutes ago   Up 3 minutes
etcd4     prgs/etcd:3.5.18   "bash -c 'while true…"   etcd4     3 minutes ago   Up 3 minutes
```

#### 4) Criando Cluster Etcd

Com os containers no ar, vamos manipular o cluster ***etcd***. Será criado os certificados, logo após iniciaremos os serviço para então manipularmos o cluster **etcd**.

##### 4.1) Gerando Certificados

Nessa etapa vamos conectar no container ***etcd1***, vamos elege-lo como o Líder do cluster. Essa eleição é gerenciada pelo próprio **etcd**, mas como vamos subir primeiramente o seriço nesse pod , naturalmente ele será o primário.

Conecte-se ao container para criar os certificados. Se observar o docker-compose irá perceber que todos container montam uma pasta chamada **/shared** que será usada para servir alguns scripts auxiliares.

#### OBS.: Conecte-se ao container etcd1

Gerando a CA ...

```bash
➜  docker git:(main) docker exec -it etcd1 bash
etcd1:/#
etcd1:/#
etcd1:/# sh /shared/certificates/01-ca.sh
mkdir: can't create directory 'certs': File exists
2025/01/28 15:43:33 [INFO] generating a new CA key and certificate from CSR
2025/01/28 15:43:33 [INFO] generate received request
2025/01/28 15:43:33 [INFO] received CSR
2025/01/28 15:43:33 [INFO] generating key: rsa-2048
2025/01/28 15:43:33 [INFO] encoded CSR
2025/01/28 15:43:33 [INFO] signed certificate with serial number 199282527108325187656066368740126098416128628370
/shared/certificates
```

Gerando Certificados...

```bash
etcd1:/#
etcd1:/#
etcd1:/# sh /shared/certificates/02-certificates.sh
2025/01/28 15:44:18 [INFO] generate received request
2025/01/28 15:44:18 [INFO] received CSR
2025/01/28 15:44:18 [INFO] generating key: rsa-2048
2025/01/28 15:44:18 [INFO] encoded CSR
2025/01/28 15:44:18 [INFO] signed certificate with serial number 279978118034933400831631814414308346156700762443
2025/01/28 15:44:18 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
/shared/certificates
```

Check Certificado...

```bash
etcd1:/#
etcd1:/#
etcd1:/# sh /shared/certificates/03-check.sh
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            31:0a:a7:2c:42:82:39:58:99:62:8c:5a:9e:cb:cc:e7:a5:79:81:4b
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=BR, ST=Goiania, L=Brasil, O=Kubernetes, OU=Etcd, CN=etcd
        Validity
            Not Before: Jan 28 18:39:00 2025 GMT
            Not After : Jan 28 18:39:00 2026 GMT
        Subject: C=BR, ST=Goiania, L=Brasil, O=Kubernetes, OU=Etcd, CN=etcd
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:e5:20:5d:d3:da:a8:19:80:e0:37:42:22:89:f4:
                    98:1c:b8:ba:25:bf:3d:db:2f:b4:45:6b:85:03:31:
                    15:f8:c3:5f:ce:ed:48:ef:88:5b:ef:b7:25:34:94:
```

Iniciando o Serviço em **foreground**, se pressionado **Crtl + C**, o seriço será interrompido.

```bash
etcd1:/# 
etcd1:/# 
etcd1:/# sh /shared/startup.sh
{"level":"warn","ts":"2025-01-28T15:47:33.215784-0300","caller":"embed/config.go:689","msg":"Running http and grpc server on single port. This is not recommended for production."}
...
...
...
```

#### 4.2) Iniciando Serviço Etcd

Em outro terminal vamos inicar o segundo membro ***ETCD2***

```bash
➜  docker docker ps
CONTAINER ID   IMAGE              COMMAND                  CREATED          STATUS          PORTS     NAMES
e4ef77ba9c3d   prgs/etcd:3.5.18   "bash -c 'while true…"   21 minutes ago   Up 21 minutes             etcd2
0dde325b1dfd   prgs/etcd:3.5.18   "bash -c 'while true…"   21 minutes ago   Up 21 minutes             etcd1
d9225df9f69b   prgs/etcd:3.5.18   "bash -c 'while true…"   21 minutes ago   Up 21 minutes             etcd4
347a53c6d137   prgs/etcd:3.5.18   "bash -c 'while true…"   21 minutes ago   Up 21 minutes             etcd3
```

```bash
➜  docker docker exec -it etcd2 bash
etcd2:/#
etcd2:/#
```

Dentro do container vamos apenas iniciar o serviço, pois o certifido criado pelo **etcd1** é compatilhado entre os containers.

```bash
etcd2:/#
etcd2:/# 
etcd2:/# sh /shared/startup.sh
{"level":"warn","ts":"2025-01-28T15:55:33.395461-0300","caller":"embed/config.go:689","msg":"Running http and grpc server on single port. This is not recommended for production."}
...
...
...
```

Faremos a mesma coisa com o container **etcd3**.

```bash
➜  docker docker exec -it etcd3 bash
etcd3:/#
etcd3:/#
```

```bash
etcd3:/#
etcd3:/#
etcd3:/# sh /shared/startup.sh
{"level":"warn","ts":"2025-01-28T15:57:18.364972-0300","caller":"embed/config.go:689","msg":"Running http and grpc server on single port. This is not recommended for production."}
...
...
...
```

#### 5) Manipulando Etcd

Agora que temos um cluster rodando, a brincadeira começa a ficar interessante. Abra um outro terminal e contecte ao container ***etcd1***.

##### 5.1) Check Status

Check....

```bash
➜  docker docker exec -it etcd1 bash
etcd1:/#
etcd1:/#
etcd1:/# ps fax
PID   USER     TIME  COMMAND
    1 root      0:00 bash -c while true; do echo running...; sleep 10; done
   73 root      0:00 bash
  143 root      0:00 sh /shared/startup.sh
  153 root      3:02 /usr/local/bin/etcd --name etcd1 --client-cert-auth --peer-client-cert-auth --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt --cert-file=/etc/kubernetes/pki/etcd/etcd.crt --key-file
  267 root      0:00 bash
  273 root      0:00 sleep 10
  274 root      0:00 ps fax
etcd1:/#
etcd1:/#
etcd1:/# sh /shared/api/01-list-member.sh
+------------------+---------+-------+--------------------+-------------------------+------------+
|        ID        | STATUS  | NAME  |     PEER ADDRS     |      CLIENT ADDRS       | IS LEARNER |
+------------------+---------+-------+--------------------+-------------------------+------------+
| 437845357edc9660 | started | etcd3 | https://etcd3:2380 | https://172.20.0.3:2379 |      false |
| d06ed975495649c7 | started | etcd2 | https://etcd2:2380 | https://172.20.0.4:2379 |      false |
| f8783ef4b88dbabc | started | etcd1 | https://etcd1:2380 | https://172.20.0.5:2379 |      false |
+------------------+---------+-------+--------------------+-------------------------+------------+
```

##### 5.2) Endpoint Status

```bash
etcd1:/#
etcd1:/#
etcd1:/# sh /shared/api/02-endpoint-health.sh
+--------------------+--------+-------------+-------+
|      ENDPOINT      | HEALTH |    TOOK     | ERROR |
+--------------------+--------+-------------+-------+
| https://etcd1:2379 |   true | 38.251301ms |       |
| https://etcd3:2379 |   true | 40.240264ms |       |
| https://etcd2:2379 |   true | 57.756944ms |       |
+--------------------+--------+-------------+-------+
etcd1:/#
etcd1:/#
```

Informções dos nodes e detalhes de quem é o lider.

```bash
etcd1:/#
etcd1:/#
etcd1:/# sh /shared/api/03-endpoint-status.sh
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://etcd1:2379 | f8783ef4b88dbabc |  3.5.18 |   20 kB |      true |      false |         2 |         12 |                 12 |        |
| https://etcd2:2379 | d06ed975495649c7 |  3.5.18 |   20 kB |     false |      false |         2 |         12 |                 12 |        |
| https://etcd3:2379 | 437845357edc9660 |  3.5.18 |   20 kB |     false |      false |         2 |         12 |                 12 |        |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
etcd1:/#
etcd1:/#
etcd1:/#
```

##### 5.3) Inserindo Registros

O Banco do **etcd** armazena dados no formato chave, valor. Inserindo registros....

```bash
etcd1:/# sh /shared/api/04-insert-item.sh
OK
OK
OK
etcd1:/#
```

##### 5.4) Lendo Registros

```bash
etcd1:/# sh /shared/api/05-read-item.sh
---------
chave1
value1
---------
chave2
value2
---------
chave3
value3
---------
etcd1:/#
```

##### 5.5) Deletando Registros

```bash
etcd1:/# sh /shared/api/06-delete-item.sh
1
etcd1:/#
```

Check ...

```bash
etcd1:/# 
---------
chave1
value1
---------
chave2
value2
---------
---------
```


#### 6) Gerenciando os Membros do Cluster

Ao lidar com cluster de qualquer tipo de serviço temos que ter a habilidade de conseguir remover e adicionar membros no cluster. Se observamos o nosso compose subimos 4 container, mas apenas os membros **( etcd1, etcd2, etcd3 )**, faz parte do cluster.

Vamos inserir um novo membro a um cluster existente.

##### 6.1) Inserindo Novo Membro

Ainda conectado no contariner ***etcd1***, mas poderia ser executado em qualquer membro do cluster o processo de inserção do novo membro.

```bash
etcd1:/#
etcd1:/#
etcd1:/# sh /shared/api/07-add-member.sh
Member 42daf3d2139377a2 added to cluster 22d85c70e44661b0

ETCD_NAME="etcd4"
ETCD_INITIAL_CLUSTER="etcd4=https://etcd4:2380,etcd3=https://etcd3:2380,etcd2=https://etcd2:2380,etcd1=https://etcd1:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://etcd4:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
etcd1:/#
etcd1:/#
```

Agora abra um novo terminal e conecte-se ao container ***etcd4***

```bash
➜  docker ps
CONTAINER ID   IMAGE              COMMAND                  CREATED          STATUS          PORTS     NAMES
e4ef77ba9c3d   prgs/etcd:3.5.18   "bash -c 'while true…"   55 minutes ago   Up 55 minutes             etcd2
0dde325b1dfd   prgs/etcd:3.5.18   "bash -c 'while true…"   55 minutes ago   Up 55 minutes             etcd1
d9225df9f69b   prgs/etcd:3.5.18   "bash -c 'while true…"   55 minutes ago   Up 55 minutes             etcd4
347a53c6d137   prgs/etcd:3.5.18   "bash -c 'while true…"   55 minutes ago   Up 55 minutes             etcd3
```

Uma vez que ele foi aceito como membro, vamos inicia-lo.

```bash
docker exec -it etcd4 bash
etcd4:/#
etcd4:/# 
etcd4:/# sh /shared/startup-new-member.sh
{"level":"warn","ts":"2025-01-28T16:28:04.840858-0300","caller":"embed/config.go:689","msg":"Running http and grpc server on single port. This is not recommended for production."}
...
...
```

Se voltarmos ao container **etcd1** e checar o status do cluster, veremos que temos um novo membro...

```bash
etcd1:/#
etcd1:/#
etcd1:/# sh /shared/api/01-list-member.sh
+------------------+---------+-------+--------------------+-------------------------+------------+
|        ID        | STATUS  | NAME  |     PEER ADDRS     |      CLIENT ADDRS       | IS LEARNER |
+------------------+---------+-------+--------------------+-------------------------+------------+
| 42daf3d2139377a2 | started | etcd4 | https://etcd4:2380 | https://172.20.0.2:2379 |      false |
| 437845357edc9660 | started | etcd3 | https://etcd3:2380 | https://172.20.0.3:2379 |      false |
| d06ed975495649c7 | started | etcd2 | https://etcd2:2380 | https://172.20.0.4:2379 |      false |
| f8783ef4b88dbabc | started | etcd1 | https://etcd1:2380 | https://172.20.0.5:2379 |      false |
+------------------+---------+-------+--------------------+-------------------------+------------+
etcd1:/#
etcd1:/#
etcd1:/#
```

#### 6.2) Deletando um Membro

Vamos imaginar um cenário onde é necessário remover um membro. No nosso exemplo vamos remover o membro **etcd2**.

O script espera que seja informado um **Id**, caso não seja informado, será retornado a seguinte mensagem.

```bash
Member Id não informado. Liste os membros para obter o Id
```

```bash
etcd1:/#
etcd1:/#
etcd1:/# sh /shared/api/08-delete-member.sh d06ed975495649c7
Member d06ed975495649c7 removed from cluster 22d85c70e44661b0
etcd1:/#
etcd1:/#
```

Check se o membro foi removido com sucesso....

```bash
etcd1:/#
etcd1:/#
etcd1:/# sh /shared/api/01-list-member.sh
+------------------+---------+-------+--------------------+-------------------------+------------+
|        ID        | STATUS  | NAME  |     PEER ADDRS     |      CLIENT ADDRS       | IS LEARNER |
+------------------+---------+-------+--------------------+-------------------------+------------+
| 42daf3d2139377a2 | started | etcd4 | https://etcd4:2380 | https://172.20.0.2:2379 |      false |
| 437845357edc9660 | started | etcd3 | https://etcd3:2380 | https://172.20.0.3:2379 |      false |
| f8783ef4b88dbabc | started | etcd1 | https://etcd1:2380 | https://172.20.0.5:2379 |      false |
+------------------+---------+-------+--------------------+-------------------------+------------+
etcd1:/#
etcd1:/#
```

Mais operações com ETCD pode ser obtida por meio da documentação oficial 

[Etcd.Io](https://etcd.io/){:target="_blank"}