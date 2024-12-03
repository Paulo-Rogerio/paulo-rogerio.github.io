---
layout: post
collection: linux
permalink: /linux/chef
title:  "Linux Chef Workstation"
author: Paulo Rogério
date:   2024-11-28 20:50:00 -0300
categories: [linux]
published: true
---

# 🚀 Devops Chef

- [1) Chef Server](#1-chef-server)
  - [1.1) Vms](#11-vms)
  - [1.3) Install](#13-install)
  - [1.3) Configure](#13-configure)
- [2) Chef WorkStation](#2-chef-workstation)
  - [2.1) Install](#21-install)
  - [2.2) Configure](#22-configure)
- [3) Chef Node](#2-chef-node)
  - [3.1) Install](#31-install)
- [4) Meu Primeiro CookBook](#4-meu-primeiro-cookbook)
- [5) Install and Configure PostgreSQL](#5-Install-and-configure-postgresql)


## 1) Chef Server
Nesse tutorial iremos aprender a utilizar essa poderosa ferramenta **Chef** para provisionamento de recursos dentro de uma máquina. O chef é baseado em Ruby, então todos os seus manifestos tem a extensão **.rb**. Nesse tutorial vamos criar todo o cenário usando VirtualBox.

### 1.1) Vms
Como dito no bloco anterior vamos usar VirtualBox para emular o cenário.

| Vm | S.O | Finalidade
| --- | --- | --- |
| Vbox | Ubuntu 20.04 | Chef Server
| Vbox | Ubuntu 20.04 | Chef Workstation
| Vbox | Ubuntu 20.04 | Chef Node - Nginx
| Vbox | Ubuntu 20.04 | Chef Node - PostgreSQL

Cada **Vm** possui 2 inerfaçe de Rede, sendo uma para comunicação interna **( Máquina Host com Máquina Virtual)** e outra interface para saída de internet. O lance é que ao adicionar uma máquina ***( Node para ser gerenciada pela Chef)***, ele usa a interface **NAT** como meio de comunicação entre os Hosts, e isso não é bom para nossos testes pois essa interface não consegue interagir com as outras máquinas.

Para resolver esse problema cada máquina conterá 2 interfaces sendo:
 - **NAT** ( Dedicada para acesso Externo )
 - **Internal** ( Dedicada para acesso Interno)

Outra característica é que a máquina **Chef Server** funcionará como gateway de internet para as demais máquinas, possam ser catalogadas no **Chef-Server** usando o IP da **Rede Interna**.

| Vm | Finalidade | Rede NAT | Rede Internal
| --- | --- | --- | --- |
| Vbox | Chef Server | 10.2.0.15 | 192.168.56.10
| Vbox | Chef WorkStation | 10.2.0.15 | 192.168.56.20
| Vbox | Chef Node - Nginx | 10.2.0.15 | 192.168.56.30
| Vbox | Chef Node - PostgreSQL | 10.2.0.15 | 192.168.56.40

#### Como adequar a rede das Vms?
Criar um script para rotear internet para os Hosts ( Vms ).

```bash
mkdir -p /opt/devops/firewall && \
touch /opt/devops/firewall/firewall.sh && \
chmod +x /opt/devops/firewall/firewall.sh

echo "echo 1 > /proc/sys/net/ipv4/ip_forward" > /opt/devops/firewall/firewall.sh
echo "iptables -A POSTROUTING -t nat -o enp0s3 -j MASQUERADE" >> /opt/devops/firewall/firewall.sh
```

Colocar o script para ser iniciado no boot do Sistema Operacional ( crontab ).

```bash
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
@reboot /opt/devops/firewall/firewall.sh
```

#### E nas estações de trabalho ( Nodes )?
Basta apontar o gateway padrão para a máquina que está funcionando como router.

```bash
vi /etc/netplan/00-installer-config.yaml
```

```yml
network:
  ethernets:
    enp0s3:
      dhcp4: true
      dhcp4-overrides:
       use-routes: false
    enp0s8:
      dhcp4: false
      addresses: [192.168.56.30/24]
      nameservers:
        addresses: [4.2.2.2]
      routes:
      - to: 0.0.0.0/0
        via: 192.168.56.10
  version: 2
```

```bash
route -vn
```

```bash
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.56.10   0.0.0.0         UG    0      0        0 enp0s8
10.0.2.0        0.0.0.0         255.255.255.0   U     0      0        0 enp0s3
192.168.56.0    0.0.0.0         255.255.255.0   U     0      0        0 enp0s8
```

### 1.2) Install
Com a infraestrutura pronta podemos iniciar os trabalhos. Vamos instalar o servidor **Chef Server**, faça download da ultima versão do pacote nesse Link.

* [Download Chef Server](https://www.chef.io/downloads/tools/infra-server)

Na escrita dessa doc a versão usada foi a ***chef-server-core_15.1.7-1_amd64.deb***

```bash
dpkg -i chef-server-core_15.1.7-1_amd64.deb
```

### 1.3) Configure
Os comandos abaixo pode demorar um pouco....

```bash
chef-server-ctl reconfigure
chef-server-ctl status
chef-server-ctl service-list
chef-server-ctl install chef-manage
chef-server-ctl reconfigure
chef-manage-ctl reconfigure
chef-manage-ctl reconfigure --accept-license
chef-server-ctl upgrade
```

#### Backup
```bash
chef-server-ctl backup
ls /var/opt/chef-backup
```

#### Reiniciando Aplicação
```bash
echo "vm.overcommit_memory = 1" >> /etc/sysctl.conf
echo "never" > /sys/kernel/mm/transparent_hugepage/enabled
chef-server-ctl restart
```

#### Logs
```bash
chef-server-ctl tail
```

#### Criando Usuário
Cada usuário com acesso a plataforma deverá ter uma chave, e por meio dessa que o desenvolvedor submete os cookbooks para dentro do server.

```bash
mkdir -p /root/.keys-chef
chef-server-ctl user-create paulo "Paulo Rogerio" "Chef Manager" email@gmail.com 123456 -f /root/.keys-chef/paulo.pem
```

#### Criando Organização
Essas organizações precisam ser vinculadas a algum usuário administrador. Observe que cada organização contém sua chave.

```bash
chef-server-ctl org-create production "Production" --association_user paulo -f /root/.keys-chef/production.pem
chef-server-ctl org-create development "Development" --association_user paulo -f /root/.keys-chef/development.pem
```

## 2) Chef WorkStation
Agora vamos configurar o ambiente de desenvolvimento, é aqui que o **Desenvolvedor** irá trabalhar.

### 2.1) Install
Vamos instalar o servidor **Chef Workstation**, faça download da ultima versão do pacote nesse Link.

* [Download Chef WorkStation](https://www.chef.io/downloads/tools/workstation)

Na escrita dessa doc a versão usada foi a ***chef-workstation_22.10.1013-1_amd64.deb***

```bash
dpkg -i chef-workstation_22.10.1013-1_amd64.deb
echo 'eval "$(chef shell-init bash)"' >> ~/.bashrc
chef -v
ruby -v
knife -v
which ruby
```

### 2.2) Configure
Navegue no servidor na seguinte estrutura, e faça o dowload do knife.rb. Esse arquivo contém os parâmetros necessários para submeter os uploads dos cookbooks.

![alt text](/images/linux/chef-workstation/01-workstation.jpg)

#### Knife
```bash
touch .chef/knife.rb
```

```bash
current_dir = File.dirname(__FILE__)
log_level                :info
log_location             STDOUT
node_name                "paulo"
client_key               "#{current_dir}/paulo.pem"
chef_server_url          "https://chef/organizations/estudos"
cookbook_path            ["~/projetos/chef-repo/cookbooks"]
```

#### Private Key
Nesse etapa você deverá copiar sua chave privada criada no **Chef Server**.

```bash
chmod 400 paulo.pem
```

#### Diretório onde ficará seus Projetos
```bash
mkdir -p ~/projetos
```

#### Testando comunicação com o Server
Ao executar o comando abaixo, você receberá um erro, pois o certificado não é valido, então é necessário baixar o certificado auto assinado gerado pelo **Chef** para que ele possa confiar.

```bash
knife client list (Erro SSL)
```

#### Download Certificados...
```bash
knife ssl fetch
```

#### List Users...
```bash
knife user list
```

## 3) Chef Node
Agora é hora de subir os agentes nas estações remotamente. O comando abaixo deve ser executado na maquina **WorkStation**.

### 3.1) Install
O comando abaixo deve ser executado a partir da máquina workstation.
ssh-user: Nome do usuário
ssh-password: Senha do usuário. Aqui tem um ponto de observação que é a utilização de chaves
públicas autorizadas no servidor de destino ( Authorized keys ) que também podem ser utilizadas.
node-name: é o rótulo, como vc deseja vizualizar o host.

```bash
knife bootstrap 192.168.56.30 --ssh-user root --ssh-password 123456 --node-name nginx
```
## 4) Meu Primeiro CookBook

```bash
mkdir -p ~/projetos/chef-repo/cookbooks
cd ~/projetos/chef-repo
chef generate cookbook cookbooks/base
```

```bash
cd cookbooks/base
ls recipes/
default.rb 
```

```bash
chef generate recipe timezone
```

### Definindo Timezone

```bash
touch recipes/timezone.rb
ls recipes/
default.rb  timezone.rb
```

```bash
vi recipes/timezone.rb 
```

```ruby
link "/etc/localtime" do
  to "/usr/share/zoneinfo/America/Sao_Paulo"
  not_if "readlink /etc/localtime | egrep -q '^Sao_Paulo$'"
end
```

### Includes

```bash
vi recipes/default.rb 
```

```ruby
include_recipe 'base::timezone'
```

### Rspec

```bash
cookstyle -a recipes/default.rb
Inspecting 1 file
.

1 file inspected, no offenses detected
```

```bash
cookstyle -a recipes/timezone.rb
Inspecting 1 file
.

1 file inspected, no offenses detected
```

### Testando CookBook

```ruby
chef-client --local-mode --why-run --runlist "recipe[base]"
```

### Testando Receitas de forma isolada

```ruby
chef-client --local-mode --why-run --runlist "recipe[base::timezone]"
```

### Upload Cookbook

```bash
cd ..
knife cookbook upload base
knife cookbook list

chef-client --local-mode --why-run --runlist "recipe[base::timezone]"

knife node run_list add nginx recipe[create-user::user]
knife node run_list add nginx recipe[create-user]
ssh root@192.168.56.30 chef-client --runlist "recipe["create-user"]"
```
