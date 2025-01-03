---
layout: post
collection: aws
permalink: /aws/pfsense-routing-subnets
title:  "AWS Pfsense Router Subnets"
author: Paulo Rogério
date:   2024-11-28 20:50:00 -0300
categories: [aws]
published: true
---

# Usando Pfsense como Concentrador de Redes na AWS

- [1) Porque usar Pfsense como concentrador?](#1-porque-usar-pfsense-como-concentrador)
- [2) Qual cenário vamos abordar?](#2-qual-cenário-vamos-abordar)  
- [3) VPCs](#3-vpcs)
  - [3.1) Primeira VPC](#31-primeira-vpc)
    - [3.1.1) Criando VPC](#311-criando-vpc)
    - [3.1.2) Criando SubNet](#312-criando-subnet)
    - [3.1.3) Criando Internet Gateway](#313-criando-internet-gateway) 
    - [3.1.4) Tabela de Roteamento](#314-tabela-de-roteamento)   
  - [3.2) Segunda VPC](#32-segunda-vpc)
    - [3.2.1) Criando VPC](#321-criando-vpc)
    - [3.2.2) Criando SubNet](#322-criando-subnet)
    - [3.2.3) Criando Internet Gateway](#323-criando-internet-gateway) 
    - [3.2.4) Tabela de Roteamento](#324-tabela-de-roteamento)
- [4) EC2 PfSense ](#4-ec2-pfsense)
    - [4.1) Criando EC2](#41-criando-ec2)
    - [4.2) Conectando na Instância](#42-conectando-na-instância)
    - [4.3) Adicionando Usuário Admin](#43-adicionando-usuário-admin)
    - [4.4) Configurações Básicas](#44-configurações-básicas)
    - [4.5) Certificado Web Browser](#45-certificado-web-browser)
    - [4.6) Adicionando Interface de Rede](#46-adicionando-interface-de-rede)
    - [4.7) Desabilitando Firewall](#47-desabilitando-firewall)
    - [4.8) Adicionando Ip Estático na EC2](#48-adicionando-ip-estático-na-ec2)
- [5) OpenVPN](#5-openvpn)
    - [5.1) Criando uma CA](#51-criando-uma-ca)
    - [5.2) Criando um Certificado](#52-criando-um-certificado)
    - [5.3) Criando Certificado para Usuário](#53-criando-certificado-para-usuário)
    - [5.4) Instalação de Pacotes](#54-instalação-de-pacotes)
        - [5.4.1) FreeRadios](#541-freeradios)
        - [5.4.2) OpenVPN Exporter](#542-openvpn-exporter)
    - [5.5) FreeRadios Configurações](#55-freeradios-configurações)
        - [5.5.1) Interfaces](#551-interfaces)
        - [5.5.2) Nas Clients](#552-nas-clients)
        - [5.5.3) Usuário FreeRadios](#553-usuário-freeradios)    
    - [5.6) OpenVPN Wizard](#56-openvpn-wizard)
    - [5.7) Adicionando Usuário de VPN](#57-adicionando-usuário-de-vpn)
    - [5.8) VPN Connect](#58-vpn-connect)
- [6) IPSec](#6-ipsec)
    - [6.1) IPSec Wizard](#61-ipsec-wizard)
        - [6.1.1) Permissões AWS Necessárias](#611-permissões-aws-necessárias)
        - [6.1.2) Pacotes Pfsense Necessários](#612-pacotes-pfsense-necessários)
        - [6.1.3) Fechando Túnel](#613-fechando-túnel)
    - [6.2) IPSec Manualmente](#62-ipsec-manualmente)
        - [6.2.1) Configurando VPN na AWS](#621-configurando-vpn-na-aws)
        - [6.2.2) Fechando com Primeiro IP Público](#622-fechando-com-primeiro-ip-público)
            - [6.2.2.1) Fase 1](#6221-fase-1)
            - [6.2.2.2) Fase 2](#6222-fase-2)
            - [6.2.2.3) Fase 3](#6223-fase-3)            
        - [6.2.3) Fechando com Segundo IP Público](#623-fechando-com-segundo-ip-público)
            - [6.2.3.1) Fase 1](#6231-fase-1)
            - [6.2.3.2) Fase 2](#6232-fase-2)
            - [6.2.3.3) Fase 3](#6233-fase-3)            
    - [6.3) Fechando Conexão com Primeiro IP Público](#63-fechando-conexão-com-primeiro-ip-público)
    - [6.4) Fechando Conexão com Segundo IP Público](#64-fechando-conexão-com-segundo-ip-público)        

## 1) Porque usar Pfsense como concentrador?

A própria **AWS** forcesse um serviço de Túnel VPN entre as VPC, porém isso é cobrado. O objetivo é utilizar o **pfSense** para interligar as redes VPC.

## 2) Qual cenário vamos abordar?

Teremos **2 VPC** na mesma região, cada qual com sua respectiva faixa de IP. O desenho abaixo ilustrará melhor o papel do Pfsense.

![alt text](/images/aws-pfsense-router/diagrama-rede/1.png)

## 3) VPCs

### 3.1) Primeira VPC

Vamos criar nossa **Primeira VPC**. Observer que não temos nenhuma VPC associada nessa conta.

![alt text](/images/aws-pfsense-router/primeira-vpc/vpc/1.png)

#### 3.1.1) Criando VPC

Os passos abaixos irá criar uma **Nova VPC**, a faixa de Rede usada será: **10.170.0.0/16**

![alt text](/images/aws-pfsense-router/primeira-vpc/vpc/2.png)

![alt text](/images/aws-pfsense-router/primeira-vpc/vpc/3.png)

![alt text](/images/aws-pfsense-router/primeira-vpc/vpc/4.png)

Habilitando essa flag, fará com que as EC2 criadas nessa VPC já tenham os nomes das instâncias associadas ao DNS da AWS.

![alt text](/images/aws-pfsense-router/primeira-vpc/vpc/5.png)

#### 3.1.2) Criando SubNet

Localize o menu **Subnet**, e clique no menu **Create subnet**, conforme mostra na imagem abaixo.

![alt text](/images/aws-pfsense-router/primeira-vpc/subnet/1.png)

Escolha qual VPC irá associar a essa subnet.

![alt text](/images/aws-pfsense-router/primeira-vpc/subnet/2.png)

Um detalhe importante é que podemos criar várias subnets dentro de uma VPC, para esse material iremos criar uma Subnet na classe C, ou seja, a subnet terá o seguinte endereço de rede: **10.170.1.0/24**

![alt text](/images/aws-pfsense-router/primeira-vpc/subnet/3.png)

Para que uma **EC2** possa ter um serviço acessado **externamente**, ou seja, que outras pessoas possam acessar o conteúdo dessa **EC2** é necessário que ela tenha um **IP Público** vinculado a essa **EC2**, quando habilitado essa flag dentro da subnet, em outras palavras, vc está permitindo que essa **EC2** vinculada a essa Subnet possa ser acessada externamente por um **IP Publico**.

**Obs.:** Caso queira criar uma **Subnet apenas com acesso interno**, basta *NÃO* habilitar o recurso abaixo. 

![alt text](/images/aws-pfsense-router/primeira-vpc/subnet/4.png)

![alt text](/images/aws-pfsense-router/primeira-vpc/subnet/5.png)

#### 3.1.3) Criando Internet Gateway

No menu localize **Internet Gateway**, e clique em **Create internet gateway**.

![alt text](/images/aws-pfsense-router/primeira-vpc/internet-gateway/1.png)

![alt text](/images/aws-pfsense-router/primeira-vpc/internet-gateway/2.png)

![alt text](/images/aws-pfsense-router/primeira-vpc/internet-gateway/3.png)

Após criado deve-se attachar qual VPC estará vinculada a esse internet gateway.

![alt text](/images/aws-pfsense-router/primeira-vpc/internet-gateway/4.png)


#### 3.1.4) Tabela de Roteamento   

É aqui que vamos definir se a subnet pode acessar a Internet, devemos criar uma rota default apontando para o gateway de internet criado no passo anterior.

![alt text](/images/aws-pfsense-router/primeira-vpc/route-table/1.png)

![alt text](/images/aws-pfsense-router/primeira-vpc/route-table/2.png)

Adiconando a rota default para internet.

![alt text](/images/aws-pfsense-router/primeira-vpc/route-table/3.png)

![alt text](/images/aws-pfsense-router/primeira-vpc/route-table/4.png)

Por fim precisamos associar qual subnet essa configuração de rota irá aplicar-se.

![alt text](/images/aws-pfsense-router/primeira-vpc/route-table/5.png)


### 3.2) Segunda VPC

Vamos criar nossa **Segunda VPC**. 

#### 3.2.1) Criando VPC

Basicamente iremos realizar os mesmos passos feitos na criação da primeira VPC, será enfatizado apenas os aspectos específicos dessa segunda VPC.

Será criado uma VPC para atender o ambiente de desenvolvimento, cuja faixa de rede é: **10.160.0.0/16**

![alt text](/images/aws-pfsense-router/segunda-vpc/vpc/1.png)

![alt text](/images/aws-pfsense-router/segunda-vpc/vpc/2.png)

#### 3.2.2) Criando SubNet

Criar uma nova subnet e vincular e esssa recém criada **VPC => 10.160.0.0/16**

![alt text](/images/aws-pfsense-router/segunda-vpc/subnet/1.png)

A subnet que será criada terá o seguinte endereço associado: **10.160.1.0/24**

![alt text](/images/aws-pfsense-router/segunda-vpc/subnet/2.png)

#### 3.2.3) Criando Internet Gateway

Agora vamos criar um outro internet gateway e vincular a segunda VPC **10.160.0.0/24**

![alt text](/images/aws-pfsense-router/segunda-vpc/internet-gateway/1.png)

Depois de criado deve-se atachar qual **VPC** esse internet gateway irá atender.

![alt text](/images/aws-pfsense-router/segunda-vpc/internet-gateway/2.png)

![alt text](/images/aws-pfsense-router/segunda-vpc/internet-gateway/3.png)

#### 3.2.4) Tabela de Roteamento


Vamos definir se a subnet pode acessar a Internet criando uma rota default apontando para o gateway de internet criado no passo anterior.

![alt text](/images/aws-pfsense-router/segunda-vpc/route-table/1.png)

![alt text](/images/aws-pfsense-router/segunda-vpc/route-table/2.png)

Adiconando a rota default para internet.

![alt text](/images/aws-pfsense-router/segunda-vpc/route-table/3.png)

Por fim precisamos associar qual subnet essa configuração de rota irá aplicar-se.

![alt text](/images/aws-pfsense-router/segunda-vpc/route-table/4.png)

## 4) EC2 PfSense

### 4.1) Criando EC2

Nesse passo a passo vamos criar uma Instância EC2 com Pfsense.

![alt text](/images/aws-pfsense-router/ec2/pfsense/startup/1.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/startup/2.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/startup/3.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/startup/4.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/startup/5.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/startup/6.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/startup/7.png)

### 4.2) Conectando na Instância

Para descobrir a senha gerada pelo instalador, será necessário fazer um snapshot do boot do Sistema onde a senha é visualizada no browser, para realizar o primeiro acesso.

![alt text](/images/aws-pfsense-router/ec2/pfsense/conectando-pfsense/1.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/conectando-pfsense/2.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/conectando-pfsense/3.png)

### 4.3) Adicionando Usuário Admin

Após login, vamos dar inicio as configurações inicias. A primeira configuração a ser realizada, será a criação de um usuário com perfil **Admin**.

![alt text](/images/aws-pfsense-router/ec2/pfsense/user-admin/1.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/user-admin/2.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/user-admin/3.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/user-admin/4.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/user-admin/5.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/user-admin/6.png)

### 4.4) Configurações Básicas

Algumas configurções básicas são necessárias para garantir acesso externo do Pfsense para internet.

![alt text](/images/aws-pfsense-router/ec2/pfsense/configuracoes-basicas/1.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/configuracoes-basicas/2.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/configuracoes-basicas/3.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/configuracoes-basicas/4.png)

### 4.5) Certificado Web Browser

Se vc tem um certificado válido para um domínio, pode-se usá-lo no frontend do Pfsense.

![alt text](/images/aws-pfsense-router/ec2/pfsense/certificado-web/1.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/certificado-web/2.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/certificado-web/3.png)

### 4.6) Adicionando Interface de Rede

Nessa etapa , vamos adicionar uma segunda interface de rede. Essa interface funcionará como uma rede local. 

![alt text](/images/aws-pfsense-router/ec2/pfsense/interface-rede/1.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/interface-rede/2.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/interface-rede/3.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/interface-rede/4.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/interface-rede/5.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/interface-rede/6.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/interface-rede/7.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/interface-rede/8.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/interface-rede/9.png)

### 4.7) Desabilitando Firewall

Nesse momento, o objtivo é concentrar o acesso e não bloquear os acessos, por isso, vamos desabilitar o firewall.

![alt text](/images/aws-pfsense-router/ec2/pfsense/desabilitar-firewall/1.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/desabilitar-firewall/2.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/desabilitar-firewall/3.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/desabilitar-firewall/4.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/desabilitar-firewall/5.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/desabilitar-firewall/6.png)

### 4.8) Adicionando Ip Estático na EC2

Nessa etapa vamos adicionar um **IP Fixo** para facilitar os acessos via VPN, para isso desligue a VM e localize o menu, **Elastic IPS**

![alt text](/images/aws-pfsense-router/ec2/pfsense/ip-elastic/1.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/ip-elastic/2.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/ip-elastic/3.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/ip-elastic/4.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/ip-elastic/5.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/ip-elastic/6.png)


## 5) OpenVPN

### 5.1) Criando uma CA

Nessa etapa vamos configurar o OpenVPN com autenticação baseada em FreeRadios + Google Autentication. Para iniciarmos os trabalhos, vamos criar primeiramente o Autoridade Certificadora auto assinado.

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/ca/1.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/ca/2.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/ca/3.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/ca/4.png)

