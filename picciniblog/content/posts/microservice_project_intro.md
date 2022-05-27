---
author:
  name: "Patrick Piccini"
date: 2022-05-26T9:00:50-03:00
linktitle: Um projeto completo de Microservices
type:
- post
- posts
title: Um projeto completo de Microservices
weight: 10
---

## Table of Contents

---

A um tempo desejava entender melhor como os tão famosos microsserviços conseguem trabalhar individualmente porem todos conectados. Foi então que decidi projetar uma pequena aplicação onde iria me aprofundar nesses conhecimentos, e que também iria me desafia a criar uma aplicação completa seguindo o **Software Development Life Circle**. Que parte desde a criação da ideia, até o deploy.

Isso pode ser representado da seguinte fluxo:

![img1](/images/microservice_project/img1.jpg)

Então, meu nome é Patrick Berlatto Piccini, e esse é meu projeto completo de microservices.

## Entendimento do problema

Nesse projeto, utilizarei algumas ferramentas contidas na OCI (Oracle Claud Infraestructure) visto que, recentemente (Fevereiro 2022) passei na certificação &quot;Oracle Cloud Infrastructure Foundations 2021 Associate&quot;. Se você desejar seguir os passos da criação da aplicação, e desenvolver junto comigo o código, é opcional utilizar o OCI.

Vamos começar...

### Objetivo

Criar duas aplicações básicas de microsserviços:

O primeiro deles deverá ser um cadastro de usuários, contendo as seguintes informações:

- create\_user, show\_all\_user, show\_one\_user, edit\_user, edit\_password e delete\_user

Tabela de usuários &quot;users&quot; deverá conter os campos: user\_id, nick\_name, full\_name, password, cpf, email, phone\_number, created\_at, updated\_at.

O segundo será um serviço de OS (ordem de serviço) que deverá conter no cadastro, o user\_id do usuário contido no banco de dados. Deverá ter as seguintes informações:

- order\_id, user\_id, item\_description, item\_quantity, item\_price, total\_value, created\_at, updated\_at.

A arquitetura da aplicação será a seguinte: criaremos uma API que será responsável por distribuir as requisições através de um broker de mensagens chamado RabbitMQ, e também criar as filas e tabelas necessárias para a aplicação.

Nesse broker irá ter duas filas onde a API fará a separação das mensagens e enviará ao seu devido destino, onde teremos dois microservices, uma para _usuários_, e outro para os _orders_. Cada microsserviço é conectado a um banco de dados Postgres onde será armazenado as informações dos _usuários_ e dos _orders._

Juntamente a API, haverá uma camada de memória cache onde utilizaremos o Redis para fazer essa função. Então, caso uma requisição já tenha sido feita, a API irá verificar antes no dados em Cache se já existe essa informação, assim o usuário terá o retorno muito mais rápido.

![img2](/images/microservice_project/img2.jpg)

## Arquitetura do Projeto Completo

~~~ Estrutura
MS-application
│   .gitignore
│   docker-compose-services.yml
│
├───API
│   │   docker-compose-api.yml
│   │   Dockerfile
│   │   requirements.txt
│   │   server.py
│   │
│   ├───config
│   │       database_connection.py
│   │       rabbitmq_connection.py
│   │       redis_connection.py
│   │       __init__.py
│   │
│   └───rabbitmq_controller
│           rabbit_queues.py
│           __init__.py
│
├───MS1
│   │   docker-compose-microservice1.yml
│   │   Dockerfile
│   │   main.py
│   │   requirements.txt
│   │
│   ├───config
│   │       database_connection.py
│   │       rabbitmq_connection.py
│   │       __init__.py
│   │
│   ├───criptografy
│   │       hash_password.py
│   │       __init__.py
│   │
│   ├───database_controller
│   │       postgres_worker.py
│   │       __init__.py
│   │
│   └───rabbitmq_controller
│           rabbit_worker.py
│			__init__.py
│
└───MS2
    │   docker-compose-microservice2.yml
    │   Dockerfile
    │   main.py
    │   requirements.txt
    │
    ├───config
    │       database_connection.py
    │       rabbitmq_connection.py
    │       __init__.py
    │
    ├───database_controller
    │       postgres_worker.py
    │       __init__.py
    │
    └───rabbitmq_controller
            rabbit_worker.py
            __init__.py


~~~