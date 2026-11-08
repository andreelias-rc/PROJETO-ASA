# Projeto 01 - DevOps com Vagrant e Ansible

**Instituição:** Instituto Federal da Paraíba - Campus João Pessoa  
**Disciplina:** Administração de Sistemas Abertos  
**Professor:** Leonidas Lima  
**Período:** 2026.2  
**Integrante:** André Luís Henrique Elias Monteiro  
**Matrícula:** 20251380024

## 1. Objetivo

Provisionar e configurar uma infraestrutura virtual com Vagrant e Ansible, aplicando conceitos de Infraestrutura como Código (IaC) na administração de sistemas Debian 12. O ambiente é composto por quatro servidores virtuais com funções específicas de rede, armazenamento, aplicação e acesso cliente.

## 2. Infraestrutura

O arquivo `Vagrantfile` define as quatro máquinas virtuais utilizando o provider VirtualBox e a box `debian/bookworm64`. A infraestrutura utiliza clonagem vinculada (`linked_clone`), desativa a substituição das chaves SSH do Vagrant e a verificação automática de Guest Additions. As máquinas `arq`, `db` e `app` possuem 512 MB de RAM; a máquina `cli` possui 1024 MB.

| Máquina | Nome de host | IP da rede privada | Função |
|---|---|---|---|
| arq | arq.andre.devops | 192.168.56.124 | DHCP, DNS, NFS e LVM |
| db | db.andre.devops | 192.168.56.125, por reserva DHCP | MariaDB e autofs |
| app | app.andre.devops | 192.168.56.126, por reserva DHCP | Apache2 e autofs |
| cli | cli.andre.devops | DHCP, faixa 192.168.56.50-100 | Firefox ESR, SSH/X11 e autofs |

A rede privada utiliza o segmento `192.168.56.0/24`. Os endereços `.125` e `.126` foram definidos para a realização individual do projeto, pois a fórmula do enunciado original utiliza as matrículas de dois integrantes.

## 3. Funcionamento da infraestrutura

### 3.1. Provisionamento

O Vagrant gerencia a criação, a inicialização e a configuração de rede das máquinas virtuais. O Ansible executa um playbook de configurações comuns em cada máquina, seguido do playbook correspondente à função do servidor. Um gatilho do Vagrant desabilita o DHCP do próprio VirtualBox na rede privada, evitando concorrência com o DHCP configurado no servidor `arq`.

A inicialização lógica ocorre na ordem `arq`, `db`, `app` e `cli`. O servidor `arq` disponibiliza DHCP e DNS antes de os outros equipamentos obterem endereços na rede privada.

### 3.2. Configurações comuns

As quatro máquinas recebem atualizações do sistema operacional, cliente NFS, servidor SSH e Chrony. O Chrony utiliza o servidor `pool.ntp.br`, e o fuso horário é definido como `America/Recife`.

O grupo `ifpb` e o usuário `andre` são criados em cada máquina. O grupo `ifpb` recebe permissão de administração por `sudo`. O acesso SSH utiliza chaves públicas, impede autenticação por senha, bloqueia login de `root` e restringe o acesso aos grupos `vagrant` e `ifpb`. Antes da autenticação é apresentada a mensagem: `Acesso apenas para pessoas com autorização expressa!!!`.

### 3.3. Servidor de arquivos - arq

O servidor `arq` utiliza três discos adicionais de 10 GB no LVM. Esses discos compõem o Volume Group `dados`, que contém o Logical Volume `ifpb` de 15 GB, formatado em ext4 e montado automaticamente no diretório `/dados`.

O serviço NFS exporta `/dados/nfs` para a rede `192.168.56.0/24`. O usuário `nfs-ifpb`, sem acesso a shell, é proprietário do diretório. O compartilhamento utiliza `all_squash` para mapear usuários remotos para esse usuário e `sync` para efetuar gravações síncronas.

O serviço ISC DHCP é autoritativo para a rede privada, oferece endereços na faixa de `192.168.56.50` a `192.168.56.100`, utiliza gateway `192.168.56.1`, DNS `192.168.56.124`, lease padrão de 180 segundos e lease máximo de 3600 segundos. Os endereços dos servidores `db` e `app` são vinculados aos respectivos endereços MAC.

O Bind9 mantém a zona direta `andre.devops` e a zona reversa `56.168.192.in-addr.arpa`, com registros de `arq`, `db` e `app`. O serviço atende à rede interna e encaminha consultas externas aos resolvers `1.1.1.1` e `8.8.8.8`.

### 3.4. Servidor de banco de dados - db

O servidor `db` executa o MariaDB. O serviço `autofs` disponibiliza automaticamente o compartilhamento NFS do servidor `arq` no diretório `/var/nfs`.

### 3.5. Servidor de aplicação - app

O servidor `app` executa o Apache2. A página inicial apresenta a descrição do projeto, o nome do integrante e a matrícula. O `autofs` também disponibiliza o compartilhamento do servidor `arq` em `/var/nfs`.

### 3.6. Máquina cliente - cli

A máquina `cli` recebe seu endereço por DHCP e possui os pacotes `firefox-esr` e `xauth`. Seu servidor SSH permite encaminhamento X11. O compartilhamento NFS do servidor `arq` é disponibilizado automaticamente em `/var/nfs` por meio do `autofs`.

## 4. Arquivos do projeto

| Arquivo | Finalidade |
|---|---|
| `Vagrantfile` | Definição das quatro máquinas virtuais, discos, rede e provisionamento |
| `ansible.cfg` | Configuração do Ansible |
| `hosts.ini` | Inventário dos servidores |
| `playbooks/conf-comum.yml` | Configurações de usuários, SSH, NTP e NFS cliente |
| `playbooks/conf-arq.yml` | Configuração de DHCP, DNS, LVM e NFS |
| `playbooks/conf-db.yml` | Configuração do MariaDB e autofs |
| `playbooks/conf-app.yml` | Configuração do Apache2 e autofs |
| `playbooks/conf-cli.yml` | Configuração do cliente e autofs |
| `playbooks/files/` | Arquivos usados pelos serviços e pela página web |

## 5. Integração e verificação dos serviços

O provisionamento configura os serviços com inicialização automática. A integração da rede envolve a concessão de endereços pelo DHCP, a resolução de nomes pelo Bind9 e a comunicação entre as máquinas pelos nomes de host. O volume LVM fornece o armazenamento exportado pelo NFS, enquanto o `autofs` disponibiliza esse recurso aos servidores `db`, `app` e ao cliente `cli`. O Apache2 fornece a página de identificação do projeto e o MariaDB mantém o serviço de banco de dados.

A verificação funcional contempla o estado das quatro máquinas, os serviços SSH, Chrony, ISC DHCP, Bind9, NFS, Apache2, MariaDB e autofs, os registros DNS diretos e reversos, os volumes LVM, as concessões DHCP e o acesso ao compartilhamento NFS. O acesso à aplicação web pode ser realizado pelo nome `app.andre.devops` quando a resolução DNS estiver configurada no equipamento de origem.