### 5.2) Criando um Certificado

Agora vamos criar um certificado para nosso serviço.

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/certificado/1.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/certificado/2.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/certificado/3.png)

### 5.3) Criando Certificado para Usuário

Cada usuário que precisar conectar-se em nossa VPN precisará de um certificado assinado pela certificadora **OpenVPN_CA**, gerada nos passos anteriores.

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/certificado-usuario/1.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/certificado-usuario/2.png)

### 5.4) Instalação de Pacotes

Instalar os pacotes abaixo para exportar as configurações do usuário.

#### 5.4.1) FreeRadios

Instalar FreeRadius

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/packages/5.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/packages/6.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/packages/7.png)

#### 5.4.2) OpenVPN Exporter

Instalar OpenVPN Exporter

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/packages/1.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/packages/2.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/packages/3.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/packages/4.png)

### 5.5) FreeRadios Configurações

#### 5.5.1) Interfaces

Qual interface o serviço FreeRadiu vai ficar listen?

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/freeradius/1.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/freeradius/2.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/freeradius/3.png)

#### 5.5.2) Nas Clients

Aqui vamos definir qual usuário irá conectar-se no FreeRadius para validar a autenticação do usuário que está requisitando o login.

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/nas-client/3.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/nas-client/4.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/nas-client/5.png)

