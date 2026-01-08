---
layout: post
collection: kubernetes
permalink: /kubernetes/external-secrets
title:  "External Secrets"
author: Paulo Rogério
date:   2026-01-07 22:34:00 -0300
categories: [kubernetes]
published: true
---

## 🚀 External Secrets 

- [1) Secrets Vault](#1-secrets-vault)
- [2) Repositórios](#2-repositórios)
- [3) Deploy](#3-deploy)
- [4) Youtube Demo](#4-youtube-demo)

#### 1) Secrets Vault


Esse material foi inspirado no material do Jeferson <a href="https://www.youtube.com/watch?v=NlQCTuWXuGk" target="_blank">LINUXtips</a>, porém aqui faço integracao do vault com **LDAP e PostgreSQL**. 

Para que essa integração possa funcionar é necessário que o **Kind** esteja rodando na mesma rede que os containers **Docker** dos serviços externos (***PostgreSQL / LDAP***). 

```yaml
x-postgres: &postgres-common
  image: postgres:15
  restart: always
  networks:
    - kind
  healthcheck:
    test: 'pg_isready -U postgres --dbname=postgres'
    interval: 10s
    timeout: 5s
    retries: 5

services:

  postgres:
    <<: *postgres-common
    container_name: postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: "123456"
      POSTGRES_DB: postgres
      POSTGRES_HOST_AUTH_METHOD: "scram-sha-256"
      POSTGRES_INITDB_ARGS: "--auth-host=scram-sha-256"
      PGDATA: "/data"
    command: |
      postgres
      -c wal_level=replica
      -c hot_standby=on
      -c max_wal_senders=10
      -c max_replication_slots=10
      -c hot_standby_feedback=on
    volumes:
      - ./_data/pgdata/postgres:/data
      - ./_data/scripts:/scripts
    ports:
      - 5433:5432

  openldap:
    image: osixia/openldap:latest
    container_name: openldap
    hostname: openldap
    ports:
      - "389:389"
      - "636:636"
    volumes:
      - ./_data/ldif:/ldif
      - ./_data/scripts:/scripts
    environment:
      - LDAP_ORGANISATION=prgs
      - LDAP_DOMAIN=prgs.corp
      - LDAP_ADMIN_USERNAME=admin
      - LDAP_ADMIN_PASSWORD=123456
      - LDAP_CONFIG_PASSWORD=123456
      - "LDAP_BASE_DN=dc=prgs,dc=corp"
      - LDAP_TLS_CRT_FILENAME=server.crt
      - LDAP_TLS_KEY_FILENAME=server.key
      - LDAP_TLS_CA_CRT_FILENAME=ca.crt
      - LDAP_TLS_VERIFY_CLIENT="allow"
      - ADMIN_BASIC_AUTHORIZATION_ENABLED=false
      - ADMIN_LDAP_AUTHORIZATION_ENABLED=true
    extra_hosts:
      - "openldap.prgs.corp:127.0.0.1"
    networks:
      - kind

  phpldapadmin:
    image: osixia/phpldapadmin:latest
    container_name: phpldapadmin
    hostname: phpldapadmin
    ports:
      - "8080:80"
    environment:
      - PHPLDAPADMIN_LDAP_HOSTS=openldap
      - PHPLDAPADMIN_HTTPS=false
    depends_on:
      - openldap
    networks:
      - kind

networks:
  kind:
    driver: bridge
```

Será necessário ajustar seu **/etc/hosts** para simular um DNS local


```bash
# Estudos Vault
#==============================
192.168.100.95     postgres
172.19.0.241       vault.prgs-corp.xyz
#==============================
```

#### 2) Repositórios

Segue o link do repositório dos estudos.

<a href="https://gitlab.com/meetups-prgs/external-secrets/vault" target="_blank">External Secrets</a>


#### 3) Deploy

Em cada pasta existe um script que carrega os demais scripts necessário para a compilação do produto final. 

##### Iniciar os containers docker

```bash
cd 01-docker-compose
sh restart.sh
```

* Sobe os conteiners - PostgreSQL e LDAP 
* Configura o Banco
* Configura o LDAP 

```bash
 Running 4/4
 ✔ Network 01-docker-compose_kind  Created                                                                                                                                               0.1s 
 ✔ Container openldap              Started                                                                                                                                               0.5s 
 ✔ Container postgres              Started                                                                                                                                               0.4s 
 ✔ Container phpldapadmin          Started                                                                                                                                               0.7s 
sleep 15...
DO
CREATE DATABASE
BEGIN
REVOKE
REVOKE
GRANT
ALTER DEFAULT PRIVILEGES
ALTER DEFAULT PRIVILEGES
ALTER DEFAULT PRIVILEGES
ALTER DEFAULT PRIVILEGES
GRANT
GRANT
GRANT
GRANT
COMMIT
DO
                             List of roles
 Role name |                         Attributes                         
-----------+------------------------------------------------------------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS
 vault     | 

 datname  
----------
 postgres
 vault
(2 rows)

Apply: estrutura.ldif
adding new entry "ou=users,dc=prgs,dc=corp"

adding new entry "ou=groups,dc=prgs,dc=corp"

adding new entry "cn=paulo.devops,ou=users,dc=prgs,dc=corp"

adding new entry "cn=alvaro.dbre,ou=users,dc=prgs,dc=corp"

adding new entry "cn=benicio.developer,ou=users,dc=prgs,dc=corp"

adding new entry "cn=prgs/sandbox/hello-world/app-java-rw,ou=groups,dc=prgs,dc=corp"

adding new entry "cn=prgs/production/hello-world/app-java-rw,ou=groups,dc=prgs,dc=corp"

adding new entry "cn=prgs/production/hello-world/app-java-ro,ou=groups,dc=prgs,dc=corp"

adding new entry "cn=ggadmin,ou=groups,dc=prgs,dc=corp"

----------
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={1}mdb,cn=config"


Login
cn=paulo.devops,ou=users,dc=prgs,dc=corp
```

##### Provisionar o vault

```bash
cd 02-vault
sh deploy.sh
```

* Instala o Vault 
* Incializa o Vault e faz o Unseal 
* Configura o plugin do LDAP para autenticação externa
* Cria os paths
* Cria as policies


```bash
service/docker created
"hashicorp" already exists with the same configuration, skipping
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "metallb" chart repository
...Successfully got an update from the "metrics-server" chart repository
...Successfully got an update from the "hello" chart repository
...Successfully got an update from the "jetstack" chart repository
...Successfully got an update from the "kong-mesh" chart repository
...Successfully got an update from the "hashicorp" chart repository
...Successfully got an update from the "argo" chart repository
...Successfully got an update from the "gloo" chart repository
...Successfully got an update from the "external-secrets" chart repository
...Successfully got an update from the "gitlab" chart repository
Update Complete. ⎈Happy Helming!⎈
I0108 09:24:26.287915 3468493 warnings.go:110] "Warning: spec.SessionAffinity is ignored for headless services"
NAME: vault
LAST DEPLOYED: Thu Jan  8 09:24:25 2026
NAMESPACE: vault
STATUS: deployed
REVISION: 1
NOTES:
Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using
Vault with Kubernetes available here:

https://developer.hashicorp.com/vault/docs


Your release is named vault. To learn more about the release, try:

  $ helm status vault
  $ helm get manifest vault
Waiting Pod Health...
Waiting Pod Health...
Waiting Pod Health...
Waiting Pod Health...
Waiting Pod Health...
Waiting Pod Health...
Waiting Pod Health...
Waiting Pod Health...
Pods Running
secret/https-tls-secret created
gateway.gateway.networking.k8s.io/gateway created
httproute.gateway.networking.k8s.io/httproute-backend created
httproute.gateway.networking.k8s.io/httproute-redirect created
Waiting Pod Health...
Waiting Pod Health...
Pods Running
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    1/3
Unseal Nonce       ea8678cf-b626-2aed-c854-dc214ebd3ea9
Version            1.20.4
Build Date         2025-09-23T13:22:38Z
Storage Type       postgresql
HA Enabled         false
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    2/3
Unseal Nonce       ea8678cf-b626-2aed-c854-dc214ebd3ea9
Version            1.20.4
Build Date         2025-09-23T13:22:38Z
Storage Type       postgresql
HA Enabled         false
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    5
Threshold       3
Version         1.20.4
Build Date      2025-09-23T13:22:38Z
Storage Type    postgresql
Cluster Name    vault-cluster-7ed3de58
Cluster ID      6007cfa6-18ac-8784-70f7-f9f80ddfaa01
HA Enabled      false
Vault Selado:   false
Vault Token :   ******** 
WARNING! The VAULT_TOKEN environment variable is set! The value of this
variable will take precedence; if this is unwanted please unset VAULT_TOKEN or
update its value accordingly.

Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                ddddddddddddddddddddddddd
token_accessor       2NDiQ0Co0iHPV8r1XGtUfRIe
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
Success! Enabled ldap auth method at: ldap/
Success! Data written to: auth/ldap/config
WARNING! The VAULT_TOKEN environment variable is set! The value of this
variable will take precedence; if this is unwanted please unset VAULT_TOKEN or
update its value accordingly.

Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                ddddddddddddddddddddddddd
token_accessor       2NDiQ0Co0iHPV8r1XGtUfRIe
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
Success! Enabled the kv secrets engine at: prgs/
=================== Secret Path ===================
prgs/data/production/hello-world/app-java/postgres

======= Metadata =======
Key                Value
---                -----
created_time       2026-01-08T12:26:11.140660239Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
================= Secret Path =================
prgs/data/production/hello-world/app-java/kafka

======= Metadata =======
Key                Value
---                -----
created_time       2026-01-08T12:26:11.264082816Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
================= Secret Path =================
prgs/data/sandbox/hello-world/app-java/postgres

======= Metadata =======
Key                Value
---                -----
created_time       2026-01-08T12:26:11.385012066Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
================ Secret Path ================
prgs/data/sandbox/hello-world/app-java/kafka

======= Metadata =======
Key                Value
---                -----
created_time       2026-01-08T12:26:11.502876106Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
WARNING! The VAULT_TOKEN environment variable is set! The value of this
variable will take precedence; if this is unwanted please unset VAULT_TOKEN or
update its value accordingly.

Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                cccccccccccccccccccccccc
token_accessor       2NDiQ0Co0iHPV8r1XGtUfRIe
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
Success! Uploaded policy: prgs_production_hello_world_app_java_ro
Success! Data written to: auth/ldap/groups/prgs/production/hello-world/app-java-ro
WARNING! The VAULT_TOKEN environment variable is set! The value of this
variable will take precedence; if this is unwanted please unset VAULT_TOKEN or
update its value accordingly.

Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                dddddddddddddddddddddddd
token_accessor       2NDiQ0Co0iHPV8r1XGtUfRIe
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
Success! Uploaded policy: prgs_production_hello_world_app_java_rw
Success! Data written to: auth/ldap/groups/prgs/production/hello-world/app-java-rw
WARNING! The VAULT_TOKEN environment variable is set! The value of this
variable will take precedence; if this is unwanted please unset VAULT_TOKEN or
update its value accordingly.

Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                dddddddddddddddddddddddd
token_accessor       2NDiQ0Co0iHPV8r1XGtUfRIe
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
Success! Uploaded policy: prgs_sandbox_hello_world_app_java_rw
Success! Data written to: auth/ldap/groups/prgs/sandbox/hello-world/app-java-rw
```

##### Provisionar o external secrets

```bash
cd 03-external-secret
sh deploy.sh
```

* Cria Token Estático 
* Testa Token Estático
* Instala o Operator external Secrets
* Cria o secret store - backend vault 
* Cria o external secret

```bash
WARNING! The VAULT_TOKEN environment variable is set! The value of this
variable will take precedence; if this is unwanted please unset VAULT_TOKEN or
update its value accordingly.

Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                dddddddddddddddddddddddd
token_accessor       2NDiQ0Co0iHPV8r1XGtUfRIe
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
Success! Uploaded policy: prgs-read
WARNING! The VAULT_TOKEN environment variable is set! The value of this
variable will take precedence; if this is unwanted please unset VAULT_TOKEN or
update its value accordingly.

Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                gggggggggggggggggggggggggggggggggggggggggggggggggggggggggggggggggggggggggggggggggggg
token_accessor       oIqMlYWJrG0ZeuKbLyvp9teF
token_duration       8760h
token_renewable      true
token_policies       ["default" "prgs-read"]
identity_policies    []
policies             ["default" "prgs-read"]
Keys
----
kafka
postgres
=================== Secret Path ===================
prgs/data/production/hello-world/app-java/postgres

======= Metadata =======
Key                Value
---                -----
created_time       2026-01-08T12:26:11.140660239Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

==== Data ====
Key     Value
---     -----
pass    postgres-prod
user    postgres
"external-secrets" already exists with the same configuration, skipping
I0108 09:29:59.999838 3473589 warnings.go:110] "Warning: unrecognized format \"int32\""
I0108 09:30:00.012948 3473589 warnings.go:110] "Warning: unrecognized format \"int64\""
I0108 09:30:00.013028 3473589 warnings.go:110] "Warning: unrecognized format \"int32\""
I0108 09:30:00.035082 3473589 warnings.go:110] "Warning: unrecognized format \"int32\""
I0108 09:30:00.035277 3473589 warnings.go:110] "Warning: unrecognized format \"int64\""
I0108 09:30:00.128090 3473589 warnings.go:110] "Warning: unrecognized format \"int32\""
I0108 09:30:00.128129 3473589 warnings.go:110] "Warning: unrecognized format \"int64\""
I0108 09:30:00.147074 3473589 warnings.go:110] "Warning: unrecognized format \"int32\""
I0108 09:30:00.147116 3473589 warnings.go:110] "Warning: unrecognized format \"int64\""
NAME: external-secrets
LAST DEPLOYED: Thu Jan  8 09:29:58 2026
NAMESPACE: external-secrets
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
external-secrets has been deployed successfully in namespace external-secrets!

In order to begin using ExternalSecrets, you will need to set up a SecretStore
or ClusterSecretStore resource (for example, by creating a 'vault' SecretStore).

More information on the different types of SecretStores and how to configure them
can be found in our Github: https://github.com/external-secrets/external-secrets
Waiting Pod Health...
Waiting Pod Health...
Waiting Pod Health...
Waiting Pod Health...
Pods Running
Aguarde Webhook heath...
namespace/production created
secret/vault-policy-token created
NAME                 TYPE     DATA   AGE
vault-policy-token   Opaque   1      1s
secretstore.external-secrets.io/vault-backend created
NAME            AGE   STATUS   CAPABILITIES   READY
vault-backend   0s    Valid    ReadWrite      True
externalsecret.external-secrets.io/external-secret-java-app-production created
NAME                                  STORETYPE     STORE           REFRESH INTERVAL   STATUS         READY
external-secret-java-app-production   SecretStore   vault-backend   10s                SecretSynced   True
NAME                                  TYPE     DATA   AGE
external-secret-java-app-production   Opaque   4      0s
vault-policy-token                    Opaque   1      1s
apiVersion: v1
data:
  kafka-pass: a2Fma2EtcHJvZA==
  kafka-user: a2Fma2E=
  postgres-pass: cG9zdGdyZXMtcHJvZA==
  postgres-user: cG9zdGdyZXM=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"external-secrets.io/v1","kind":"ExternalSecret","metadata":{"annotations":{},"name":"external-secret-java-app-production","namespace":"production"},"spec":{"data":[{"remoteRef":{"key":"production/hello-world/app-java/postgres","property":"user"},"secretKey":"postgres-user"},{"remoteRef":{"key":"production/hello-world/app-java/postgres","property":"pass"},"secretKey":"postgres-pass"},{"remoteRef":{"key":"production/hello-world/app-java/kafka","property":"user"},"secretKey":"kafka-user"},{"remoteRef":{"key":"production/hello-world/app-java/kafka","property":"pass"},"secretKey":"kafka-pass"}],"refreshInterval":"10s","secretStoreRef":{"kind":"SecretStore","name":"vault-backend"},"target":{"creationPolicy":"Owner","name":"external-secret-java-app-production"}}}
    reconcile.external-secrets.io/data-hash: 7c071b10a285d140b38b00c330a6dd5a3a32ded7f7a7230b70fa49dc
  creationTimestamp: "2026-01-08T12:30:52Z"
  labels:
    reconcile.external-secrets.io/created-by: 35990a2d31fdd455f9ade83ff39aa5ae53cf500b91a6af1f0f20f311
    reconcile.external-secrets.io/managed: "true"
  name: external-secret-java-app-production
  namespace: production
  ownerReferences:
  - apiVersion: external-secrets.io/v1
    blockOwnerDeletion: true
    controller: true
    kind: ExternalSecret
    name: external-secret-java-app-production
    uid: 0cf56bac-e7b2-4c49-9511-249bafde065e
  resourceVersion: "4682"
  uid: 260de3c5-80f1-44fb-aba2-635d66b185e2
type: Opaque
```


##### Provisionar ambiente de teste 

```bash
cd 04-deployment
sh apply.sh

namespace/production unchanged
deployment.apps/hello created
```

```bash
sh teste.sh

Waiting Pod Health...
Pods Running

KAFKA_DB_PASS=kafka-prod
KAFKA_DB_USER=kafka
POSTGRESQL_DB_PASS=postgres-prod
POSTGRESQL_DB_USER=postgres
```

###### ALTERE NO UI DO VAULT AS CREDENCIAS 

![alt text](../../../../images/kubernetes/external-secrets/vault-change-password.png "Vaut Credentials")

```bash
sh kill.sh && sh teste.sh

matando o Pod para recarregar novas variaveis...

pod "hello-7b4c4754f6-jjb2b" deleted from production namespace

Waiting Pod Health...
Pods Running

KAFKA_DB_PASS=kafka-prod
KAFKA_DB_USER=kafka
POSTGRESQL_DB_PASS=123456
POSTGRESQL_DB_USER=postgres
```

#### 4) Youtube Demo

Você pode acompanhar todo esse processo de implementação no link abaixo.

<a href="https://youtu.be/xSY3V_JFGaw" target="_blank">Demo</a>
