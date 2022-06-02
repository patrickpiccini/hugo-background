---
author:
  name: "Patrick Piccini"
date: 2025-05-26T9:01:53-03:00
linktitle: Microservice Project – Step 3
type:
- post
- posts
title: Microservice Project – Step 3
weight: 10
---
## Table of Contents
- [Criação do Docker-Compose-Services](#cria%C3%A7%C3%A3o-do-docker-compose-services)
- [Teste na OCI](#teste-na-oci)
- [Resumo](#resumo)
---

Nesse próximo passo, parto do pressuposto que meu ambiente já está configurado e com todas as instalações feitas, assim, posso começar a criar as primeiras linhas de código.

Devemos começar criando uma pasta chamada _MS-Application._ Essa pasta será onde iremos criar todos os microsserviços, configuração de containers de serviço, e mais alguns detalhes, que serão mostrados no decorrer das publicações.

Na raiz da pasta, criamos um arquivo chamado _docker-compose-services.yml._

~~~ Estrutura
MS-Application
│   docker-compose-services.yml
~~~
## Criação do Docker-Compose-Services

Como vamos usar containers para praticamente tudo, será necessário que os servidores Rabbit, Redis, Postgres, e interfaces de manager sejam criadas separadamente em seus devidos containers. É necessária essa separação para que nenhum servidor dependa do outro para funcionar. Com isso, já temos os primeiros passos na criação da arquitetura de microsserviços.

Para inicializar esses serviços rapidamente, iremos desenvolver um Docker-Compose e configuraremos uma _Network_, onde posteriormente estarão todos os serviços dentro dessa mesma rede.

A rede foi nomeada de _internal-network,_ nela, iremos configurar o endereço 10.5.0.0 sendo uma rede classe A, com uma máscara de rede /16. Também iremos definir o gateway da rede para 10.5.0.1.

~~~ docker
version: "3.7"
networks:
  network-service:
    driver: bridge
    name: internal-network
    ipam:
      config:
        - subnet: 10.5.0.0/16
          gateway: 10.5.0.1
~~~

Nos serviços, iniciamos configurando o RabbitMQ. Utilizaremos a imagem _rabbitmq:3-management-alpine,_ que é uma imagem mínima do Docker baseada no Alpine Linux com um índice de pacotes completo e apenas 5 MB de tamanho. Nas variáveis de ambiente é deixado com o valor padrão o user, password e host, que será utilizado posteriormente para conectar ao manager do rabbit. Nas portas, devemos colocar 15672 para o manager e 5672 para o servidor. E por fim, iremos configurar o contêiner para ficar dentro da Network que foi criada anteriormente, apontando também um ip estático para essa instancia.

[Alpine](https://hub.docker.com/_/alpine)

~~~ docker
services:
  rabbitmq:
    image: "rabbitmq:3-management-alpine"
    environment:
      RABBITMQ_DEFAULT_USER: "guest"
      RABBITMQ_DEFAULT_PASS: "guest"
      RABBITMQ_DEFAULT_VHOST: "/"
    ports:
      - "15672:15672"
      - "5672:5672"
    labels:
      NAME: "rabbitmq1"
    networks:
      network-service:
        ipv4_address: 10.5.0.10
~~~

A estrutura dos próximos serviços é bem semelhante a essa primeira, com apenas alguns detalhes como diferença. No Postgres, utilizaremos a imagem _postgres:13_, iremos indicar as variáveis padrões do user e password para conexão a instância, apontaremos a porta padrão 5432, criaremos um volume compartilhado de _.data_ para _/data/,_ configuraremosaNetwork com o ip estático, e um detalhe muito importante, será colocado o parâmetro _restart: Always_ que fará com que a instância seja reiniciada caso pare de funcionar. Se for interrompido manualmente, ele será reiniciado somente quando o daemon do Docker for reiniciado ou o próprio contêiner for reiniciado manualmente.

~~~ docker
  postgres:
    container_name: postgres_container
    image: postgres:13
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: baseapplication
    ports:
      - "5432:5432"
    volumes:
      - ./data:/data/
    networks:
      network-service:
        ipv4_address: 10.5.0.11
~~~

Para ter uma parte visual do banco de dados, faremos a criação de uma instância com o PgAdmin, que poderemos conectar ao servidor do postgres via Web. Sendo assim, iremos utilizar a imagem _dpage/pgadmin4,_ com as variáveis de e-mail e password para entrarmos na página web. Iremos apontar a porta de 16543/80 (external/internal), compartilharemos os volumes./data/:/data/ e ./postgres-backup:/var/lib/postgresql/backups, iremos configurar a network com um ip estático, e colaremos uma dependência no parâmetro _depends\_on – postgres._ Isso fará com o a instância do PgAdmin só seja criada, após a criação da instância do postgres.

~~~ docker
  pgadmin:
    container_name: pgadmin_container
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "16543:80"
    depends_on:
      - postgres
    volumes:
      - ./data/:/data/
      - ./postgres-backup:/var/lib/postgresql/backups
    networks:
      network-service:
        ipv4_address: 10.5.0.12
~~~

E para camada de cache, iremos criar uma instância com Redis utilizando a imagem _redis:alpine_, apontando a porta padrão 6379, e configurando a Network com um ip estático.

~~~ docker
  redis:
    image: redis:alpine
    container_name: redis-container
    ports:
      - "6379:6379"
    networks:
      network-service:
        ipv4_address: 10.5.0.9
~~~

No final, teremos vários containers rodando dentro de uma Rede, tendo algo parecido com a seguinte imagem:

![img16](/images/microservice_project/img16.jpg)

## Teste na OCI

Agora que temos pronto o Docker-Compose com todos os serviços que utilizaremos, devemos fazer um teste dentro da VM que criamos na OCI. Para conseguir compartilhar o código, faço _um git commit_ de minha pasta MS-Application, e dentro da instância OCI dou um _git pull_ para baixar.

OBS: Não entrarei em detalhes nos comandos do git, caso tenha alguma dúvida especifica, consulte a documentação oficial: [**https://comandosgit.github.io/**](https://comandosgit.github.io/)

Dentro a instância, inicio meus serviços com o comando _docker-compose -f docker-compose-services.yml up,_ e passo o nome do arquivo que deverá ser iniciado.

Nesse momento começará a baixar as imagens dos contêineres.

![img17](/images/microservice_project/img17.jpg)

![img18](/images/microservice_project/img18.jpg)

![img19](/images/microservice_project/img19.jpg)

Agora que está tudo rodando, pude fazer um teste me conectando ao manager do RabbitMQ pela Web. Porém como a aplicação está em cloud, é necessário que nas configurações da VNC do projeto, sejam expostas as portas que iremos utilizar para nos conectar externamente.

![img20](/images/microservice_project/img20.jpg)

Na OCI, em Networking > Virtual Cloud Networks > sua\_vnc > Security List Details > Ingress Rules. iremos adicionar as portas que utilizaremo

![img21](/images/microservice_project/img21.jpg)

![img22](/images/microservice_project/img22.jpg)

Após a liberação da porta, consegui ter acesso aos meus contêineres, tanto o do rabbit quando ao pgadmin, que também liberei a porta. Percebam que para me conectar aos meus contêineres, utilizo o ip public da VM que estou utilizando.

![img23](/images/microservice_project/img23.jpg)

![img24](/images/microservice_project/img24.jpg)

## Resumo

Nesse Step, criamos o arquivo _docker-compose-services.yml_ que é responsável por iniciar os servidores RabbitMQ, Postgres e Redis, juntamente as interfaces PgAdmin e RabbitManager. Foram liberadas as portas dos servidores para visualização externa, e também foi feito um teste acessando as interfaces externamente à OCI, tendo sucesso na conexão.