#### 5.5.3) Usuário FreeRadios    

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/user-freeradius/6.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/user-freeradius/7.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/user-freeradius/8.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/user-freeradius/9.png)

### 5.6) OpenVPN Wizard

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/1.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/2.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/3.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/4.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/5.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/6.png)

Aqui você precisa definir qual é a faixa de Rede da VPN.

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/7.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/8.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/9.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/10.png)

Depois de Configurado o Wizard, vamos ajustar alguns detalhes.

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/11.png)

Mude o autenticador para usar a base do FreeRadius.

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/12.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/13.png)

Aqui é um detalhe insteressante, pode-se definir rotas dinamicamentes e até mesmo definir os logs do serviço.

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/14.png)

Entre com o IP Fixo da EC2.

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/15.png)

Exportando a configuração do Usuário.

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/16.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/17.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/18.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/19.png)

Liberar no security Group da AWS a porta **1194** protocolo **UDP**.

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/21.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/22.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/openvpn-wizard/23.png)

### 5.7) Adicionando Usuário de VPN

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/usuario-vpn/1.png)

**DEIXE A SENHA EM BRANCO**

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/usuario-vpn/2.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/usuario-vpn/3.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/usuario-vpn/4.png)

Vamos validar se a **Senha + Autenticador** está funcionando bem. 

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/usuario-vpn/5.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/usuario-vpn/6.png)

