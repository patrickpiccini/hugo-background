---
author:
  name: "Patrick Piccini"
date: 2025-05-26T9:02:02-03:00
linktitle: Microservice Project – Step 12
type:
- post
- posts
title: Microservice Project – Step 12
weight: 10
---
![microservices](/images/microservice_project/microservices.png)
## Table of Contents
- [Instalação](#instala%C3%A7%C3%A3o)
- [Inicialização do Redis](#inicializa%C3%A7%C3%A3o-do-redis)
- [Decorator Cached](#decorator-cached)
- [Exemplo](#exemplo)
- [Microservice Project – Finalização](#microservice-project--finaliza%C3%A7%C3%A3o)

---

Findando o nosso projeto, precisaremos ainda aplicar a camada de cache em todas as rodas que nos retornam alguma informação vinda do banco de dados. Como estamos usando o Flask em nossa aplicação, existe uma extensão de cache próprio da biblioteca chamada Flask-Caching. Flask-Caching é uma extensão do Flask que adiciona suporte de cache para vários back-ends a qualquer aplicativo Flask. Ao ser executado em cima do [cachelib](https://github.com/pallets-eco/cachelib) , ele suporta todos os back-ends de cache originais do werkzeug por meio de uma API uniforme. Também é possível desenvolver seu próprio backend de armazenamento em cache por meio de subclasses de flask\_caching.backends.base.BaseCache classe, mas isso não vem ao caso nessa aplicação.

[Flask-Caching - [https://flask-caching.readthedocs.io/en/latest/](https://flask-caching.readthedocs.io/en/latest/)]

## Instalação

Antes de qualquer coisa, certifique-se de ter instalado todos os módulos contidos no arquivo _requirements.txt_ da API.

~~~ requirements
   psycopg2-binary
   Flask-Caching
   threaded
   flask
   redis
   pika
   uuid
~~~
Comando para instalação:
~~~ shell
pip install -r requeriments.txt
~~~

Primeiramente iremos fazer o import da biblioteca _Cache_ que iremos utilizar no arquivo _server.py_.

~~~ python
from flask import Flask, request
from flask_caching import Cache  # Import Cache from flask_caching module
from config.database_connection import ConnectionDatabase
...
~~~

## Inicialização do Redis

Dentro da classe _Api\_server_, apontaremos a conexão com o servidor Redis, visto que já configuramos todas as informações no arquivo _redis\_connection.py_ no Step5, precisaremos chama-las no arquivo da API.

~~~ python
    app = Flask(__name__)
    app.config.from_object('config.redis_connection.BaseConfig')
    cache = Cache(app)  # Initialize Cache

    ConnectionDatabase()
    rabbit_queues.create_queues()
~~~

## Decorator Cached

Nas rotas que serão retornadas alguma informação da banco de dados iremos colocar um novo decorator. Para armazenar funções em cache, usaremos o comando **cached**. Por parâmetro, haverá a opção _timeout_, que será o responsável por manter o cache por determinado tempo, que no nosso caso, será 30 segundos.

~~~ python
@app.route("/user/show_all_user/", methods=['GET'])
@cache.cached(timeout=30, query_string=True) ##CACHE
def list_user():
...

@app.route("/user/show_one_user/", methods=['POST'])
@cache.cached(timeout=30, query_string=True) ##CACHE
def show_user():
...

@app.route("/order/list_all_orders/", methods=['GET'])
@cache.cached(timeout=30, query_string=True) ##CACHE
def list_order():
...

@app.route("/order/list_per_users/", methods=['GET'])
@cache.cached(timeout=30, query_string=True) ##CACHE
def list_per_users():
~~~

## Exemplo

Fazendo um teste, o ganho é significativo, chegando até ser 20x mais rápido do que uma consulta padrão do postgres.

![img36](/images/microservice_project/img36.png)

## Microservice Project – Finalização

Então, chegamos ao fim desse projeto.

Durante o desenvolvimento do código, me esbarrei em diversos erros e problemas que pareciam não ter mais sentido, principalmente nas partes onde precisava fazer algum teste no ambiente da OCI. Como ainda estava aprendendo algumas coisas novas sobre a criação de aplicação em nuvem, passei por problemas essenciais para aprender e entender como devo arquitetar todo um local de produção. Também aprendi diversas coisas sobre persistência de dados, e como trabalhar com as manipulações das informações disponibilizadas pelo usuário. Sobre Docker, aprendi muito como criar Docker-Compose, visto que é extremamente importante saber isso para criação de containers rapidamente, e pude ver um pouco mais sobre Dockerfile, e buildar minhas próprias imagens, de acordo com a necessidade da aplicação. Pude me aprofundar na criação e gerenciamento do broker de mensagem utilizando Rabbit, e em como criar uma estrutura complexa utilizando o conceito RPC(Remote Procedure Call), para o retorno de informações ao requisitor, que no nosso caso é o usuário. Com todos esses aprendizados pude concluir o objetivo final que era criar microsserviços independentes e caso algum deles pare de funcionar, não cause indisponibilidade a todo a aplicação.

Uma coisa que deixei de fazer nesse projeto, mas que pretendo abordar em outro momento, é a criação de testes unitários, visto que isso é um requisito quase obrigatório no mercado de trabalho dos desenvolvedores. Então, podemos riscar de nosso fluxo de desenvolvimento essa et

![img37](/images/microservice_project/img37.png)

Espero que tenha aproveitado todo o desenvolvimento do projeto, e que sua leitura tenha agregado conhecimentos.