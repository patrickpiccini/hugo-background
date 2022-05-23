---
author:
  name: "Patrick Piccini"
date: 2022-05-22T18:00:00-03:00
linktitle: Email com Google Authentication – Refresh Token
type:
- post
- posts
title: Email com Google Authentication – Refresh Token
weight: 10
---

![refresh_token](/images/google-auth/refresh_token.png)

## Table of Contents
- [Table of Contents](#table-of-contents)
- [Introdução](#introdu%C3%A7%C3%A3o)
    - [Pré-requisitos](#pr%C3%A9-requisitos)
- [Step 1 – Instalação de bibliotecas](#step-1--instala%C3%A7%C3%A3o-de-bibliotecas)
- [Step 2 – Leitura do token.json](#step-2--leitura-do-tokenjson)
- [Step 3 – Função refresh\_token](#step-3--fun%C3%A7%C3%A3o-refresh%5C_token)
- [Step 4 – Função request\_refresh\_token](#step-4--fun%C3%A7%C3%A3o-request%5C_refresh%5C_token)
- [Step 5 – Execução](#step-5--execu%C3%A7%C3%A3o)
- [Código Completo](#c%C3%B3digo-completo)
- [Conclusão](#conclus%C3%A3o)
- [Referências](#refer%C3%AAncias)
---

## Introdução 


Nesse artigo irei continuar o assunto abordado na publicação [Email com Google Authentication](https://patrickpiccini.github.io/posts/google-auth/).

Visto que no primeiro estágio fizemos apenas uma autenticação, nos deparamos com a criação de um arquivo chamado _token.json._ Nele, está contido uma sequência de informações, sendo elas um token de autenticação, algumas informações do usuário autenticado, e uma data de expiração. Então, quando o usuário utilizar a aplicação que criamos, ele não utilizará mais o arquivo _credentias.json_, mas sim as informações contidas no _token.json._

As informações contidas do _token.json_ serão semelhantes aos dados abaixo.

~~~json
{
    "scopes": ["https://mail.google.com/"],
    "token_uri": "https://oauth2.googleapis.com/token",
    "expiry": "2022-05-21T00:49:13.081000Z",
    "token": "ya29.a0ARrdaM_Egt-trkKacPEcWEzCC9Lejs7DTo8VnPYRu",
    "client_id": "725031891889-0533ns1pin5753k.apps.googleusercontent.com",
    "client_secret": "GOCSPX-VlvdZsYe-0GXpkctKmK",
    "refresh_token": "1//0h0y8XEIbzPbiCgYIARAAGBESNwF-C7yWA9JaFT_fACew"
}
~~~

Como podemos notar, nesse token que nos é retornado há uma data de expiração. Caso o usuário fique enviando vários e-mails durante o dia, terá que passar pela tela de autenticação inumeras vezes, sendo que a Google disponibiliza apenas 4 horas de validade para cada token.

Visto isso, utilizaremos uma informação contida no _token.json_ chamada refresh\_token. Com ela, conseguimos criar um novo token de acesso, sem que o usuário precise passar pela tela do browser.

### Pré-requisitos

- [Python](https://www.python.org/downloads/) 2.6 ou superior;

- Gerenciamento de pacotes [PIP](https://pypi.org/project/pip/);

- Ter criado o código baseado no primeiro artigo publicado ([Email com Google Authentication](https://patrickpiccini.github.io/posts/google-auth/)).

### Step 1 – Instalação de bibliotecas

~~~ bash
pip install --upgrade jsonlib DateTime requests
~~~

### Step 2 – Leitura do token.json

Para entender quando iremos criar um novo token de acesso, nos basear pela data de expiração contida no arquivo. Será necessário apontarmos uma variável chamada _date\_time\_now_, que irá conter a data/hora atual. Após isso, iremos ler esse _token.json,_ e verificar se a data/hora atual é maior que a data de expiração. Caso sejam, invocaremos uma função chamada de _refresh\_token._

Todas as informações lidas do arquivo _token.json_ serão atribuída à variável _Info\_json,_ que passaremos como parâmetro para a nova função criada.

~~~python
    if os.path.exists('token.json'):
        with open('token.json', 'r') as verify:
            info_json = json.load(verify)

            if date_time_now > info_json['expiry']:
                refresh_token(info_json)

        creds = Credentials.from_authorized_user_file(
            'token.json', SCOPES)
~~~

### Step 3 – Função refresh\_token

A função refresh\_token será responsável por requisitar à API do Google um novo token de acesso. Dentro dessa função será feita toda a manipulação de requisição de um novo token, adição de horas de expiração, e escrita das novas informações dentro do arquivo já existente _token.json._

Primeiramente precisaremos criar o corpo da requisição, seguindo alguns padrões exigidos.

~~~python
def refresh_token(info_json):
    try:
        refresh_token_obj = {
            "client_id": str(info_json["client_id"]).replace("u'", "'"),
            "client_secret": str(info_json["client_secret"]).replace("u'", "'"),
            "refresh_token": str(info_json["refresh_token"]).replace("u'", "'"), 
            "grant_type": "refresh_token"
        }
~~~

Logo abaixo, vamos reservar uma variável chamada _refresh\_credentials_. para ela, posteriormente atribuiremos uma nova função nomeada de _request\_refresh\_token_, passando como parâmetro as informações que criamos na variável _refresh\_token\_obj._

Após isso, a função _request\_refresh\_token_ retornará um response. Iremos carregar as informações em formato json na variável _refresh\_toke\_obj._ Separaremos mais duas informações, uma variável contendo a soma do horário atual + 4 horas, visto que o token é válido por quatro horas, e também o token de acesso retornado da requisição _refresh\_credentials._

~~~ python
   refresh_credentials = request_refresh_token(refresh_token_obj)

        refresh_toke_obj = json.loads(refresh_credentials.text)
        expiry_time_refresh_token = datetime.now() + timedelta(hours=4)
        access_token = refresh_toke_obj['access_token']
~~~

Por fim, uma exception da função caso ocorra falha em alguma dessas informações que manipulamos.

~~~ python
    except Exception as error:
        print('Erro criacao de refresh_token.\n{}'.format(str(error)))
~~~

### Step 4 – Função request\_refresh\_token

O que abordaremos agora será a função itada anteriormente, _a request\_refresh\_token._ Nela, iremos apenas fazer uma requisição post para a url [https://oauth2.googleapis.com/token](https://oauth2.googleapis.com/token), passando as informações que montamos na variável _refresh\_token\_obj_ anteriormente.

~~~python
def request_refresh_token(refresh_token_obj):
    try:
        return requests.post('https://oauth2.googleapis.com/token', data=refresh_token_obj)
~~~

Assim, finalizaremos com uma sequência de possíveis exceções que podem ocorrem na requisição.

~~~python
    except requests.exceptions.Timeout as e:
        print('Request Timeout exception:\n{}'.format(v))
        return
    except requests.exceptions.TooManyRedirects as e:
        print('Request too many redirects exception:\n{}'.format(str(e)))
        return
    except requests.exceptions.RequestException as e:
        print('Request exception:\n{}'.format(str(e)))
        return
~~~

### Step 5 – Execução

O funcionamento será o mesmo mostrado no artigo [Email com Google Authentication](https://patrickpiccini.github.io/posts/google-auth/).
Executando o arquivo _SendEmail.py,_ abrirá uma página de seleção de usuário para autenticação.

![img1](/images/google-auth/img1.png)

Como meu aplicativo não foi publicado, irá aparecer uma tela de verificação para aceitar o acesso as informações confidenciais da conta do google que desejamos autenticar.

![img2](/images/google-auth/img2.png)

![img3](/images/google-auth/img3.png)

![img4](/images/google-auth/img4.png)

Após isso, a conexão será autenticada e o email será enviado ao destinatário.

![img5](/images/google-auth/img5.png)

Depois de o usuário passar por esse estágio de autenticação, não precisará mais refazer todos esses passos, visto que a atualização que fizemos no código já irá gerar novos tokens automaticamente.

### Código Completo

[Code - Email com Google Authentication – Refresh Token](https://github.com/patrickpiccini/email-google-auth-2)

## Conclusão

Nesse artigo abordamos a segunda etapa para aplicação de autenticação utilizando Gmail API. Foi mostrado como verificar a validade do token de acesso através da data de expiração. Caso esse token esteja expirado, criamos duas novas funções chamadas de _refresh\_token_ e _request\_refresh\_token_, responsáveis por requisitar um novo token de acesso, e inseri-lo no arquivo _token.json_ com uma nova data de expiração. Assim, o usuário não precisará ficar passando pelas telas de Login com o Google diversas vezes ao dia. Tudo isso a nova atualização no código fará em back-end para o usuário.

Reforçando o que citei no artigo anterior: Existem outros métodos de autenticação de usuário, e diferentes formas de desenvolver o código para a autenticação, basta saber fazer a procura certa no google que encontrará.

Espero que tenha gostado dessa publicação, e que possa usá-la de alguma forma em seu dia a dia.

### Referências

[Apps menos seguros e a Conta do Google](https://support.google.com/accounts/answer/6010255?p=less-secure-apps&amp;hl=pt-BR&amp;visit_id=637824317526826001-2187211079&amp;rd=1));

[Create a Google Cloud project](https://developers.google.com/workspace/guides/create-project);

[Create access credentials](https://developers.google.com/workspace/guides/create-credentials);

[Python Quickstart](https://developers.google.com/gmail/api/quickstart/python);

[Sending Email](https://developers.google.com/gmail/api/guides/sending).