### 5.8) VPN Connect

Estabelecendo Conexão

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/vpn-connect/24.png)

![alt text](/images/aws-pfsense-router/ec2/pfsense/openvpn/vpn-connect/25.png)

## 6) IPSec
### 6.1) IPSec Wizard

#### 6.1.1) Permissões AWS Necessárias

Nessa etapa vamos usar o próprio **PfSense**, para criar as configurações necessárias tanto na **AWS**, quanto no **PfSense**. Para isso é necessário que o **PfSense** tenha acesso a uma conta na AWS e atrelar as seguintes permissões.

![alt text](/images/aws-pfsense-router/ipsec/ipsec-wizard/permissoes-iam/1.png)
![alt text](/images/aws-pfsense-router/ipsec/ipsec-wizard/permissoes-iam/2.png)
![alt text](/images/aws-pfsense-router/ipsec/ipsec-wizard/permissoes-iam/3.png)
![alt text](/images/aws-pfsense-router/ipsec/ipsec-wizard/permissoes-iam/4.png)
![alt text](/images/aws-pfsense-router/ipsec/ipsec-wizard/permissoes-iam/5.png)
![alt text](/images/aws-pfsense-router/ipsec/ipsec-wizard/permissoes-iam/6.png)

#### 6.1.2) Pacotes Pfsense Necessários

