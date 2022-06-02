---
author:
  name: "Patrick Piccini"
date: 2025-05-26T9:01:59-03:00
linktitle: Microservice Project – Step 9
type:
- post
- posts
title: Microservice Project – Step 9
weight: 10
---
## Table of Contents
- [Argon2](#argon2)
- [Hash de Senha](#hash-de-senha)
    - [Encapsulamento](#encapsulamento)
- [Controlador DataBase](#controlador-database)
- [Resumo](#resumo)
---

Nesse step iremos abordar um assunto muito importante, onde devemos sempre reservar um tempo para desenvolvimento, que é a criptografia de informações sensíveis. O que faremos será uma classe para criptografar a senha do usuário antes de salvar no banco de dados.

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
    │
    ├───criptografy
    │       hash_password.py
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

## Argon2

Para o hash de senhas, utilizaremos um algoritmo campeão da competição de Hashing de Senha em julho de 2015. Argon2 é um algoritmo de hash de senha seguro. Ele foi projetado para ter um tempo de execução configurável e consumo de memória. Isso significa que você pode decidir quanto tempo leva para fazer o hash de uma senha e quanta memória é necessária.

[argon2: [https://argon2-cffi.readthedocs.io/en/stable/argon2.html](https://argon2-cffi.readthedocs.io/en/stable/argon2.html)]

## Hash de Senha

O tema de criptografia de senha pode ser muitas vezes assustador, contudo, nos dias atuais temos diferentes algoritmos e bibliotecas que podem nos auxiliar a aplicar uma complexidade maior no nível de segurança das informações. Por isso, faremos uma implementação simples usando a biblioteca **argon2** e iremos usar um conceito básico de O.O(Orientação a Objetos), que será o encapsulamento de dados.

### Encapsulamento

O encapsulamento consiste em evitar que esses dados sofram modificações ou acessos indevidos. Para isso, é criada uma estrutura que contém métodos chamados _**getters**_ e _**setters**_, que poderão ser utilizados em qualquer outra classe, sem causar inconsistência na criação do código.

_**Getter**_ – O método getter tem como objetivo retornar o valor que foi solicitado, porém, de forma que não prejudique a exatidão do dado.

_**Setter**_ – O método setter, recebe um valor como atributo de qualquer tipo suportado pela linguagem, podendo assim, acessar o dado bloqueado e fazer sua devida modificação.

Agora que temos conhecimento do que iremos utilizar, vamos para o desenvolvimento.

Primeiramente importaremos a biblioteca para fazer o hash das senhas

~~~ python
from argon2 import PasswordHasher
~~~

Vamos criar uma classe chamada **EncriptPassword** e em seu constructor \_\_init\_\_ iremos instanciar a biblioteca **PasswordHasher.** Tambémcriaremos os atributo _self.\_\_password_ e _self.\_\_hash\_password_ sendo privados, utilizando o &quot;\_\_&quot;_._ É nesses atributos que serão encapsulados.

~~~ python
    def __init__(self, password):
        self.ph = PasswordHasher()
        self.__password = ''
        self.__hash_password = ''
~~~

Agora criaremos os construtores getters e setters da senha e do hash da senha.

~~~ python
    #password
    def get_pass(self):
        return self.__password
    
    def set_pass(self, password_imput):
        self.__password = password_imput

    #hash password
    def get_hash_pass(self):
        return self.__hash_password
    
    def set_hash_pass(self, hash_pass):
        self.__hash_password = hash_pass
~~~

Nesse momento, criaremos o método que fará o hash da senha. Para isso usaremos a função _hash_ e passaremos o _self.\_\_password_ como atributo. Essa senha será hasheada e atribuída ao atributo privado _\_\_hash\_password._ Assim, classes externas não poderão alterar a senha já criptografada.

~~~ python
    def hash_password(self):
        self.__hash_password = self.ph.hash(self.__password)
~~~

Criaremos também um método que será responsável por comparar se a senha hasheada é equivalente à senha normal. Esse método será muito importante para troca de senha, edição das informações do usuário e deletar usuário, onde será verificado se a senha postada na requisição é compatível com a senha existente no banco de dados.

~~~ python
   def verify_hash(self):
        try:
            return self.ph.verify(self.__hash_password, self.__password)
        except Exception as error:
            print('Erro in verify hash',error)
            return False
~~~

## Controlador DataBase

Agora que temos nossa classe para criptografar as senhas, precisaremos criar alguns métodos a mais em nosso _postgres\_worker_.

Devemos lembrar de importar a classe **EncriptPassword** ¸ ao arquivo_postgres\_worker.py.

~~~ python
from config.database_connection import ConnectionDatabase
from criptografy.hash_password import EncriptPassword
from datetime import datetime
~~~

O primeiro método é o _verify\_password\_database(self, db\_password, old\_pass)._ Ele será o responsável por receber como parâmetro a senha já criptografada, e a senha normal em formato string. Como resultado, nos retornará um valor booleano que será o que nos autorizará a continuar com algumas funções do sistema como: alterar a senha, alterar as informações do usuário e deletar um usuário. Caso a senha seja incompatível com a que há no banco de dados, a modificação será negada, e o usuário receberá um aviso como response.

~~~ python
    def verify_password_database(self, db_password, old_pass):
        try:
            EP = EncriptPassword()
            EP.set_pass(old_pass)
            EP.set_hash_pass(db_password[0])
            response_verify = EP.verify_hash()

            if response_verify is True:
                return True
            return False
        except Exception as error:
            print(error)
~~~

Também será criado um método para criptografar a senha, onde novamente utilizaremos as funções da classe **EncriptPassword.** O retorno desse método deverá ser a senha já hasheada, visto que será ela que colocaremos no banco de dados.

~~~ python
    def encript_password(self, data):
        try:
            HS = EncriptPassword()
            HS.set_pass(data)
            HS.hash_password()
            return HS.get_hash_pass()
        except Exception as error:
            print(error)
            return f'[X] ERROR ON INCRIPTED PASSWORD! \
        {error}'

~~~

Como exemplo de onde usaríamos esses novos métodos, deixarei abaixo um exemplo de uma criação de usuário, em que precisaremos colocar a senha. E a função de deletar um usuário, onde só será deletado caso a senha seja a mesma contida no banco de dados.

~~~ python
    def insert_user(self, data):
            try:
                encripted_password = self.encript_password(data['password'])

                query_insert = 'INSERT INTO users (full_name, nick_name ,password ,cpf , email, phone_number, created_at, updated_at)VALUES (%s,%s,%s,%s,%s,%s,%s,%s)'
                vars_query = (data['name'], data['nick_name'], encripted_password, data['cpf'],
                              data['email'], data['phone_number'], self.date_time_formate, self.date_time_formate)
                self.PSQL.cursor.execute(query_insert, vars_query)
                self.PSQL.connection.commit()

                print('[✓] INSERTION DONE IN POSTGRES!')
                return '[✓] User created successfully! '
            except Exception as error:
                print(error)
                return f'[X] ERROR INSERTING IN POSTGRES! {error}'
            finally:
                self.PSQL.cursor.close()
~~~

~~~ python
    def delete_user(self, data):
            try:
                result_psw = self.take_pass(data['nick_name'])
                verify = self.verify_password_database(
                    result_psw, data['password'])

                if verify == False:
                    return f'[X] PASSWORD NOT EQUAL!'

                sql_delete_query = 'DELETE FROM users WHERE nick_name=%s'
                vars_query_select = data['nick_name']
                self.PSQL.cursor.execute(sql_delete_query, (vars_query_select,))
                row_count = self.PSQL.cursor.rowcount
                self.PSQL.connection.commit()

                print('[✓] DELETE DONE SUCCESSFULLY IN POSTGRES!')
                return {'Altered Lines': row_count}
            except Exception as error:
                print(error)
                return f'[X] ERROR ON DELETE IN POSTGRES! {error}'
            finally:
                self.PSQL.cursor.close()
~~~

OBS: lembrando novamente que todas as funções de interação com o banco de dados estão disponíveis no repositório git do projeto.

## Resumo

Nesse passo abordamos assuntos muito importantes como a criptografia da senha do usuário e um ramo da Programação Orientada a Objetos (POO), chamada de encapsulamento. Criamos os métodos de criptografia de senha usando o algoritmo argon2. Por fim, implementamos novas funções no _postgres\_worker.py,_ que serão responsáveis por verificar a senha contida no banco, e também criptografar a senha para salvar no banco de dados.

Aqui, terminamos o desenvolvimento do primeiro Microservice (MS1). Na próxima etapa do projeto, iremos desenvolver o segundo Microservice (MS2), que será responsável por criar as orders no banco de dados.