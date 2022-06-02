---
author:
  name: "Patrick Piccini"
date: 2025-05-26T9:01:54-03:00
linktitle: Microservice Project – Step 4
type:
- post
- posts
title: Microservice Project – Step 4
weight: 10
---
![microservices](/images/microservice_project/microservices.png)
## Table of Contents
- [Requirements](#requirements)
- [Criação API](#cria%C3%A7%C3%A3o-api)
- [Dockerfile](#dockerfile)
- [Docker-Compose](#docker-compose)
- [Resumo](#resumo)
---

Nesse passo, iremos começar a criar o rosto da API. Irei explicar como estruturei as rotas para a conexão utilizando Flask, e como funciona o disparo das funções das rotas. Criaremos a própria imagem para futuramente rodar a aplicação utilizando Dockerfile, juntamente a um Docker-Compose para subir as instancias Docker. Por fim, farei um teste de conexão localmente as rotas que criei.

No diretório raiz, criaremos a pasta API com os seguintes arquivos:

~~~ Estrutura
MS-application
└───API
    │   docker-compose-api.yml
    │   Dockerfile
    │   requirements.txt
    │   server.py
~~~

## Requirements

No arquivo _requeriments.txt_ ficarão todas as instalações de bibliotecas e framework que serão utilizadas na aplicação da API. As bibliotecas devem ficar separadas por linha, visto que usaremos um comando para uma instalação recursiva.

~~~ requirements
requirements.txt
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

## Criação API

Dentro o arquivo _server.py_ iremos criar uma espécie de API-Gateway, onde ela será responsável apenas por distribuir as informações para as filas do rabbit. Cada rota terá sua funcionalidade, como **create\_user** para criar um usuário, e **show\_all\_user** para mostrar todos os usuários. A mesma lógica será aplicada referente aos **orders** , como **create\_order, list\_order** e assim sucessivamente.

Mostrarei apenas a criação de uma das rotas, pois a estrutura será a mesma para todas as outras, basta alterar o nome da rota e função que será disparada após acessá-la.

Primeiro, deve-se importar o framework **Flask** juntamente a funcionalidade **request.**

Devemos criar uma classe chamada **Api\_server** e instanciar o Flask a variável **app.** Após isso, para definir uma rota iremos utilizar o @app.route(&quot;rota&quot;, method), onde apontaremos /user/create\_user/ como destino, sendo o **POST** o único método que essa rota irá aceitar.

Abaixo da rota fica a função que será disparada após ser acessada, que nomearemos de **create\_user().** Dentro da &#39;def&#39;, faremos uma verificação se o method é o mesmo que esperamos nessa função. Caso isso seja atendido, coloquei como teste, um return que apenas mostrará o que enviamos no corpo da requisição. Caso não atenda, será mostrado um erro referente ao método enviado.

~~~ python
from flask import Flask, request

class Api_server():
    app = Flask(__name__)

    @app.route("/user/create_user/", methods=['POST']) # <- rota que será acessada
    def create_user(): # <- metodo que será disparado ao acessar a rota acima 
        if request.method == 'POST':
            payload = request.get_json()

            return {'Status': 200, 'Message': payload}
        else:
            return {'Status': 404, 'Message': 'Erro no envio do method'}
~~~

Para executar o servidor, nas ultimas linhas iremos instanciar a classe que criamos, e utilizaremos o comando app.run(host, porta), para definir o host e porta que irá rodar a API.

~~~ python
if __name__ == '__main__':

    APP = Api_server()
    APP.app.run('0.0.0.0', 7000)
~~~

Para testar, executei o arquivo _server.py_ e enviei um JSON como corpo da requisição para a rota que criei anteriormente, juntamente ao método POST:

![img25](/images/microservice_project/img25.jpg)

![img26](/images/microservice_project/img26.jpg)

OBS: Criei todas as demais rotas, mas não irei exemplificar cada uma delas pois a funcionalidade é a mesma. Se tiverem curiosidade, basta ir ao repositório da aplicação e olhar as outras rotas.

## Dockerfile

No arquivo _Dockerfile_ iremos criar uma imagem personalizada, adequada a aplicação que criaremos:

~~~ docker
FROM python:3.8-alpine                  <- Imagem base utilizada
COPY . /app/                            <- cópia da pasta API para /app/
ENV HOST='10.0.0.34'                    <- ip privado 
ENV HOST_DATABASE='10.5.0.11'           <- ip do postgres

ENV CACHE_TYPE='redis'                  <- type do redis
ENV CACHE_REDIS_HOST='10.0.0.25'        <- ip do redis
ENV CACHE_REDIS_PORT='6379'             <- porta do redis
ENV CACHE_REDIS_DB=0

EXPOSE 7000                             <- porta que será a API
WORKDIR /app                            <- diretório de trabalho
RUN pip install --upgrade pip           <- instalação do pip
RUN pip install -r requirements.txt     <- instalação das libs
CMD python server.py                    <- comando para iniciar a API
~~~

## Docker-Compose

O Docker-Compose para subir a aplicação é bem simples. Terá apenas um service onde irá rodar o Dockerfile no &#39;_build: . &#39;_.A exposição da porta 7000, que será a porta da API, e apontamento da Network que foi criada no Docker-Compose, onde é preciso especificar que a rede que estará sendo utilizada é externa, na parte _external: true_

~~~ docker
version: "3.7"
services:
  api:
    build: .
    ports:
      - "7000:7000"
    networks:
      - internal-network

networks:
    internal-network:
        external: true
~~~
## Resumo

Nesse Step, criamos o arquivo _requeriments.txt_ que é responsável por conter todas as bibliotecas que iremos utilizar. Foi explicado como funciona o sistema de rotas do Flask e também criamos uma rota como exemplificar. Por fim, desenvolvemos os arquivos Dockerfile e Docker-Compose, que serão responsáveis pela inicialização da API.