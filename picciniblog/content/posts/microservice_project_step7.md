---
author:
  name: "Patrick Piccini"
date: 2022-05-26T9:01:57-03:00
linktitle: Microservice Project – Step 7
type:
- post
- posts
title: Microservice Project – Step 7
weight: 10
---
## Table of Contents
- [Config](#config)
- [Requeriments](#requeriments)
- [Docker-Compose](#docker-compose)
- [Dockerfile](#dockerfile)
- [Main.py](#mainpy)
- [Resumo](#resumo)
---

Agora vamos para o próximo nível da nossa aplicação. Como já temos a API pré-configurada com as filas e as tabelas do banco criados, começaremos a criar os microservices para consumi-las. A arquitetura do serviço será bem semelhante ao que criamos na API, contendo _requirements_, _dockerfile_, _docker-compose_, pasta de _config_ e mais algumas coisas.

Na pasta raiz da aplicação MS-Application, vamos criar os seguintes arquivos:

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
~~~

## Config

Na pasta **config** , os arquivos _database\_connection.py_ e _rabbitmq\_connection.py_ são exatamente as mesmas configurações dos arquivos contido na API, então apenas vamos copiar aos arquivos da API.

## Requeriments

No arquivo _requeriments.txt_ ficarão todas as instalações de bibliotecas e framework que serão utilizados na aplicação do microservices. As bibliotecas devem ficar separadas por linha, visto que usaremos um comando para uma instalação recursiva.

~~~ requirements
requirements.txt
   psycopg2-binary
   argon2-cffi
   flask
   pika
~~~
Comando para instalação:
~~~ shell
pip install -r requeriments.txt
~~~

## Docker-Compose

No arquivo _docker-compose-microservice1.yml,_ configuraremos a inicialização da imagem do serviço. Será feito o Build da imagem que criaremos posteriormente, apontado a Network que criamos juntamente aos serviços logo no início do projeto.

~~~ docker
version: "3.7"
services:
  microservice1:
    build: .
    networks:
      - internal-network

networks:
    internal-network:
        external: true
~~~

## Dockerfile

No arquivo Dockerfile iremos configurar a imagem de nosso primeiro microservices, contendo as variáveis de ambiente adequadas para o serviço. Diferente da imagem do python que usamos na API, no microservice precisaremos instalar a imagem completa do python, sendo ela: python:3.8.

~~~ docker
FROM python:3.8
COPY . /app/
ENV HOST='10.0.0.25'
ENV HOST_DATABASE='10.5.0.11'
WORKDIR /app
RUN pip install --upgrade pip
RUN pip install -r requirements.txt
CMD python main.py
~~~

## Main.py

O arquivo _main.py_ será responsável por interceptar as mensagens enviadas para a fila de usuário.

Primeiramente, devemos importar os arquivos de conexão com o postgres e rabbit. Após isso, criaremos a classe _Main(),_ onde no método _\_\_init\_\_,_ chamaremos as conexões com os servidores.

~~~ python
from config.database_connection import ConnectionDatabase
from config.rabbitmq_connection import ConnectionRabbitMq
from rabbitmq_controller.rabbit_worker import RabbitWorker

class Main():

    def __init__(self):
        self.PSQL = ConnectionDatabase()
        self.RMQ = ConnectionRabbitMq()
        self.RMQ_WORKER = RabbitWorker()
~~~

Feito isso, criaremos um método chamado _consume\_queue,_ que consumirá a fila que desejamos. Nesse caso será _user,_ e posteriormente o consumo disparará uma função para o processamento do dado recebido da fila. Nesse momento não iremos criar a função de processamento, porem iremos apenas nomeá-la no lugar correto.

Com o comando _basic\_consume_ consumiremos a fila _user_, no on\_message\_callback será disparado a função para o processamento da informação recebida na fila.

Com o comando _start\_consuming,_ é processado o evento de I/O até que todos as mensagens sejam processadas.

~~~ python
    def consume_queue(self):
        self.RMQ.channel.basic_qos(prefetch_count=1)
        self.RMQ.channel.basic_consume(queue='user', on_message_callback=self.RMQ_WORKER.callback)

        print('     [⇄] Waiting for messages. To exit press CTRL+C')
        self.RMQ.channel.start_consuming()
~~~

Finalizaremos com a chamada do método _consume\_queue_ para inicializar o serviço.

~~~ python
if __name__ == '__main__':

    MA = Main()
    MA.consume_queue()
~~~

## Resumo

Nesse Step criamos nosso primeiro microservices para o processamento dos dados recebidos através da fila de usuário. Copiamos os arquivos de conexão _database\_connection.py_ e _rabbitmq\_connection.py_ da API, visto que é a mesma conexão que precisaremos fazer nos microservices. Também criamos o arquivo _main.py_ que será o responsável por inicializar o serviço e disparar o método _consume\_queue,_ que como o próprio nome diz, irá consumir a fila de usuários.