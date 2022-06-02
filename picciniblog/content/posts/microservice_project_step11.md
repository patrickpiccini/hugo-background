---
author:
  name: "Patrick Piccini"
date: 2025-05-26T9:02:01-03:00
linktitle: Microservice Project – Step 11
type:
- post
- posts
title: Microservice Project – Step 11
weight: 10
---
![microservices](/images/microservice_project/microservices.png)
## Table of Contents
- [Tag API](#tag-api)
- [Controlador DataBase](#controlador-database)
    - [Manipulação de datas](#manipula%C3%A7%C3%A3o-de-datas)
- [Create\_order](#create%5C_order)
- [List\_all\_orders](#list%5C_all%5C_orders)
- [Métodos Criado](#m%C3%A9todos-criado)
- [Controlador Rabbit](#controlador-rabbit)
- [Resumo](#resumo)
---

No passo anterior concluímos a arquitetura do Microsserviço 2, e fizemos a conexão com os serviços do Rabbit e Postgres. Agora, vamos fazer a lógica do sistema dos orders, onde diferenciaremos as interações com o banco de dados através de uma tag, enviada pela API, e dependendo da interação de for solicitada, haverá o acionamento de uma função equivalente a solicitação.

Na pasta raiz da aplicação MS-Application, vamos criar os seguintes arquivos

~~~ Estrutura
MS-application
└───MS1
    │   docker-compose-microservice1.yml
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

## Tag API

Antes de começarmos a codar, vamos entender o cenário onde paramos. Temos uma API que recebe as informações de uma criação de um order, por exemplo, e envia essas informações para o microsserviço. Para o microsserviço entender o que deve ser feito com aquelas informações recebidas, a API deverá envia uma tag juntamente ao corpo das informações, que posteriormente será identificada. Entendendo esse fluxo, vamos voltar ao código da API e fazer essa pequena alteração da inserção da tag.

No arquivo _server.py, vamos_ ao método de criação de usuário que abordamos nos Steps anteriores, e inserirmos ao payload que recebemos um item chamado _type_, com o valor _create\_order_.

~~~ python
@app.route("/order/create_order/", methods=['POST'])
    def create_order():
        if request.method == 'POST':
            payload = request.get_json()
            payload['type'] = 'create_order'

            corr_id = rabbit_queues.rpc_async(json.dumps(payload), "order")
            while rabbit_queues.queue[corr_id] is None:
                time.sleep(0.1)

            return {'Status': 200, 'Message': json.loads(rabbit_queues.queue[corr_id])}
        else:
            return {'Status': 404, 'Message': 'Erro no envio do method'}
~~~
## Controlador DataBase

O controlador do database será o responsável por fazer o famoso **CRUD.** CRUD é a composição da primeira letra de 4 funções básicas de um sistema que trabalha com banco de dados:

✔️ C: Create (criar) - criar um novo registro.

👀 R: Read (ler) - ler (exibir) as informações de um registro.

♻️ U: Update (atualizar) - atualizar os dados do registro.

❌ D: Delete (apagar) - apagar um registro.

No desenvolvimento desse estágio mostrarei apenas uma interação com o banco de dados, visto que o desenvolvimento é muito complexo, e a leitura ficará muito maçante. O que faremos agora será a inclusão de um order ao banco. Para isso devemos inserir as informações através de um comando SQL.

Primeiramente, devemos importar todas as dependências necessárias, que serão a conexão com o banco, feita no _database\_connection.py,_ e a biblioteca datetime.

~~~ python
from config.database_connection import ConnectionDatabase
from datetime import datetime
~~~

Vamos criar uma Classe chamada **PostgresWorker** , com o construtor _\_\_init\_\__ contendo a conexão com o banco, captura do horário atual, e a formatação da data.a.

~~~ python
    def __init__(self):
        self.PSQL = ConnectionDatabase()
        self.date_time = datetime.now()
        self.date_time_formate = self.date_time.strftime('%Y/%m/%d %H:%M')
~~~

### Manipulação de datas

Para entender um pouco melhor sobre a manipulação de datas, confira o tópico &quot;Manipula de datas&quot; do Step 8.

## Create\_order

O que faremos agora será criar uma função chamada _create\_order_ que fará a inserção/criação de um order no nosso banco de dados. O método receberá como parâmetro _data_, que serão os dados para fazermos a interação com o banco.

Devemos verificar antes se existe cadastrado o usuário no banco de dados. Para isso passaremos o nick\_name da pessoa a uma função chamada _information\_user._ Dependendo do retorno, o usuário receberá uma mensagem informando que não existe esse usuário cadastrado no manco de dados.

Dentro do método também teremos a variável _query\_insert_ que conterá a query para a execução do comando. Na variável _vars\_query_ terão as informações recebidas da fila, que serão inseridas da query. A função _&quot;cursor.execute(query\_update, vars\_query)&quot;_ é responsável pela execução do comando, onde pegará a _query\_update_, inserir os valores de _vars\_query,_ e executar a instrução. A função _&quot;connection.commit&quot;_ é responsável por fazer as alterações do banco para a persistência do database. Por fim, é retornado alguma informação. Tudo isso ficará dentro de um try-except, para caso haja alguma falha na interação com o banco, seja lançado um except com o erro.

Como esse caso é apenas uma inserção no banco, apenas retornará uma mensagem ao usuário. Contudo, em casos de retorno de alguma informação do banco ao usuário, essas informações serão passadas no return.

~~~ python
    # create a order in database
    def create_order(self, data):
        try:
            user_id = self.information_user(data['nick_name'])
            if user_id == None:
                return f"[X] DON'T HAVE THIS USER IN DATA-BASE!"

            total_value = data['item_quantity'] * data['item_price']

            query_insert = 'INSERT INTO orders (user_id ,item_description ,item_quantity , item_price, total_value, created_at, updated_at)VALUES (%s,%s,%s,%s,%s,%s,%s)'
            vars_query = (user_id[0], data['item_description'], data['item_quantity'],
                          data['item_price'], total_value, self.date_time_formate, self.date_time_formate)
            self.PSQL.cursor.execute(query_insert, vars_query)
            self.PSQL.connection.commit()

            print('[✓] INSERTION DONE IN POSTGRES!')
            return '[✓] Order created successfully! '
        except Exception as error:
            print(error)
            return f'[X] ERROR INSERTING IN POSTGRES! {error}'
        finally:
            self.PSQL.cursor.close()
~~~

## List\_all\_orders

Para exemplificar uma situação em que precisamos retornar uma informação do banco de dados para o usuário, vamos criar a função _list\_all\_orders._ A estrutura da função será semelhante a que criamos acima. Teremos somente uma query de instrução, que será um select dos _order\_id_ e _item\_description_ de toda a tabela. Serão capturadas as informações através do comando &quot;cursor.fetchall&quot;, e manipuladas aos dados recebidos do banco, para mostrar mais organizado ao usuário. Tudo isso ficará dentro de um try-except, para caso haja alguma falha na interação com o banco, seja lançado um except com o erro.

~~~ python
# Show all orders of batabase
    def list_all_orders(self):
        try:
            sql_select_query = 'SELECT order_id, item_description FROM orders'
            self.PSQL.cursor.execute(sql_select_query)
            record = self.PSQL.cursor.fetchall()
            self.PSQL.connection.commit()

            dict_all_orders = []
            for index in range(len(record)):
                dict_respose={
                    'order_id': record[index][0],
                    'item_description': record[index][1],
                }
                dict_all_orders.append(dict_respose)

            print('[✓] SELECT ALL USER DONE SUCCESSFULLY IN POSTGRES!')
            return dict_all_orders
        except Exception as error:
            print(error)
            return f'[X] ERROR ON SELECT IN POSTGRES! {error}'
        finally:
            self.PSQL.cursor.close()
~~~

Com isso podemos ter uma base de como fazer o nosso CRUD. As demais funções poderão ser visualizadas no repositório git do projeto.

## Métodos Criado

Os métodos precisaremos criar são:

- **create\_order** ­– Cria um order;
- **edit\_order** – Edita as informações de um order;
- **list\_all\_orders** – Lista todos os orders cadastrados;
- **list\_per\_users** – Lista os orders por usuário;
- **show\_order** – Mostra todas as informações de um order;
- **delete\_order** – Deleta um order;
- **information\_user** – Captura o user\_id de um usuário;
- **information\_order** – Seleciona todas as informações de um order.

## Controlador Rabbit

O controlador do rabbit será o responsável por identificar a tag recebida e disparará uma função para a interação com o banco de dados. Porém o retorno do bando de dados deve ser retornado também ao usuário que consumiu a API. Para isso, publicaremos esse response através da fila de controle que criamos no Step6.

No arquivo _rabbit\_worker.py_ vamos iniciar importando as dependências.

~~~ python
import json, pika
from database_controller.postgres_worker import PostgresWorker
~~~

Vamos criar uma classe chamada **RabbitWorker,** e em seu construtor _\_\_init\_\__ criaremos um atributo vazio chamada _self.data._
~~~ python
class RabbitWorker():

    def __init__(self):
        self.data = ''
~~~

Logo abaixo, criaremos o método que chamamos no arquivo _main.py,_ chamado de _call-back._ Ele é o responsável por processar os dados recebidos, e retornar à fila de controle, um response ao usuário. Nos parâmetros deverão conter _ch, method, props_ e _body._ No atributo _self.data_ iremos carregá-lo com as informações recebidas do body. Logo abaixo, criaremos uma variável onde chamaremos o método _database\_manipulation(self.data)_ que criaremos posteriormente.

Após isso, através do atributo _ch_, iremos criar a parte da publicação da mensagem de response na fila de controle e dando um ACK na fila de controle.

~~~ python
    def callback(self, ch, method, props, body):
        self.data = json.loads(body)
        response_work = self.database_manipulation(self.data)

        ch.basic_publish(exchange='',
                         routing_key=props.reply_to,
                         properties=pika.BasicProperties(
                             correlation_id=props.correlation_id),
                         body=json.dumps(response_work))
        ch.basic_ack(delivery_tag=method.delivery_tag)
~~~

Como dito anteriormente, agora vamos criar o método _database\_manipulation_, que é responsável por identificar a tag que há no corpo dos dados recebidos da API, e acionar os métodos contidos no arquivo _postgres\_worker.py_ para a fazer a ação solicitada da requisição.

O que faremos é bem simples. Quando for recebido uma mensagem na fila de usuário, a mensagem será consumida. Será identificado o type da mensagem, que no caso é nossa tag, e após isso, chamaremos uma função equivalente ao type para executar um CRUD no banco de dados.

Essa identificação será através de um _if-elif_ da informação &quot;type&quot;, contida em data.

| Type | Function | Objective |
| --- | --- | --- |
| create | insert\_user | Criar usuário |
| update | alter\_user | Alterar informações de usuário |
| update\_password | alter\_password | Alterar senha de usuário |
| show\_all | show\_all\_user | Mostrar todos os usuários |
| show\_one | show\_one\_user | Mostrar um usuário |
| delete\_user | delete\_user | Deletar um usuário |

~~~ python
    def database_manipulation(self, data):
        # start connection whit postgres
        psql = PostgresWorker()
        if data['type'] == 'create_order':
            print('entrou no type')
            return psql.create_order(data)
        elif data['type'] == 'update_order':
            return psql.edit_order(data)
        elif data['type'] == 'list_all_orders':
            return psql.list_all_orders()
        elif data['type'] == 'list_per_users':
            return psql.list_per_users(data)
        elif data['type'] == 'show_order':
            return psql.show_order(data)
        elif data['type'] == 'delete_order':
            return psql.delete_order(data)
~~~

## Resumo

Nesse passo fizemos uma pequena alteração na API, inserindo uma nova informação no payload que será o ponto crucial para a identificação da ação que deve ser feita no banco de dados. Criamos o database\_worker onde há as funções de interação/manipulação do banco de dados, onde foi feito um CRUD. Criamos o rabbit\_worker, que é o responsável por identificar o que está sendo solicitado através da tag, pegar as informações recebidas e utilizá-las nas funções contidas no database\_worker. Por fim, é publicado na fila auxiliar uma mensagem de response, onde o usuário receberá essa informação já processada..