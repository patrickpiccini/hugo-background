---
author:
  name: "Patrick Piccini"
date: 2022-04-12T12:50:50-03:00
linktitle: Email com Google Authenticaito
type:
- post
- posts
title: Email com Google Authenticaito
weight: 10
---

## Table of Contents
---

- [Introdução](#introdu%C3%A7%C3%A3o)
    - [Pré-Requisitos](#pr%C3%A9-requisitos)
    - [Step 1 – Criar credenciais](#step-1--criar-credenciais)
    - [Step 2 – Instalação de bibliotecas](#step-2--instala%C3%A7%C3%A3o-de-bibliotecas)
    - [Step 3 – Criação de código de autenticação](#step-3--cria%C3%A7%C3%A3o-de-c%C3%B3digo-de-autentica%C3%A7%C3%A3o)
    - [Código Completo:](#c%C3%B3digo-completo)
    - [Step 4 – Criação do código de envio de e-mail](#step-4--cria%C3%A7%C3%A3o-do-c%C3%B3digo-de-envio-de-e-mail)
    - [Código Completo:](#c%C3%B3digo-completo)
    - [Step 5 – Execução](#step-5--execu%C3%A7%C3%A3o)
- [Conclusão](#conclus%C3%A3o)
    - [Referências](#refer%C3%AAncias)
---

## Introdução 

Nesse artigo irei abordar uma situação que recentemente a Google publicou referente a de login an conta google, utilizando somente o usuário e senha para se conectar em apps de terceiros. Segue um trecho da publicação:

_- &quot;Para proteger sua conta, o Google vai deixar de oferecer suporte para o uso de apps ou dispositivos de terceiros que solicitam login na Conta do Google usando apenas seu nome de usuário e senha. Essa mudança será válida a partir de 30 de maio de 2022. Continue lendo para mais informações.&quot; -_ [_Apps menos seguros e a Conta do Google_](https://support.google.com/accounts/answer/6010255?p=less-secure-apps&amp;hl=pt-BR&amp;visit_id=637824317526826001-2187211079&amp;rd=1).)

Tendo em vista que muitas aplicações do mercado utilizam esse método simples de login (usuário e senha), criei esse pequeno artigo explicando e desenvolvendo um método de autenticação a conta do google, através das APIs que o Google disponibiliza. Fazendo com que não haja a necessidade de liberar nas configurações da conta a opção de &quot;permitir apps menos seguros&quot;.

**Obs**: Os exemplos de códigos apresentados nesse artigo foram todos baseados nas documentações oficiais da Google, porém refartados, trazendo a clareza e a simplicidade no desenvolvimento da aplicação.

Para usar a Gmail API é necessário ter uma conta na Google Cloud Plataform, onde o cadastro pode ser feito [AQUI](https://console.cloud.google.com/freetrial/signup/tos?_ga=2.255782728.1788355950.1649683957-1725700722.1640005169&amp;_gac=1.61556062.1649692396.CjwKCAjwo8-SBhAlEiwAopc9W-9WErTEtw9O2DIPMgtBZHRMMb8iu52gwJgAgy-YPZidJP80yxSCahoCk94QAvD_BwE).


### Pré-Requisitos

- [Python](https://www.python.org/downloads/) 2.6 ou superior;

- Gerenciamento de pacotes [PIP](https://pypi.org/project/pip/);

- Um projeto do Google Cloud Platform com a API ativada. Para criar um projeto e ativar uma API, consulte [Criar um projeto e ativar a API](https://developers.google.com/workspace/guides/create-project)

A aplicação terá a seguinte arquitetura:

```
Aplication
    ↳_GoogleAuthenticator.py_
    ↳ _SendEmail.py_
    ↳ _credentials.json_
```

### Step 1 – Criar credenciais 

Para comunicação da aplicação e a API, deve ser baixado as credenciais de acesso, obtida em **≡** _> APIs &amp; Services > Credentials,_ depois em Click _Create credentials > API key._

Para visualizar a documentação oficial acesse [Create access credentials](https://developers.google.com/workspace/guides/create-credentials).

### Step 2 – Instalação de bibliotecas

Instalar a biblioteca do Google Client.

pip install --upgrade google-api-python-client google-auth-httplib2 google-auth-oauthlib pybase64 email-to

### Step 3 – Criação de código de autenticação

Criar um arquivo _GoogleAuthenticator.py_.

~~~python
pip install --upgrade google-api-python-client google-auth-httplib2 google-auth-oauthlib pybase64 email-to
~~~

Ao abrir o arquivo, deve-se fazer uma sequência de importações das bibliotecas instaladas.

~~~python
import os.path
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError
~~~



Abaixo das importações, deverá ser criado uma definition que será chamara para ocorrer a autenticação. Nela deve conter como parâmetros, _client\_secret\_file_(arquivo baixado contendo as credenciais de acesso), _api\_service\_name_(nome do serviço de API), _api\_version(_versão da API_)_ e _scopes(_é responsável por solicitar o acesso as APIs do Google_)._

~~~python
def authenticator(client_secret_file, api_service_name, api_version, *scopes):
    CLIENT_SECRET_FILE = client_secret_file
    API_SERVICE_NAME = api_service_name
    API_VERSION = api_version
    SCOPES = [scope for scope in scopes[0]]

    credentials = None
~~~

A lógica será bem simples. Primeiro deve verificar se existe no diretório o arquivo _token.json,_ que é responsável por conter em seu corpo todas informações de um usuário já autenticado pela aplicação. Caso não exista, o código ira ler um arquivo chamado _credentials.json,_ que dentro dele há as credenciais para a autenticação, como _client\_id, client\_secret_ entre outras. Ambos os arquivos são atribuídos a variável &quot; **credentials&quot;,** que será usado na chamada da API posteriormente

~~~python
    if os.path.exists('DocGoogle/token.json'):
        credentials = Credentials.from_authorized_user_file(
            'DocGoogle/token.json', SCOPES)
    
    if not credentials or not credentials.valid:
        if credentials and credentials.expired and credentials.refresh_token:
            credentials.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file(
                CLIENT_SECRET_FILE, SCOPES)
            credentials = flow.run_local_server(port=0)

~~~

Após a autenticação, iremos salvar as credenciais do usuário autenticado no arquivo _token.json,_ para que nas próximas vezes que a aplicação seja usada, não necessite passar pelo processo de autenticação novamente.

~~~python
        # Save the credentials for the next run
        with open('DocGoogle/token.json', 'w') as token:
            token.write(credentials.to_json())
~~~

Agora, é necessário fazer a chamada a API do google com os arquivos _token.json_ ou _credential.json_ que está na variável &quot; **credentials&quot;** com citado anteriormente. Com a função _build_ será construído um Resource Object para interagir com a API, e assim, casso as credenciais estejam corretas, retorne a confirmação do acesso ou não. Caso ocorra um erro na chamada a API, será disparado um except para o tratamento do erro.

~~~python
    try:
        # Call the Gmail API
        service = build(API_SERVICE_NAME, API_VERSION, credentials=credentials)
        print(API_SERVICE_NAME, 'service created successfully')
        return service

    except HttpError as error:
        # TO DO(developer) - Handle errors from gmail API.
        print('An error occurred: {}'.format(error))

~~~

### Código Completo:

~~~python
import os.path
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError


def authenticator(client_secret_file, api_service_name, api_version, *scopes):
    CLIENT_SECRET_FILE = client_secret_file
    API_SERVICE_NAME = api_service_name
    API_VERSION = api_version
    SCOPES = [scope for scope in scopes[0]]

    creds = None

    if os.path.exists('DocGoogle/token.json'):
        creds = Credentials.from_authorized_user_file(
            'DocGoogle/token.json', SCOPES)
    
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file(
                CLIENT_SECRET_FILE, SCOPES)
            creds = flow.run_local_server(port=0)
        # Save the credentials for the next run
        with open('DocGoogle/token.json', 'w') as token:
            token.write(creds.to_json())

    try:
        # Call the Gmail API
        service = build(API_SERVICE_NAME, API_VERSION, credentials=creds)
        print(API_SERVICE_NAME, 'service created successfully')
        return service

    except HttpError as error:
        # TO DO(developer) - Handle errors from gmail API.
        print('An error occurred: {}'.format(error))

~~~

### Step 4 – Criação do código de envio de e-mail

Criar um arquivo _SendEmail__.py_.

~~~python
import base64
from email import errors
from quickstart import authenticator
from email.mime.text import MIMEText
~~~

Ao abrir o arquivo, deve-se fazer uma sequência de importações das bibliotecas instaladas

Nota-se que foi importado a _def authenticator_ do arquivo _GoogleAuthenticator.py ._

É criado as variáveis _CLIENT\_SECRET\_FILE_ que é atribuído o arquivo credential.json, _API\_NAME_ contendo o nome da aplicação_, API\_VERSION_ com a versão, e _SCOPES_ com o link para a solicitação a API. Tudo isso é passado como parâmetro para a _def euthenticator._

~~~python
CLIENT_SECRET_FILE = 'DocGoogle/credentials.json'
API_NAME = 'gmail'
API_VERSION = 'v1'
SCOPES = ['https://mail.google.com/']

service = authenticator(CLIENT_SECRET_FILE, API_NAME, API_VERSION, SCOPES)
~~~

Apenas com essa parte já se consegue criar uma autenticação apenas executando o arquivo _SendEmail.py,_ porém como o objetivo é enviar um email com o usuario autenticado, continuarei mostrando o final do código.

Agora é necessário criar os campos para envio de email como, Titulo, Mensagem, Remetente e Destinatário.

~~~python
message = MIMEText('Python Mail test using API Google')
message['from'] = "your_email@gmail.com"
message['to'] = 'recipient@gmail.com'
message['subject'] = 'API Google'
raw_string = base64.urlsafe_b64encode(message.as_string())
~~~

Nesse momento é executado o envio do email passando o response da autenticação feita na linha 11, e os demais dados preenchidos, todas essas informações foram codificadas em base64 para facilitar transferência na Internet.

~~~python
try:
    message = service.users().messages().send(userId='me', body={'raw': raw_string}).execute()
    print ('Message Id: {}').format(message['id'])
except errors.HttpError as error:
     print ('An error occurred: {}').format(error)

~~~

### Código Completo:

~~~python
import base64
from email import errors
from GoogleAuthenticator import authenticator
from email.mime.text import MIMEText

CLIENT_SECRET_FILE = 'DocGoogle/credentials.json'
API_NAME = 'gmail'
API_VERSION = 'v1'
SCOPES = ['https://mail.google.com/']

service = authenticator(CLIENT_SECRET_FILE, API_NAME, API_VERSION, SCOPES)

message = MIMEText('Python Mail test using API Google')
message['from'] = "your_email@gmail.com"
message['to'] = 'recipient@gmail.com'
message['subject'] = 'API Google'
raw_string = base64.urlsafe_b64encode(message.as_string())

try:
    message = service.users().messages().send(userId='me', body={'raw': raw_string}).execute()
    print ('Message Id: {}').format(message['id'])
except errors.HttpError as error:
     print ('An error occurred: {}').format(error)').format(error)

~~~

### Step 5 – Execução

Executando o arquivo _SendEmail.py,_ abrirá uma página de seleção de usuário para autenticação.

![img1](/images/google-auth/img1.png)

Como meu aplicativo não foi publicado, irá aparecer uma tela de verificação para aceitar o acesso as informações confidenciais da conta do google que desejamos autenticar.

![img2](/images/google-auth/img2.png)

![img3](/images/google-auth/img3.png)

![img4](/images/google-auth/img4.png)

Após isso a conexão será autenticada e o email será enviado ao destinatário.

![img5](/images/google-auth/img5.png)

## Conclusão

Nesse artigo foi abordado uma técnica básica de autenticação de usuário do google para aplicações de terceiros. Com apenas as fardamentas disponibilizada como o Gmail API e sua biblioteca. No decorrer do desenvolvimento percebe-se a estrutura que foi utilizada, é fácil para ser implementada em qualquer aplicação, basta adaptá-la a regra de negócio.

Existem outros métodos de autenticação de usuário, e diferentes formas de desenvolver o código para a autenticação, basta saber fazer a procura certa no google que encontrará.

Espero que tenha gostado dessa publicação, e que possa usá-lo de alguma fora em seu dia a dia.

### Referências

[https://support.google.com/accounts/answer/6010255?p=less-secure-apps&amp;hl=pt-BR&amp;visit\_id=637824317526826001-2187211079&amp;rd=1](https://support.google.com/accounts/answer/6010255?p=less-secure-apps&amp;hl=pt-BR&amp;visit_id=637824317526826001-2187211079&amp;rd=1)).

[https://developers.google.com/workspace/guides/create-project](https://developers.google.com/workspace/guides/create-project)

[https://developers.google.com/workspace/guides/create-credentials](https://developers.google.com/workspace/guides/create-credentials)

[https://developers.google.com/gmail/api/quickstart/python](https://developers.google.com/gmail/api/quickstart/python)

[https://developers.google.com/gmail/api/guides/sending](https://developers.google.com/gmail/api/guides/sending)