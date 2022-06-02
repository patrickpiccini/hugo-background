---
author:
  name: "Patrick Piccini"
date: 2022-06-02T13:00:52-02:00
linktitle: Microservice Project – Step 1
type:
- post
- posts
title: Microservice Project – Step 1
weight: 10
---

## Table of Contents
- [Criação do Compartment](#cria%C3%A7%C3%A3o-do-compartment)
- [Criação de Usuário](#cria%C3%A7%C3%A3o-de-usu%C3%A1rio)
- [Criação de Grupo](#cria%C3%A7%C3%A3o-de-grupo)
- [Criação de Policies](#cria%C3%A7%C3%A3o-de-policies)
- [Criação de SSH-KEY](#cria%C3%A7%C3%A3o-de-ssh-key)
- [Criação de Instancia VM](#cria%C3%A7%C3%A3o-de-instancia-vm)
- [Resumo](#resumo)
---


Nesse primeiro passo, iremos configurar um ambiente OCI para rodar toda a aplicação em cloud.

## Criação do Compartment

Iremos criar um _conpartiment._

No ícone de hamburguer na página inicial do OCI, iremos em > Identity > Security > compartments, e criaremos um compartimento com o nome **Developmet**.

![img3](/images/microservice_project/img3.jpg)

## Criação de Usuário

Após isso, criaremos um usuário de desenvolvimento para ter acesso somente ao compartimento **Developmet**.

Novamente no menu lateral, em > Identity > Security > users, iremos criar o usuário **devel\_user,** com o tipo IAM USER. Esses usuários podem acessar os serviços do Oracle Cloud Infrastructure, mas nem todos os serviços do Cloud Platform. Os usuários do IAM são cenários de usuário atípico, como acesso de administrador de emergência.

![img4](/images/microservice_project/img4.jpg)

![img5](/images/microservice_project/img5.jpg)

## Criação de Grupo

No menu lateral, em > Identity > Security > groups, criaremos um grupo com o nome de **Developer\_Group.** Depois disso, iremos adicionar o usuário **devel\_user** ao grupo clicando no botão "Add user to Group".

![img6](/images/microservice_project/img6.jpg)

## Criação de Policies

Finalizando o processo de criação de usuário, grupo, e adição do usuário ao grupo, é necessário atribuir uma sequência de políticas de permissões ao grupo, para que assim, o grupo criado tenha acesso ao compartimento _Developer_ para fazer as devidas interações.

Vamos em > Identity > Security > Policies

![img7](/images/microservice_project/img7.jpg)

Após ser direcionado a página de policies, selecionamos o compartimento **Developmet** para receber a política que criamos. Na policie, iremos permitir o Developer\_Group (e todos seus usuários) a utilização de todos os recursos no compartimento OCI.

![img8](/images/microservice_project/img8.jpg)

Também será criado uma outra policie no compartimento **root** , para liberar o acesso ao terminal cloud shell, através dos usuários do grupo _Developmet._ Isso deverá ser feito devido ao fato de que precisaremos de uma ssh-key disponibilizada pelo usuário para conseguir criar nossa instancia no OCI.

![img9](/images/microservice_project/img9.jpg)

## Criação de SSH-KEY

Logado no usuário _devel\_user,_ conectaremos a Oracle Cloud Shell para adquirir uma ssh\_key que posteriormente iremos utilizar. Para isso, vamos usar alguns simples comandos para a criação dessa chave.

![img10](/images/microservice_project/img10.jpg) 

~~~ cli
ssh-keygen – Para criação da chave

cat /home/your_user/.ssh/id_rsa.pub 
– Mostrará o conteúdo contido no arquivo id_rsa.pub,
  que foi criado com o comando anterior
~~~

![img11](/images/microservice_project/img11.jpg)

## Criação de Instancia VM

Ainda conectado ao usuário _devel\_user_ criaremos uma instância, que nada mais é que uma Virtual Machine. Em > compute > instances > create instance._
![img12](/images/microservice_project/img12.jpg)

Ao criar uma instância na página de configuração da VM, devemos colar a ssh-key adquirida anteriormente no campo onde solicita essa chave. Isso é feito para que se consiga acessar a VM remotamente. As imagens do Oracle Linux, CentOS ou Ubuntu usam esse par de chaves SSH ao contrário de uma senha para autenticar um usuário.

![img13](/images/microservice_project/img13.jpg)

OBS: A configuração da instância que irei utilizar, são disponibilizadas pelo serviço _Oracle Cloud – Free Tier,_ sendo ela 1 VM de computação baseadas em AMD com 1/8 OCPU\*\* e 1 GB de memória cada.

Para mais informações dos serviços Free Tier, acesse: [https://www.oracle.com/br/cloud/free/](https://www.oracle.com/br/cloud/free/)

## Resumo

Nesse Step, começamos as criar o ambiente onde rodaremos nossa aplicação. Configuramos um compartment, usuário e grupo de usuário, que será o responsável pelo acesso ao desenvolvimento na instância OCI. Definimos algumas políticas de usuário. Também criamos uma ssh-key que utilizamos para a criação da instancia OCI.