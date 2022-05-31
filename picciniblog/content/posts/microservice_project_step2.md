---
author:
  name: "Patrick Piccini"
date: 2022-05-26T9:01:52-03:00
linktitle: Microservice Project – Step 2
type:
- post
- posts
title: Microservice Project – Step 2
weight: 10
---
## Table of Contents
- [Entrar na Instancia via shell](#entrar-na-instancia-via-shell)
- [Instalar Docker](#instalar-docker)
- [Instalar Docker-Compose](#instalar-docker-compose)
- [Instalar Git](#instalar-git)
- [Resumo](#resumo)
---

Após Configurar a infraestrutura no Oracle Cloud, necessitamos fazer instalações de alguns programas que iremos utilizar. O Docker para a criação dos nossos containers de servidores e microsserviços, o Docker-Compose para facilitar a criação dos containers, e o Git, para versionamento de código.

Como desenvolvi os códigos fora de nossa VM da OCI, o Git é o fator principal para que todos os códigos sejam disponíveis facilmente.

## Entrar na Instancia via shell

Logado no usuário **devel\_user** , iniciei o cloud shell para conectar-se ao VM criada através do SSH Connection.

\* Detalhe, essa conexão pode ser feita de qualquer terminal ou computador que tenha acesso ao SSH Connection, desde que se tenha cadastrado na hora da criação da VM a SSH-KEY da máquina que onde se conectará.

Com o comando **ssh opc@\<ip\_public\>** entrei na máquina e comecei a fazer a instalação com o pacote de instalação Dandified YUM (DNF) no Oracle Linux 8.

## Instalar Docker

Com a sequência de comando abaixo, foi instalado o Docker na VM:

~~~ shell
dnf install -y dnf-utils zip unzip
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
dnf remove -y runc
dnf install -y docker-ce --nobest
systemctl enable docker.service
systemctl start docker.service
systemctl status docker.service
docker version
~~~

Adicionar o usuário ao grupo docker para poder executar comandos Docker;

~~~ shell
sudo usermod -aG docker opc
newgrp docker  
~~~

Vemos que Docker está rodando e pronto para ser usando.

![img14](/images/microservice_project/img14.jpg)

## Instalar Docker-Compose

Foi instalado o Docker-Compose na VM com os seguintes comandos:

~~~ shell
sudo dnf -y install curl
curl -s https://api.github.com/repos/docker/compose/releases/latest|grep browser_download_url|grep docker-compose-linux-x86_64|cut -d '"' -f 4|wget -qi –
ls -1 docker-compose-linux-x86_64*
sha256sum -c docker-compose-linux-x86_64.sha256
chmod +x docker-compose-linux-x86_64
sudo mv docker-compose-linux-x86_64 /usr/local/bin/docker-compose
docker-compose version
~~~

Docker-Compose instalado com sucesso.

![img15](/images/microservice_project/img15.jpg)

## Instalar Git

Comandos para instalação do Git:

~~~ shell
yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel -y
yum install gcc perl-ExtUtils-MakeMaker -y
cd /usr/local/
wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.35.1.tar.gz
tar zxvf git-2.35.1.tar.gz
cd git-2.35.1/
make prefix=/usr/local/git all
make prefix=/usr/local/git install
echo "export PATH=$PATH:/usr/local/git/bin" >> /etc/bashrc
source /etc/bashrc
~~~

## Resumo

Nesse Step, nos conectamos a instância da OCI e instalamos algumas dependências para o nosso ambiente de produção. Instalamos o Docker, Docker-Compose e Git.