Para que o **Pfesene**, possa manipular **AWS**, é necessário que tenha um pacote para que essa interação aconteça.

![alt text](/images/aws-pfsense-router/ipsec/ipsec-wizard/pacotes/1.png)

#### 6.1.3) Fechando Túnel

![alt text](/images/aws-pfsense-router/ipsec/ipsec-wizard/fechando-tunel/1.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-wizard/fechando-tunel/2.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-wizard/fechando-tunel/3.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-wizard/fechando-tunel/4.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-wizard/fechando-tunel/5.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-wizard/fechando-tunel/6.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-wizard/fechando-tunel/7.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-wizard/fechando-tunel/8.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-wizard/fechando-tunel/9.png)


### 6.2) IPSec Manualmente

#### 6.2.1) Configurando VPN na AWS

Nessa etapa vamos contruir o tunel manualmente, isso deixará claro o processo que é realizado pelo Wizard.

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/1.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/2.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/3.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/4.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/5.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/6.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/7.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/8.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/9.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/10.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/11.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/12.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/13.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/14.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/15.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/16.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/17.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/configurando-aws/18.png)

```yaml
! Amazon Web Services
! Virtual Private Cloud

! AWS utilizes unique identifiers to manipulate the configuration of 
! a VPN Connection. Each VPN Connection is assigned an identifier and is 
! associated with two other identifiers, namely the 
! Customer Gateway Identifier and Virtual Private Gateway Identifier.
!
! Your VPN Connection ID 		  : vpn-0207bc02e3bd667a4
! Your Virtual Private Gateway ID  : vgw-0e1a3b80deef026d6
! Your Customer Gateway ID		  : cgw-013eeff0568cf8b5c
!
!
! This configuration consists of two tunnels. Both tunnels must be 
! configured on your Customer Gateway for redundancy.
!
! --------------------------------------------------------------------------------
! IPSec Tunnel #1
! --------------------------------------------------------------------------------
! #1: Internet Key Exchange (IKE) Configuration
!
! A policy is established for the supported ISAKMP encryption, authentication, Diffie-Hellman, lifetime, 
! and key parameters.The IKE peer is configured with the supported IKE encryption,  authentication, Diffie-Hellman, lifetime, and key 
! parameters.Please note, these sample configurations are for the minimum requirement of AES128, SHA1, and DH Group 2.
! Category "VPN" connections in the GovCloud region have a minimum requirement of AES128, SHA2, and DH Group 14.
! You will need to modify these sample configuration files to take advantage of AES256, SHA256,  or other DH 
! groups like 2, 14-18, 22, 23, and 24.
! NOTE: If you customized tunnel options when creating or modifying your VPN connection, you may need to modify these sample configurations to match the custom settings for your tunnels.
!
! Higher parameters are only available for VPNs of category "VPN," and not for "VPN-Classic".
! The address of the external interface for your customer gateway must be a static address.
! Your customer gateway may reside behind a device performing network address translation (NAT). To 
! ensure that NAT traversal (NAT-T) can function, you must adjust your firewall 
! rules to unblock UDP port 4500. 
| If not behind NAT, and you are not using an Accelerated VPN, we recommend disabling NAT-T. If you are using an Accelerated VPN, make sure that NAT-T is enabled.
!
!
Go to VPN-->IPSec. Add a new Phase1 entry (click + button )

General information
 a. Disabled : uncheck
 b. Key Exchange version :V1
 c. Internet Protocol : IPv4
 d. Interface : WAN
 e. Remote Gateway: 52.67.200.218
 f. Description: Amazon-IKE-vpn-0207bc02e3bd667a4-0
 
 Phase 1 proposal (Authentication)
 a. Authentication Method: Mutual PSK
 b. Negotiation mode : Main
 c. My identifier : My IP address
 d. Peer identifier : Peer IP address
 e. Pre-Shared Key: YQPKqoO7jIMspTUWCUXl7FzvnbihbgIw
 
 Phase 1 proposal (Algorithms)
 a. Encryption algorithm : aes128 
 b. Hash algorithm :  sha1
 c. DH key group :  2
 d. Lifetime : 28800 seconds
 
 Advanced Options
 a. Disable Rekey : uncheck
 b. Responder Only : uncheck
 c. NAT Traversal : Auto
 d. Deed Peer Detection : Enable DPD
    Delay between requesting peer acknowledgement : 10 seconds
	Number of consecutive failures allowed before disconnect : 3 retries
	
	

! #2: IPSec Configuration
! 
! The IPSec transform set defines the encryption, authentication, and IPSec
! mode parameters.
! Category "VPN" connections in the GovCloud region have a minimum requirement of AES128, SHA2, and DH Group 14.
! Please note, you may use these additionally supported IPSec parameters for encryption like AES256 and other DH groups like 2, 5, 14-18, 22, 23, and 24.
! Higher parameters are only available for VPNs of category "VPN," and not for "VPN-Classic".

Expand the VPN configuration clicking in "+" and then create a new Phase2 entry as follows:

 a. Disabled :uncheck
 b. Mode : Tunnel
 c. Local Network : Type: LAN subnet
    Address :  ! Enter your local network CIDR in the Address tab 
 d. Remote Network : Type : Network 
    Address :  ! Enter your remote network CIDR in the Address tab
 e. Description : Amazon-IPSec-vpn-0207bc02e3bd667a4-0
 
 Phase 2 proposal (SA/Key Exchange)
 a. Protocol : ESP
 b. Encryption algorithms :aes128 
  c. Hash algorithms : hmac-sha1-96
  d. PFS key group :   2
e. Lifetime : 3600 seconds 

Advanced Options

Automatically ping host : ! Provide the IP address of an EC2 instance in VPC that will respond to ICMP.


! --------------------------------------------------------------------------------


! --------------------------------------------------------------------------------
! IPSec Tunnel #2
! --------------------------------------------------------------------------------
! #1: Internet Key Exchange (IKE) Configuration
!
! A policy is established for the supported ISAKMP encryption, authentication, Diffie-Hellman, lifetime, 
! and key parameters.The IKE peer is configured with the supported IKE encryption,  authentication, Diffie-Hellman, lifetime, and key 
! parameters.Please note, these sample configurations are for the minimum requirement of AES128, SHA1, and DH Group 2.
! Category "VPN" connections in the GovCloud region have a minimum requirement of AES128, SHA2, and DH Group 14.
! You will need to modify these sample configuration files to take advantage of AES256, SHA256,  or other DH 
! groups like 2, 14-18, 22, 23, and 24.
! NOTE: If you customized tunnel options when creating or modifying your VPN connection, you may need to modify these sample configurations to match the custom settings for your tunnels.
!
! Higher parameters are only available for VPNs of category "VPN," and not for "VPN-Classic".
! The address of the external interface for your customer gateway must be a static address.
! Your customer gateway may reside behind a device performing network address translation (NAT). To 
! ensure that NAT traversal (NAT-T) can function, you must adjust your firewall 
! rules to unblock UDP port 4500. 
| If not behind NAT, and you are not using an Accelerated VPN, we recommend disabling NAT-T. If you are using an Accelerated VPN, make sure that NAT-T is enabled.
!
!
Go to VPN-->IPSec. Add a new Phase1 entry (click + button )

General information
 a. Disabled : uncheck
 b. Key Exchange version :V1
 c. Internet Protocol : IPv4
 d. Interface : WAN
 e. Remote Gateway: 54.233.165.172
 f. Description: Amazon-IKE-vpn-0207bc02e3bd667a4-1
 
 Phase 1 proposal (Authentication)
 a. Authentication Method: Mutual PSK
 b. Negotiation mode : Main
 c. My identifier : My IP address
 d. Peer identifier : Peer IP address
 e. Pre-Shared Key: rZGpXdbGjFWMY6vyant6LqcxLycHx0rE
 
 Phase 1 proposal (Algorithms)
 a. Encryption algorithm : aes128 
 b. Hash algorithm :  sha1
 c. DH key group :  2
 d. Lifetime : 28800 seconds
 
 Advanced Options
 a. Disable Rekey : uncheck
 b. Responder Only : uncheck
 c. NAT Traversal : Auto
 d. Deed Peer Detection : Enable DPD
    Delay between requesting peer acknowledgement : 10 seconds
	Number of consecutive failures allowed before disconnect : 3 retries
	
	

! #2: IPSec Configuration
! 
! The IPSec transform set defines the encryption, authentication, and IPSec
! mode parameters.
! Category "VPN" connections in the GovCloud region have a minimum requirement of AES128, SHA2, and DH Group 14.
! Please note, you may use these additionally supported IPSec parameters for encryption like AES256 and other DH groups like 2, 5, 14-18, 22, 23, and 24.
! Higher parameters are only available for VPNs of category "VPN," and not for "VPN-Classic".

Expand the VPN configuration clicking in "+" and then create a new Phase2 entry as follows:

 a. Disabled :uncheck
 b. Mode : Tunnel
 c. Local Network : Type: LAN subnet
    Address :  ! Enter your local network CIDR in the Address tab 
 d. Remote Network : Type : Network 
    Address :  ! Enter your remote network CIDR in the Address tab
 e. Description : Amazon-IPSec-vpn-0207bc02e3bd667a4-1
 
 Phase 2 proposal (SA/Key Exchange)
 a. Protocol : ESP
 b. Encryption algorithms :aes128 
  c. Hash algorithms : hmac-sha1-96
  d. PFS key group :   2
e. Lifetime : 3600 seconds 

Advanced Options

Automatically ping host : ! Provide the IP address of an EC2 instance in VPC that will respond to ICMP.


! --------------------------------------------------------------------------------



! Additional Notes and Questions
!  - Amazon Virtual Private Cloud Getting Started Guide: 
!       http://docs.amazonwebservices.com/AmazonVPC/latest/GettingStartedGuide
!  - Amazon Virtual Private Cloud Network Administrator Guide: 
!       http://docs.amazonwebservices.com/AmazonVPC/latest/NetworkAdminGuide
!  - XSL Version: 2009-07-15-1119716
```

