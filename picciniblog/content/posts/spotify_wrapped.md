---
author:
  name: "Patrick Piccini"
date: 2022-05-26T9:50:50-03:00
linktitle: Data Analytcs - Spotify Wrapped
type:
- post
- posts
title: Data Analytcs - Spotify Wrapped
weight: 10
---
![data-analytics](/images/spotify_wrapped/data-analytics.jpg)

## Table of Contents

- [Projeto Spotify Wrapped](#projeto-spotify-wrapped)
    - [O que é spotify wrapped](#o-que-%C3%A9-spotify-wrapped)
    - [DataSet](#dataset)
- [Exploratory Data Analysis EDA](#exploratory-data-analysis-eda)
- [Extração](#extra%C3%A7%C3%A3o)
- [Exploração](#explora%C3%A7%C3%A3o)
- [Limpeza de Dados](#limpeza-de-dados)
- [Agregação](#agrega%C3%A7%C3%A3o)
    - [10 artistas mais escutados.](#abaixo-observaremos-os-10-artistas-mais-escutados)
    - [10 musicas mais escutadas.](#abaixo-observaremos-as-10-musicas-mais-escutadas)
- [Visualização](#visualiza%C3%A7%C3%A3o)
- [Storytelling](#storytelling)
---

## Projeto Spotify Wrapped 

Recentemente venho estudando a área de Data Analytics, Data Science e as várias derivações de inteligência artificial, e descobri um grande interesse por esse mundo. Foi então que decidi iniciar um pequeno projeto onde eu possa praticar meus conhecimentos adquiridos, e também mostrar como o nosso cotidiano está repleto de tecnologia aplicada sobre inteligência artificial.

O que iremos aprender:
- Extração
- Exploração
- Limpeza
- Agregação
- Storytelling

### O que é spotify wrapped
Hoje, venho mostrar um projeto utilizando a plataforma de streaming de música mais utilizada no mundo, o Spotify. Iremos criar um recorço semelhante ao que o Spotify disponibiliza, chamada "Spotify Wrapped", que nada mais é do que uma retrospectiva das músicas, artistas entro outros, mais ouvidos durante o seu ano.

![wrapped](/images/spotify_wrapped/wrapped.png)

### DataSet
Nesse projeto irei utilizar o meu DataSet pessoal de meu perfil do Spotify. Para conseguir as suas informações pessoais, basta ir nas [Configurações de Privacidade](https://www.spotify.com/br/account/privacy/) da sua conta, e no final da página, seguir os passos para solicitação de seus dados. Dentro de alguns dias, você recebera por e-mail a confinação para baixas seus dados.

![steps](/images/spotify_wrapped/steps.png)


## Exploratory Data Analysis (EDA)
O que faremos hoje, será trabalha com um conceito primordial para o papel de um analista de dados, a análise Exploratória de dados(EAD).
- A EAD é um processo de analisar e resumir de forma detalhada um DataSet ou conjunto de dados. O objetivo é por várias técnicas, extrair informações precisas e claras, para a extração de _insights_.


Antes de começar a analisar, importaremos as bibliotecas que iremos utilizar, e também carregar nosso dataset.

Como padrão, o spotify disponibiliza todas as informações em formato JSON, como observamos no exemplo abaixo:

~~~json
{
  "endTime" : "2021-04-29 03:33",
  "artistName" : "Tame Impala",
  "trackName" : "Let It Happen",
  "msPlayed" : 345700
}
~~~

### Como índices temos:
-  endTime - Data e hora em que o fluxo terminou no formato UTC (Fuso Horário Universal Coordenado).
-  artistName - Nome do "criador" para cada fluxo (por exemplo, o nome do artista se for uma faixa de música).
-  trackName - Nome dos itens ouvidos ou assistidos (por exemplo, título da faixa de música ou nome do vídeo).
-  msPlayed - “msPlayed”- Representa quantos milissegundos a faixa foi ouvida pelo usuário.

## Extração

~~~ python
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt
from datetime import timedelta, datetime

dataframe0 = pd.read_json('./data-frame/StreamingHistory0.json')
dataframe1 = pd.read_json('./data-frame/StreamingHistory1.json')
dataframe2 = pd.read_json('./data-frame/StreamingHistory2.json')
print(f"dataframe0: {dataframe0.shape}\ndataframe1: {dataframe1.shape}\ndataframe2: {dataframe2.shape}")
~~~ 
~~~ipynb
dataframe0: (10000, 4)
dataframe1: (10000, 4)
dataframe2: (7154, 4)
~~~
Como quero fazer uma análise completa, irei concatenar dos dataframes, transformando-os apenas em um.

~~~ python
sptf = pd.concat([dataframe0,dataframe1,dataframe2], ignore_index=True)
print(f"Novo DataFrame: {sptf.shape}")
~~~ 
~~~ ipynb
Novo DataFrame: (27154, 4)
~~~ 
Como podemos ver, foi concatenado os 3 dataframes, somando um total de 27154 linhas, e 4 colunas.

O que faremos agora, é olhar para os dados que temos, e explorar essas informações que temos.
## Exploração
~~~ python
sptf.head()
~~~ 
![img1](/images/spotify_wrapped/img1.png)

~~~ python
sptf.columns
~~~ 
~~~iptnb
Index(['endTime', 'artistName', 'trackName', 'msPlayed'], dtype='object')
~~~
~~~ python
sptf.dtypes
~~~ 
~~~ ipynb
endTime       object
artistName    object
trackName     object
msPlayed       int64
dtype: object
~~~ 
Nas colunas que são _object_ podemos pressupor serem textos, e na coluna que é _int64_ um número inteiro.
- Estatísticas descritivas.

Para termos uma visão mais panorâmica do nosso DataSet podemos utilizar o comando describe(). Com ele conseguimos extrair as informações básicas como quantidade, média, minha e máxima, alguns percentuais e std. Esse comando visa, gerar estatísticas descritivas de nosso DataSet. Também pode ser aplicado individualmente com apenas uma coluna.

Como ha apenas o msPlayer como um valor numérico, todos esses cálculos serão feitos apenas pobre ele.

~~~ python
sptf.describe()
~~~ 

![img2](/images/spotify_wrapped/img2.png)

~~~ python
sptf.min()
~~~ 
~~~ ipynb
endTime                        2021-04-29 03:33
artistName                      #TeuFuturo Cast
trackName      Som de chuva - dormir, relaxar  
msPlayed                                      0
dtype: object
~~~ 

Como podemos notar acima, o valor mínimo de msPlayed é 0(Zero), então vamos verificar se essas informações estão corretas.

~~~ python
msPlayed_zero = sptf.query('msPlayed <= 0')
msPlayed_zero.head(10)
~~~ 
![img3](/images/spotify_wrapped/img3.png)

Após uma pequena análise, podemos notar que as músicas que tem o valor 0(zero), são musicas que não tiveram o seu tempo escutado contabilizado. Com isso, caso seja feita alguma análise mais complexa futuramente, esses valores podem ser considerados _outliers_.

O que são _outliers_:
São valores que os diferencia drasticamente de todos os outros, ou que fogem do padrão dos valores, podendo causar anomalias nos resultados de algoritmos mais complexos, como uma Regressão Linear.

Vejamos isso em um grafico scatter em um  diagrama de caixa:
~~~ python
plt.figure(figsize=(12, 6))
plt.scatter(sptf['msPlayed'], sptf['msPlayed'])
plt.xlabel('msPlayed')
print('scatter:')
plt.show()

sns.boxplot(y=sptf['msPlayed'])
print('boxplot:')
plt.show()
~~~ 

![graf1](/images/spotify_wrapped/graf1.png)
--
![graf2](/images/spotify_wrapped/graf2.png)

Verificando os gráficos, podemos notar uma coisa estranha. Existe um valor extremamente distante de todos os outros, um outlier. Porem como são os meus dados do Spotify, me questionei "Será que tem uma música muito a mais que todas as outras?". Foi aí que fui investigar, e descobri que havia realmente escutado toda essa quantia de horas.
### Mas esses outliers não vão desbalancear toda a minha análise? 

![img](https://i.pinimg.com/originals/f1/85/3a/f1853a77c203e8bb4e6615ae7b62d325.gif)

De certa forma sim, porem às vezes as informações que precisamos estão nesses outliers, e é o nosso caso.

Outra coisa que percebi, é que os dados que o spotify disponibilizou não estão 100% corretos. Pude ver músicas que já ouvido muitas vezes com o msPlayed ZERADO!

Vacilou heim Spotfy...

Mas tudo bem, deixamos passar dessa vez :D.
## Limpeza de Dados

Agora que entendemo os dados que iremos trabalhar, muitas vezes nos deparamos com informações inconsistentes. Então devemos executar uma série de limpeza ou alteração dessas informações.

- Para verificar as linas com NaN use:

Caso tenhamos valores valorem omissos como None, numpy.NaN, ou até mesmo Vazio, conseguimos identificá-los através do comando _notna()_, onde nos retorna a própria tabela, porem com os valores boleanos, representando se há ou não um valor NaN naquele índice.

- True  - Não tem NaN
- False - Tem NaN

Iremos calcular a quantia total de informações NaN, e como podemos ver, não ha nenhum valor NaN.

~~~ python
sptf.notna().value_counts()
~~~ 
~~~ ipynb
endTime  artistName  trackName  msPlayed
True     True        True       True        27154
dtype: int64
~~~ 

Como não temos nenhum dado NaN, o que podemos fazer é retirar todas as músicas que tenha o msPlayed menor ou igual a 0(zero).

~~~ python
sptf = sptf.query('msPlayed > 0')

# musica com menos tempo escutado
sptf.query('msPlayed <= 10')
~~~ 
![img4](/images/spotify_wrapped/img4.png)

## Agregação
Na parte de agregação, buscamos resumir os dados através de métricas e estatísticas como soma, media, etc. para extrair insigths. 
- Insigths - É a capacidade de tirar conclusões sobre os dados

O que precisaremos fazer é buscar as músicas mais ouvidas e também os artistas mais ouvidos. 

#### Abaixo observaremos os 10 artistas mais escutados.
~~~ python
artistas = sptf[['artistName','msPlayed']].groupby('artistName').agg('sum').reset_index()
artistas = artistas.sort_values(by='msPlayed' ,ascending=False)
artistas_top_10 = artistas[:10]
artistas_top_10
~~~ 

![img5](/images/spotify_wrapped/img5.png)

#### Abaixo observaremos as 10 musicas mais escutadas.

~~~ python
musicas = sptf.sort_values(by='msPlayed' ,ascending=False)
musicas_top_10 = musicas[:10]
musicas_top_10
~~~ 

![img6](/images/spotify_wrapped/img6.png)

Vamos também criar mais uma coluna como as posição do ranking.

~~~python
positions = ['1º','2º','3º','4º','5º','6º','7º','8º','9º','10º']

artistas_top_10['position'] = positions
musicas_top_10['position'] = positions
~~~

Com isso, já temos tudo que precisamos para fazer uma aplicação parecida com Spotify Wrapped.

## Visualização

Na etapa de visualização, é buscado criar gráficos que melhor representem os insights, gerados por agregação.

Para ficar um pouco mais fácil para pessoas leigas verem essas informações, iremos plotar um barplot.

~~~ python
chart = sns.barplot(data=artistas_top_10,  x='artistName',y='msPlayed')
chart.set_xticklabels(chart.get_xticklabels(), rotation=90)
chart.set(title='Top 10 Artistas')
artistas_top_10
~~~ 
![img7](/images/spotify_wrapped/img7.png)
--
![graf3](/images/spotify_wrapped/graf3.png)

~~~ python
chart = sns.barplot(data=musicas_top_10, x=positions,y='msPlayed')
chart.set_xticklabels(chart.get_xticklabels(), size=10)
chart.set(title='Top 10 Músicas')
musicas_top_10
~~~ 
![img8](/images/spotify_wrapped/img7.png)
--
![graf4](/images/spotify_wrapped/graf3.png)

## Storytelling
Na etapa de storytelling, buscamos organizar as conclusões que temos dos dados, através de um formato de história, para facilitar a transmissão do conhecimento.

Baseado nos dados disponibilizados pelo Spotify, foi extraído os 10 artistas mais estucados, e as 10 musicas mais escutadas no último ano, sendo que:
- 1 Podcast
- 1 Autio Relaxante
- 8 Músicas de diversos gêneros

Nota-se que o usuário tem um gosto musical muito diversificado. Também podemos supor que o usuário possa ter alguma dificuldade para dormir, visto que a segunda música mais ouvida é relacionada ao relaxamento profundo para dormir mais facilmente. Também pode-se perceber que o usuário gosta de conhecimentos gerais, visto que escuta um Podcast com o foco na diversidade.

Portanto, recomenda-se aumentar a relação do usurário com a plataforma, recomendando novos podcast, com o formato semelhante ao que já escuta. Novos sons relaxantes para manter a qualidade do sono ainda melhor, e o anúncio de novos lançamentos do artista "Pineapple StormTv", sendo que ocupa duas posições de músicas mais ouvidas e está no Top 3 artistas mais escutados.

