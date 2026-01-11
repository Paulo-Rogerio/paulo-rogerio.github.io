---
layout: post
collection: kubernetes
permalink: /kubernetes/argocd-pipeline
title:  "Argocd Pipeline"
author: Paulo Rogério
date:   2026-01-10 12:18:00 -0300
categories: [kubernetes]
published: true
---

## 🚀 ArgoCD Pipeline 

- [1) Escopo do Projeto](#1-escopo-do-projeto)
- [2) Minha Aplicação](#2-minha-aplicação)
- [3) Ferramentas Necessárias](#3-ferramentas-necessárias)
- [4) Rodando Minha Aplicação Localmente](#4-rodando-minha-aplicação-localmente)
- [5) Configurando Ambiente para Pipeline](#5-configurando-ambiente-para-pipeline)
- [6) Fluxo da Pipeline](#6-fluxo-da-pipeline)
- [7) RollBack](#7-rollback)
- [8) Youtube Demo](#8-youtube-demo)

#### 1) Escopo do Projeto

Projeto que contempla o aprendizado de pipelines usando **Gitlab**. O projeto foi dividido em vários **Repositórios**, simulando um ambiente produtivo.

![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-01.png "ArgoCD Projects")

 
| Repositóorio | Descrição |
| --- | --- |
| **Java-App** | Repositório que contém aplicação Java |
| **CICD** | Repositóro que contém manifestos usados pela Pipeline do Gitlab |
| **Docker** | Repositório que Dockerfiles usado na InfraEstrutura e Pipelines |
| **Helm-Chart** | Repositório para armazenamento dos Package Registry |


#### 2) Minha Aplicação

Se trata de uma aplicação **Java** que usa **maven** para buildar e gerar o artefato. É uma aplicação bem simples que implementa uma classe que estende **HttpServlet** e implementa a lógica para responder requisições HTTP nesta rota **hello-world/hello**.

```bash
├── hello-world
│   ├── src
        └── main
            ├── java
            │   └── prgs
            │       └── corp
            │           └── HelloWorldServlet.java
            └── webapp
                ├── index.jsp
                └── WEB-INF
                    └── web.xml
```

Voce verá outros diretórios que não fazem parte da aplicação em sí **Ex: k8s**, porém é utlizado pela pipeline. Já a pasta **scripts**, não se faz necessário mante-la, mas foi mantido apenas para fins didáticos.

```bash
drwxrwxr-x 6 paulo paulo 4096 Dec 11 10:36 k8s
drwxrwxr-x 2 paulo paulo 4096 Jan  2 15:54 scripts
```

#### 3) Ferramentas Necessárias

Para esse laboratório vamos precisar das seguintes ferramentas:

| Ferramenta | Descrição |
| --- | --- |
| **JDK** | Compilador do Java |
| **Maven** | Compilar, Empacotar e Gerar o .WAR |
| **Docker** | Buildar Imagem |
| **Docker Compose** | Executar Imagem pré-compilada |
| **Kind** | Executor do Kubernetes configurado ( Gateway API, ArgoCD, MetalLB ) |
| **Runner Gitlab Kubernetes** | Uma conta no Gitlab e Tokens previamente configurados para criar Runners Locais |
| **Helm** | Gerenciador de Pacotes para o Kubernetes |
| **ArgoCD** | Gerenciar Deploys |
| **MetalLB** | Expor IP para service do Type LoadBalancer |


Outro detalhe ainda relacionados a pré-requisitos , são as **variáveis sensíveis**. Para esse Demo, meu projeto requer essas variáveis:


| Variáveis | Descrição |
| --- | --- |
| **PIPELINE_DEPLOY_GIT_MAIL** | Email conhecido pelo git - Gerar Tag |
| **PIPELINE_DEPLOY_GIT_NAME** | Usuario conhecido pelo git - Gerar Tag |
| **PIPELINE_DEPLOY_TOKEN** | Token do Gitlab usado para gerenciar os artefatos helm ( Registry Package ) |
| **PROJECT_ID** | Identificador do Projeto helm-charts para que helm saiba aonde deve levar os pacotes compilados |
| **TLS_CRT** | Certificado gerado pelo letsencrypt |
| **TLS_KEY** | Key do certificado gerado pelo letsencrypt |

![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-21.png "Gitlab Variaveis")

#### 4) Rodando Minha Aplicação Localmente

Criar uma branch para contemplar minhas mudancas 

```bash
git checkout -b chore/azul
```

Na raiz do projeto , existe um script **run.sh**, vou executa-lo para subir minha aplicação localmente. Aqui é o fluxo do dia a dia na vida do desenvolvedor.

```bash
-rw-rw-r-- 1 paulo paulo  206 Jan  6 21:01 .env
drwxrwxr-x 8 paulo paulo 4096 Jan 10 11:50 .git
-rw-rw-r-- 1 paulo paulo  155 Jan 10 11:50 .gitignore
-rw-rw-r-- 1 paulo paulo  124 Jan  1 15:55 .gitlab-ci.yml
drwxrwxr-x 5 paulo paulo 4096 Jan  6 20:43 hello-world
-rw-rw-r-- 1 paulo paulo  176 Dec 29 12:00 .pre-commit-config.yaml
-rw-rw-r-- 1 paulo paulo 3093 Jan  6 20:43 README.md
-rwxrwxr-x 1 paulo paulo 2542 Jan  6 21:01 run.sh
```

Conteúdo do **.env**

```bash
TIMESTAMP="20260110-151312"
VERSION="0.0.2"
RELEASE="${VERSION}-${TIMESTAMP}"
_BRANCH=$(git branch --show-current)
_BRANCH=${_BRANCH//\//-}
BRANCH="${_BRANCH:-$CI_COMMIT_BRANCH}"
TAG="${RELEASE}-${BRANCH}"
```

Edite o **.run.sh** para definir o cor de fundo **background color** .

```bash
export COLOR="#0080ff" # azul
```

O que esse script **run.sh** faz?

* replace do conteúdo **.env** substituindo o conteúdo do timestemp para a hora current.
* maven build
* docker build
* inicia docker compose

Execute o **run.sh**

```bash
$ ./run.sh
Build maven...
[INFO] Scanning for projects...
[INFO] 
[INFO] -----------------------< prgs.corp:hello-world >------------------------
[INFO] Building hello-world 0.0.2
[INFO]   from pom.xml
[INFO] --------------------------------[ war ]---------------------------------
[INFO] 
[INFO] --- versions:2.20.1:set (default-cli) @ hello-world ---
[INFO] Searching for local aggregator root...
[INFO] Local aggregation root: /home/paulo/Documents/Projetos/gitlab/meetups-prgs/argocd-chart/java-app/hello-world
[INFO] Processing change of prgs.corp:hello-world:0.0.2 -> 0.0.2
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  0.560 s
[INFO] Finished at: 2026-01-10T15:23:20-03:00
[INFO] ------------------------------------------------------------------------
[INFO] Scanning for projects...
[INFO] 
[INFO] -----------------------< prgs.corp:hello-world >------------------------
[INFO] Building hello-world 0.0.2
[INFO]   from pom.xml
[INFO] --------------------------------[ war ]---------------------------------
[INFO] 
[INFO] --- clean:3.2.0:clean (default-clean) @ hello-world ---
[INFO] 
[INFO] --- resources:3.3.1:resources (default-resources) @ hello-world ---
[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!
[INFO] skip non existing resourceDirectory /home/paulo/Documents/Projetos/gitlab/meetups-prgs/argocd-chart/java-app/hello-world/src/main/resources
[INFO] 
[INFO] --- compiler:3.13.0:compile (default-compile) @ hello-world ---
[INFO] Recompiling the module because of changed source code.
[WARNING] File encoding has not been set, using platform encoding UTF-8, i.e. build is platform dependent!
[INFO] Compiling 1 source file with javac [debug target 1.8] to target/classes
[INFO] 
[INFO] --- resources:3.3.1:testResources (default-testResources) @ hello-world ---
[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!
[INFO] skip non existing resourceDirectory /home/paulo/Documents/Projetos/gitlab/meetups-prgs/argocd-chart/java-app/hello-world/src/test/resources
[INFO] 
[INFO] --- compiler:3.13.0:testCompile (default-testCompile) @ hello-world ---
[INFO] No sources to compile
[INFO] 
[INFO] --- surefire:3.2.5:test (default-test) @ hello-world ---
[INFO] No tests to run.
[INFO] 
[INFO] --- war:3.3.2:war (default-war) @ hello-world ---
[INFO] Packaging webapp
[INFO] Assembling webapp [hello-world] in [/home/paulo/Documents/Projetos/gitlab/meetups-prgs/argocd-chart/java-app/hello-world/target/hello-world-0.0.2]
[INFO] Processing war project
[INFO] Copying webapp resources [/home/paulo/Documents/Projetos/gitlab/meetups-prgs/argocd-chart/java-app/hello-world/src/main/webapp]
[INFO] Building war: /home/paulo/Documents/Projetos/gitlab/meetups-prgs/argocd-chart/java-app/hello-world/target/hello-world-0.0.2.war
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  0.975 s
[INFO] Finished at: 2026-01-10T15:23:22-03:00
[INFO] ------------------------------------------------------------------------
/home/paulo/Documents/Projetos/gitlab/meetups-prgs/argocd-chart/java-app
Replace Helm Chart...
Docker Build...
[+] Building 0.3s (7/7) FINISHED                                                                                                                                               docker:default
 => [internal] load build definition from Dockerfile                                                                                                                                     0.0s
 => => transferring dockerfile: 260B                                                                                                                                                     0.0s
 => [internal] load metadata for docker.io/jboss/wildfly:25.0.0.Final                                                                                                                    0.0s
 => [internal] load .dockerignore                                                                                                                                                        0.0s
 => => transferring context: 2B                                                                                                                                                          0.0s
 => [internal] load build context                                                                                                                                                        0.0s
 => => transferring context: 2.83kB                                                                                                                                                      0.0s
 => CACHED [1/2] FROM docker.io/jboss/wildfly:25.0.0.Final                                                                                                                               0.0s
 => [2/2] COPY ./target/hello-world-0.0.2.war /opt/jboss/wildfly/standalone/deployments/hello-world.war                                                                                  0.1s
 => exporting to image                                                                                                                                                                   0.1s
 => => exporting layers                                                                                                                                                                  0.0s
 => => writing image sha256:4ec154cf0bcfbc97efc8f800788425690d36ec4de2bd41f1569124855f4f02f9                                                                                             0.0s
 => => naming to my-registry:5000/hello-world:0.0.2-20260110-152319-chore-azul                                                                                                           0.0s
/home/paulo/Documents/Projetos/gitlab/meetups-prgs/argocd-chart/java-app
Docker Compose...
[+] Running 2/2
 ✔ Container app                    Removed                                                                                                                                              0.6s 
 ✔ Network docker-compose_app_java  Removed                                                                                                                                              0.1s 
[+] Running 2/2
 ✔ Network docker-compose_app_java  Created                                                                                                                                              0.0s 
 ✔ Container app                    Started                                                                                                                                              0.2s 
Iniciando App...Done
```

Observer que o docker **build** gerou uma imagem usando a versao **Pom.xml** concatenado com o timmestemp definido **.env** mais a sua **branch**. 

```bash
$ docker ps 
CONTAINER ID   IMAGE                                                           COMMAND                  CREATED              STATUS              PORTS                                         NAMES
a083cad6fb1e   my-registry:5000/hello-world:0.0.2-20260110-152319-chore-azul   "/opt/jboss/wildfly/…"   About a minute ago   Up About a minute   0.0.0.0:9999->8080/tcp, [::]:9999->8080/tcp   app

```

![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-02.png "Projects Running Local")


##### OBS.: Ainda não commitei minha mudancas

```bash
$git status
On branch chore/azul
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   .env
	modified:   hello-world/k8s/docker-compose/docker-compose.yml
	modified:   hello-world/k8s/helm/hello/Chart.yaml
	modified:   hello-world/src/main/java/prgs/corp/HelloWorldServlet.java
```

#### 5) Configurando Ambiente para Pipeline

Agora vamos subir nossa infra para rodar nossa pipeline. Estou assumindo que as [Ferramentas Necessárias](#3-ferramentas-necessárias) já estejam instaladas.
Caso não tenha, vou deixar um link aqui com a instrução de deploy do kind.

<a href="https://github.com/Paulo-Rogerio/provisioner-kubernetes/tree/main/kind-linux-gateway-api" target="_blank">Deploy Kind Kubernetes</a>

![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-03.png "Kind Running Local")


###### Configurando TLS Letsencrypt

Meu dominio **prgs-corp.xyz** é valido, mas para gerar um certificado, tenho que fazer isso manualmente, visto que **Hostinger**, não tem uma integração com **cert-manager**. Entao a solução foi definir isso manualmente

```yaml
services:
  certbot:
    container_name: certbot
    restart: always
    entrypoint: ["/bin/sh"]
    tty: true
    stdin_open: true
    networks:
      - certbot
    image: certbot/certbot:latest
    volumes:
      - ./letsencrypt:/etc/letsencrypt/archive
      - ./generate.sh:/generate.sh

networks:
  certbot:
    driver: bridge
```

Script **generate.sh** 

```bash
#!/bin/sh
certbot certonly \
  --manual \
  --preferred-challenges dns \
  --email <seu-email> \
  --agree-tos \
  --no-eff-email \
  -d "*.<seu-dominio>"
```

```bash
docker exec -it certbot sh -c "/generate.sh"
```

Ao gerar o certificado, deve-se informar no DNS da Hostinger o token gerado para que Letsencrypt funcione.


###### Pre-Commit

Install [Pre-Commit](https://pre-commit.com/) no seu projeto.

```bash
pre-commit install
```

###### Configurando Gitlab-CI Runner Service Account 

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: gitlab-runner-admin
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gitlab-runner-admin-binding
subjects:
  - kind: ServiceAccount
    name: gitlab-runner
    namespace: gitlab-runner
roleRef:
  kind: ClusterRole
  name: gitlab-runner-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```

###### Install Repo Gitlab Runner

Adicione o repositório e liste as versões disponíveis do pacote.

```bash
helm repo add gitlab https://charts.gitlab.io
helm search repo -l gitlab/gitlab-runner
```

###### Defina o Values.yaml

Nessa etapa é necessário habilitar um **Runner** no seu Gitlab. Ao criar será disponibilizado um **token**. Esse token deve ser armazenado nessa variavel **GITLAB_RUNNER_TOKEN** para que o processo continue.

![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-04.png "Token Runner")


```bash
cat > values.yaml <<EOF
gitlabUrl: https://gitlab.com/
runnerRegistrationToken: "${GITLAB_RUNNER_TOKEN}"
concurrent: 10
checkInterval: 30
rbac:
  create: true
  clusterWideAccess: true
  rules:
    - apiGroups: ["*"]
      resources: ["*"]
      verbs: ["*"]

runners:
  privileged: true
  config: |
    [[runners]]
      name = "kubernetes-runner"
      tags = ["kubernetes"]
      run_untagged = false

      [runners.kubernetes]
        namespace = "gitlab-runner"
        service_account = "gitlab-runner"
        tls_verify = false
        image = "ubuntu:noble"
        privileged = true
EOF
```

###### Install Runner via Helm

```bash
helm install --create-namespace --namespace gitlab-runner gitlab-runner -f values.yaml gitlab/gitlab-runner --version 0.84.1
```

Se tudo ocorreu bem, deverá ver um **Pod** do gitlab no seu cluster com status **Running**.

![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-05.png "Gitlab Runner Status")


###### Configurando Registry Local

Para facilitar subi um **Registry** Local, rodando em docker. A ideia e agillizar o processo de **push e pull**. Para que os Pods possa enchegar o container é necessário ajusar o **etc/hosts** e garantir que o registry rode na mesma rede que o kind.

```bash
IP=$(ip -4 addr show wlo1 | awk '/inet /{print $2}' | cut -d/ -f1)
docker exec prgs-control-plane sh -c "echo \"$IP my-registry\" >> /etc/hosts"
docker exec prgs-control-plane sh -c "cat /etc/hosts"
sudo sed -i "/^[^#].*my-registry/s/^[0-9.]\+/$IP/" /etc/hosts
```

###### Buildar Imagem Docker usada na pipeline.

Para esse laboratório essa imagem abaixo conterá todos os executáveis necessários para as minha esteiras.

```yaml
image: "my-registry:5000/wildfly:pipeline"
```

```bash
FROM alpine:3.20

ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk \
    MAVEN_HOME=/opt/maven \
    WILDFLY_HOME=/opt/wildfly \
    PATH=$PATH:/opt/maven/bin:/opt/wildfly/bin
    
ARG MAVEN_VERSION=3.9.12
ARG WILDFLY_VERSION=38.0.1.Final

RUN apk update && \
    apk add --no-cache \
    openjdk17 \
    curl \
    bash \
    tar \
    gzip \
    ca-certificates \
    jq \
    git \
    docker \
    helm \
    kubectl 

...
...
...
```

Clone o repositório que contém o docker e compile a imagem

```bash
git clone https://gitlab.com/meetups-prgs/argocd-chart/docker.git
```

```bash
sh build.sh
```

```bash
$ docker images
IMAGE                                                           ID             DISK USAGE   CONTENT SIZE   EXTRA
alpine/git:latest                                               32c3258b2ea4       97.6MB             0B        
alpine:latest                                                   e7b39c54cdec       8.44MB             0B        
certbot/certbot:latest                                          59e4ba50b513        191MB             0B        
gbevan/ubuntu-foreman:latest                                    c29502d0d240        1.5GB             0B        
ghcr.io/k3d-io/k3d-proxy:5.8.3                                  49c793b9faf6       63.1MB             0B        
ghcr.io/k3d-io/k3d-tools:5.8.3                                  8622faa0d552       20.7MB             0B        
jboss/wildfly:25.0.0.Final                                      856694040847        736MB             0B        
joohoi/acme-dns:latest                                          f475da35c948       49.9MB             0B        
kurzdigital/hammer-katello:latest                               0b8f3a6f5633        380MB             0B        
minio/minio:latest                                              69b2ec208575        175MB             0B        
my-registry:5000/hello-world:0.0.2-20260101-082739-main         4461c2dfca3c        736MB             0B         
my-registry:5000/hello-world:0.0.2-20260101-104146-chore-pops   e293b35a1e96        736MB             0B        
my-registry:5000/hello-world:0.0.2-20260101-105016-main         0d8b35094458        736MB             0B        
my-registry:5000/wildfly:pipeline                               82e7931acd08       1.04GB             0B     
```


#### 6) Fluxo da Pipeline

Tudo pronto para colocar no ar nosso produto. Primeiro passo então e commitar e fazer push. Ao fazer isso o runner irá ler o **.gitlab-ci.yml** que está na raiz do projeto para executar a ação.

```yaml
 include:
   - project: 'meetups-prgs/argocd-chart/cicd'
     ref: 'main'
     file: 'projetos/demo/hello-world-deploy.yml'
```

```bash
git status
On branch chore/azul
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   .env
	modified:   hello-world/k8s/docker-compose/docker-compose.yml
	modified:   hello-world/k8s/helm/hello/Chart.yaml
	modified:   hello-world/src/main/java/prgs/corp/HelloWorldServlet.java
```

```bash
git commit -am "Azul" 
check yaml...............................................................Passed
fix end of files.........................................................Passed
trim trailing whitespace.................................................Passed
[chore/azul 064dad2] Azul
 4 files changed, 5 insertions(+), 5 deletions(-)
```


```bash
git push -u origin chore/azul                                         ✔  1044  16:18:16
Warning: Permanently added 'gitlab.com' (ED25519) to the list of known hosts.
Enumerating objects: 31, done.
Counting objects: 100% (31/31), done.
Delta compression using up to 12 threads
Compressing objects: 100% (11/11), done.
Writing objects: 100% (16/16), 1.17 KiB | 600.00 KiB/s, done.
Total 16 (delta 6), reused 0 (delta 0), pack-reused 0
remote: 
remote: To create a merge request for chore/azul, visit:
remote:   https://gitlab.com/meetups-prgs/argocd-chart/java-app/-/merge_requests/new?merge_request%5Bsource_branch%5D=chore%2Fazul
remote: 
To gitlab.com:meetups-prgs/argocd-chart/java-app.git
   7154af3..064dad2  chore/azul -> chore/azul
branch 'chore/azul' set up to track 'origin/chore/azul'.
```

Qualquer **branch** exceto a **main** é submetida a **TESTES**. 

![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-06.png "Pipeline Branch Tarefa")

###### Vamos subir para Sandbox?

Alguém precisa testar o que fizemos, entao vamos subir para sandbox. Minha estratégia é definir a branch **sandbox** como um gatilho. Isso quer dizer que caso ela exista , ela deve ser destuida e recriada. Ela é volátil, ninguém deve programar nela.

![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-07.png "Pipeline Branch Sandbox")


![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-08.png "Pipeline Branch Sandbox")

O que esse estágio fez?

* Lint
* Maven Test
* Maven Build ( Gerando War )
* Docker Build
* Helm Package ( Gerando Pacote do Helm )
* ArgoCD Deploy
* CleanUP ( Limpando os artefatos / package do Helm )


![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-09.png "ArgoCD")


###### Como ver os pacotes disponiveis no Registry do Helm que o ArgoCD utiliza?

```bash
helm repo add \
  --username gitlab-ci-token \
  --password ${CI_JOB_TOKEN} hello \
  ${CI_API_V4_URL}/projects/${PROJECT_ID}/packages/helm/stable

helm repo update
helm repo list
helm search repo hello --versions --devel
```

```bash
NAME       	CHART VERSION                	APP VERSION                  	DESCRIPTION                
hello/hello	0.0.2-20260110-152319-sandbox	0.0.2-20260110-152319-sandbox	A Helm chart for Kubernetes
hello/hello	0.0.2-20260106-204347-sandbox	0.0.2-20260106-204347-sandbox	A Helm chart for Kubernetes
hello/hello	0.0.2-20260106-204347-main   	0.0.2-20260106-204347-main   	A Helm chart for Kubernetes
hello/hello	0.0.2-20260106-201851-main   	0.0.2-20260106-201851-main   	A Helm chart for Kubernetes
```

Confirmando Deploy em **Sandbox**

![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-10.png "Deploy Sandbox")

###### Validação em Sandbox com sucesso, vamos levar o código para Produção?

Criar um **Merge Request** da branch sandbox para **main**.

![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-11.png "Merge Request")


![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-12.png "Pipeline Merge Request")

O que esse estágio fez?

* Gerou uma Tag no Git
* Checkout na Tag e Maven Build ( Gerando War )
* Checkout na Tag e Docker Build
* Checkout na Tag e Helm Package ( Gerando Pacote do Helm )
* Checkout na Tag e ArgoCD Deploy
* CleanUP ( Limpando os artefatos / package do Helm )

![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-13.png "ArgoCD Production")

Após deploy finalizar, podemos checar se a aplicação realmente está funcionando.

![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-14.png "Deploy Production App")

#### 7) RollBack


Grilaram com esse cor de fundo **azul**, e querem colocar um **rosa**. Vamos criar uma branch e fazer os ajustes...

Edite o arquivo **run.sh** e descomente a linha correspondente ao **rosa**.

```bash
#===================================
# Replace Color Background
#===================================
# export COLOR="#0080ff" # azul
# export COLOR="#ff0000" # vermelho
export COLOR="#ff00d5" # rosa
# export COLOR="#00ff3f" # verde
# export COLOR="#ff7f00" # laranja
# export COLOR="#71246f" # roxo
# export COLOR="#00ffff" # ciano
```

```bash
./run.sh
```

Ao chamar o browser localmente podemos ver que a mudanca aconteceu...

![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-15.png "Deploy Change Color")

* Commit  
* Push
* Gerar Pipeline ( Teste na Branch )
* Subir para Sandbox
* Mergear para Main

Ao criar a branch **sandbox** a URL de sandbox ja deverá está rosa.

![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-16.png "Deploy Sandbox Color Rosa")

Apos o Merge... meu produção também estará com o backgroud **rosa**.


![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-17.png "Deploy Production Color Rosa")


```bash
helm repo update
helm repo list
helm search repo hello --versions --devel

NAME       	CHART VERSION                	APP VERSION                  	DESCRIPTION                
hello/hello	0.0.2-20260110-173445-sandbox	0.0.2-20260110-173445-sandbox	A Helm chart for Kubernetes
hello/hello	0.0.2-20260110-173445-main   	0.0.2-20260110-173445-main   	A Helm chart for Kubernetes
hello/hello	0.0.2-20260110-170530-sandbox	0.0.2-20260110-170530-sandbox	A Helm chart for Kubernetes
hello/hello	0.0.2-20260110-170530-main   	0.0.2-20260110-170530-main   	A Helm chart for Kubernetes
```

Preciso voltar para essa TAG **0.0.2-20260110-170530-main** pois ela é pneultima e está correta.

![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-18.png "Deploy RollBack Azul")

![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-19.png "Deploy RollBack Azul")

![alt text](../../../../images/kubernetes/argocd-pipeline/argocd-pipeline-20.png "Deploy RollBack Azul")

Rollback realizado com sucesso !!!

#### 8) Youtube Demo

Você pode acompanhar todo esse processo de implementação no link abaixo.

<a href="https://youtu.be/j1s5M-70CLw" target="_blank">Demo</a>