#### 6.2.2) Fechando com Primeiro IP Público

##### 6.2.2.1) Fase 1

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/primeiro-ip/fase-1/1.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/primeiro-ip/fase-1/2.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/primeiro-ip/fase-1/3.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/primeiro-ip/fase-1/4.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/primeiro-ip/fase-1/5.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/primeiro-ip/fase-1/6.png)

##### 6.2.2.2) Fase 2

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/primeiro-ip/fase-2/1.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/primeiro-ip/fase-2/2.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/primeiro-ip/fase-2/3.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/primeiro-ip/fase-2/4.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/primeiro-ip/fase-2/5.png)

##### 6.2.2.3) Fase 3

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/primeiro-ip/fase-3/1.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/primeiro-ip/fase-3/2.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/primeiro-ip/fase-3/3.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/primeiro-ip/fase-3/4.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/primeiro-ip/fase-3/5.png)

### 6.2.3) Fechando com Segundo IP Público

##### 6.2.3.1) Fase 1

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/segundo-ip/fase-1/1.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/segundo-ip/fase-1/2.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/segundo-ip/fase-1/3.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/segundo-ip/fase-1/4.png)

##### 6.2.3.2) Fase 2

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/segundo-ip/fase-2/1.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/segundo-ip/fase-2/2.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/segundo-ip/fase-2/3.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/segundo-ip/fase-2/4.png)

##### 6.2.3.3) Fase 3

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/segundo-ip/fase-3/1.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/segundo-ip/fase-3/2.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/segundo-ip/fase-3/3.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/segundo-ip/fase-3/4.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/segundo-ip/fase-3/5.png)

### 6.3) Fechando Conexão com Primeiro IP Público

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/conectando/primeiro-ip/1.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/conectando/primeiro-ip/2.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/conectando/primeiro-ip/3.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/conectando/primeiro-ip/4.png)

### 6.4) Fechando Conexão com Segundo IP Público        

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/conectando/segundo-ip/1.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/conectando/segundo-ip/2.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/conectando/segundo-ip/3.png)

![alt text](/images/aws-pfsense-router/ipsec/ipsec-manualmente/conectando/segundo-ip/4.png)
