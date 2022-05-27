---
author:
  name: "Patrick Piccini"
date: 2022-05-26T9:01:55-03:00
linktitle: Microservice Project – Step 5
type:
- post
- posts
title: Microservice Project – Step 5
weight: 10
---
## Table of Contents
- [Postgres](#postgres)
    - [Create\_tables](#create%5C_tables)
- [Rabbit MQ](#rabbit-mq)
- [Redis](#redis)
- [Resumo](#resumo)
---

Como visto no Step4, foi iniciado os primeiros passos pra interagir com a API gateway, e também, foi criado as configurações para a inicialização do container, quando for subir para a OCI.

Nesse passo, mostrai como configurar a conexão com os servidores do Rabbit, Redis e Postgres. E como planejei a arquitetura de pastas e módulos, para ter uma aplicação organizada.

Dentro da pasta API, devemos criar uma subpasta chamada config com os seguintes arquivos

~~~ Estrutura
MS-application
└───API
    │   docker-compose-api.yml
    │   Dockerfile
    │   requirements.txt
    │   server.py
    │
    ├───config
    │       database_connection.py
    │       rabbitmq_connection.py
    │       redis_connection.py
    │       __init__.py
~~~

## Postgres

Antes de continuar, deve-se ter certeza de que executou todas as instalações das bibliotecas, contida no arquivo _requeriments.txt._

Para começarmos, devemos importar os módulos _psycopg2_e _os,_ que iremos utilizar para a conexão.

~~~ python
import psycopg2, os
~~~

Como teremos que apontar o host do servidor Postgres para nossa aplicação, na criação do Dockerfile foi criado uma variável de ambiente chamada HOST\_DATABASE, contendo o ip privado da instancia da OCI. Porém, essa variável de ambiente só será utilizada em Produção. Sendo assim, criei uma variável dentro do arquivo _database\_connection.py_ contendo o ip público, somente para fins desenvolvimento em Homologação, utilizando os serviços que já estão rodando.

Já que iremos nos conectar a um banco de dados, essa deve ser uma das primeiras coisas que deve acontecer quando a API for inicializada. Para isso, criaremos uma classe chamada _ConnectionDatabase(),_ onde em seu _\_\_init\_\__ iremos fazer esse processo de conexão.

Dentro da função connect da biblioteca psycopg2, vamos colocar os parâmetros _host,_ que será o ip da instancia OCI, _port_ contendo a numeração padrão da porta postgres(já foi liberada a porta no OCI para a conexão externa), _batabase_, contendo o nome do banco, _user_ e _password,_ contendo usuário e senha que foi colocado na imagem que subimos do postgres.

Após o apontamento dos parâmetros obrigatórios para a conexão, iremos criar um atributo chamado _cursor,_ onde ele será o responsável por executar qualquer interação com o banco de dados. Logo abaixo há um método _create\_tables_, que iremos criar logo em seguida. Para finalizar, tudo isso está dentro de um try-except, para caso haja alguma falha na conexão, seja lançado um except com o erro.

~~~ python
# HOST_DATABSE = os.environ['HOST_DATABASE']
HOST_DATABSE = '144.22.193.219'

class ConnectionDatabase():

    def __init__(self):
        try:
            self.connection  = psycopg2.connect(
                host=HOST_DATABSE,
                port=5432,
                database="baseapplication",
                user="postgres",
                password="postgres")
            self.cursor = self.connection.cursor()
            # start tables
            self.create_tables()
            print('[✓] Connected to Postgres')
        except Exception as error:
            print(f'[X] CONNECTING POSTGRES ERROR: {error}')
~~~


### Create\_tables

Antes de começar a fazer a função que irá criar as tabelas, vamos analisar o que precisará ser feito com o modelo ER abaixo:

![img27](/images/microservice_project/img27.png)

Iremos criar uma estrutura simples. Serão duas tabelas, _users_ e _orders_. Nelas conterão seus atributos, com seus determinados tipos. Note que na tabela orders, iremos herdar um atributo da tabela user, que será o _user\_id_, onde na tabela user é a PrimaryKey. Já na tabela orders, o atributo herdado _user\_id_, será tratado como uma chave estrangeira (ForeignKey).

Em relação a cardinalidade vamos ter 1 usuário pode ter vários orders, e 1 order pode ter somente um usuário, representados pelos símbolos abaixo.

![img28](/images/microservice_project/img28.jpg)

Após essa análise, devemos criar o método _create­\_table, e criar as tabelas._ Para executar comandos SQL através da biblioteca, basta escrever em uma string o comando SQL que deseja executar, e passar como parâmetro para função _execute._ Visto isso, primeiramente iremos criar um select da versão do Postgres, para que toda a vez que seja iniciado a conexão, seja mostrado a versão. Logo após devemos criar um atributo contendo toda a query de criação das duas tabelas, baseada do modelo ER visto anteriormente. Sendo assim, como método já foi chamado no _\_\_init\_\_,_ será criado as tabelas ao instanciar a classe **ConnectionDatabase** _._

~~~ python
def create_tables(self):
        self.cursor.execute("SELECT version();")
        record = self.cursor.fetchone()
        print("[✓] You are connected to - ", record)

        create_table_query = '''
            CREATE TABLE IF NOT EXISTS users (
                user_id SERIAL NOT NULL,
                nick_name varchar(50) UNIQUE NOT NULL,
                password varchar(256) NOT NULL,
                full_name varchar(50) NOT NULL,
                cpf varchar(11) NOT NULL,
                email varchar(50) NOT NULL,
                phone_number varchar(50) NOT NULL,
                created_at TIMESTAMP,
                updated_at TIMESTAMP,
                PRIMARY KEY (user_id)
            );

            CREATE TABLE IF NOT EXISTS orders (
                order_id SERIAL NOT NULL,
                user_id integer NOT NULL,
                item_description varchar(256) NOT NULL,
                item_quantity integer NOT NULL,
                item_price integer NOT NULL,
                total_value integer NOT NULL,
                created_at TIMESTAMP,
                updated_at TIMESTAMP,
                PRIMARY KEY (order_id),
                FOREIGN KEY(user_id) REFERENCES users(user_id)
            );'''
        
        self.cursor.execute(create_table_query)
        self.connection.commit()
        print('[✓] Created tables on DataBase')
~~~


## Rabbit MQ

O Rabbit é um message borker open source fácil e leve de ser implementado, tanto local, quanto em nuvem. O rabbit suporta vários tipos de protocolo de mensagens, para haver facilidade na comunicação entre aplicações. Existem muitos outros recursos, porém, não irei detalhar eles nesse projeto.

Nesse momento, iremos fazer somente a conexão com o servidor Rabbit, onde usaremos uma variável de ambiente como no postgres, no entanto ela será a HOST, contida no **Dockerfile**. Da mesma forma que na conexão do postgres, criei uma variável estática contendo o ip público, somente para desenvolver a aplicação local, utilizando o servidor que já estão rodando os serviços.

Começamos com a importação das bibliotecas necessárias e o apontamento da variável com o IP Público da instancia OCI.

import pika, os

~~~ python
import pika, os

# HOST_RABBIT = os.environ['HOST']
HOST_DATABSE = '144.22.193.219'
~~~

Agora iremos criar uma classe chamada _RabbitConnection(),_ e faremos a conexão no método _\_\_init\_\_._ Utilizando o função de conexão da biblioteca pika chamada _BlockingConnection(),_ passaremos como parâmetros, através do metodo _ConnectionParameters()_ o host, contida na variável HOST\_RABBIT, e a porta padrão 5672. Em seguida, será criado um atributo chamado _channel_, que será responsável por executar toda e qualquer interação com o servidor rabbit, tudo isso foi criado em um try-catch, para caso haja alguma falha na conexão, seja lançado um except com o erro.

~~~ python
class RabbitConnection():

    def __init__(self) :
        try:
            self.connection = pika.BlockingConnection(
                pika.ConnectionParameters(host=HOST_RABBIT, port=5672))
            self.channel = self.connection.channel()
            print('[✓] Connected to RabbitMQ server')
            
        except Exception as error:
            print(f'[X] CONNECTING RABBIT MQ ERROR: {error}')
~~~

## Redis

Redis é um armazenamento de estrutura de dados na memória de código aberto (licenciado BSD), usado como banco de dados, cache, agente de mensagens e mecanismo de streaming. Um dos serviços que se destaca é o cache, que iremos utilizar nessa aplicação.

Para criar a conexão com o servidor Redis, iremos utilizar a variável de ambiente CACHE\_REDIS\_HOST, porem, para a fim de desenvolvimento, irei criar uma variável local contendo o ip público da instancia.

Nessa configuração iremos fazer algo um pouco diferente. Como o Redis e Flask conseguem trabalhar junto nativamente, a configuração para a conexão ao servidor, será feita através do arquivo principal _server.py_, porém, será mostrado como fazer isso no próximo passo.

No momento a configuração será a seguinte: Importar a biblioteca _os,_ criar uma classe chamada _BaseConfig(),_ e dentro dela, apontar as variáveis de ambiente contidas no Dockerfile, veja:

~~~ python
import os

class BaseConfig(object):
    CACHE_TYPE = os.environ['CACHE_TYPE']
    CACHE_REDIS_HOST = os.environ['CACHE_REDIS_HOST']
    CACHE_REDIS_PORT = os.environ['CACHE_REDIS_PORT']
    CACHE_REDIS_DB = os.environ['CACHE_REDIS_DB']
~~~

Como irei testar localmente para homologação, criarei a mesma classe, porém, com atributos com valor estático do servidor.

~~~ python
class BaseConfig(object):
    CACHE_TYPE = 'redis' 
    CACHE_REDIS_HOST = '144.22.193.219'
    CACHE_REDIS_PORT = '6379'
    CACHE_REDIS_DB = 0
~~~

## Resumo

Nesse Step, criamos as conexões com os servidores Rabbit, Redis e Postgres que já estavam rodando a instancia da OCI. Criamos a conexão com o Postgres. Foi explanado o modelo de entidade relacional que posteriormente o desenvolvemos, criando as devidas tabelas planejadas. Vimos um breve resumo do que é o rabbit e criamos a conexão com o servidor que está rodando na OCI. E por fim, apontado a conexão com o Redis, porém não utilizaremos nesse momento.