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

Ex: 

Iniciar os containers docker

```bash
cd 01-docker-compose
sh restart.sh
```

Provisionar o vault

```bash
cd 02-vault
sh deploy.sh
```

Provisionar o external scripts

```bash
cd 03-external-secret
sh deploy.sh
```

Provisionar ambiente de teste 

```bash
cd 04-deployment
sh apply.sh
```

#### 4) Youtube Demo

Você pode acompanhar todo esse processo de implementação no link abaixo.


<a href="https://youtu.be/xSY3V_JFGaw" target="_blank">Demo</a>
