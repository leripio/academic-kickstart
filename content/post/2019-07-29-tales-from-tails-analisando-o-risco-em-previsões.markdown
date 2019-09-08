---
title: 'Tales from tails: analisando o risco em previsões'
author: J. Renato
date: '2019-07-29'
slug: tales-from-tails-analisando-o-risco-em-previsões
categories:
  - Previsão
tags:
  - Previsão
  - risco
  - distribuição
  - risk measure
subtitle: ''
summary: ''
authors: []
lastmod: '2019-09-08T19:16:31-03:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---



[*No post anterior*](http://rleripio.com.br/post/combinando-modelos-de-previsao/), tratei de estratégias para combinar modelos de previsão a fim de obter melhores resultados. Melhor resultado, naquele contexto, significava apresentar menor **Root mean square forecast error (RMSFE)**.É muito comum -- tanto na literatura como na prática -- utilizar esta medida ou outras semelhantes que envolvam médias dos desvios quadráticos ou absolutos dos erros, por exemplo MSE, MSPE, MASE, MAPE, etc. **Sabe-se, contudo, que médias são muito sensíveis a valores extremos.** Portanto, um único valor extremo no conjunto de erros de previsão é capaz de elevar de maneira significativa aquelas estatísticas. Dito de outra forma, um modelo relativamente bom pode ser descartado porque apresentou uma única previsão ruim. Por esta razão, ganhou popularidade medidas que substituem a média pela mediana naquelas estatísticas anteriores. Os interessados em entender melhor as características destas medidas podem recorrer ao artigo de Hyndman e Koehler (2006): [*"Another look at measures of forecast accuracy"*](https://robjhyndman.com/papers/mase.pdf).

Neste ponto, eu gostaria de chamar atenção para uma coisa muito importante: **a estatística de acurácia utilizada para ranquear modelos é uma função-perda e, como tal, reflete o objetivo que se pretende alcançar.**Se o objetivo é reter o modelo que, em geral, não apresenta previsões muito distantes das realizações, as medidas consideradas até agora são razoáveis. Por outro lado, imagine que a previsão seja para o estoque de uma empresa ou para uma variável que define uma posição de investimento. **Nestes casos, o risco da previsão importa.**Em outras palavras, pode não fazer muito sentido escolher um modelo que, apesar de ter bom desempenho na média/mediana, apresenta maiores chances de valores extremos. No caso da empresa, tanto o excesso quanto a falta de estoque em níveis elevados pode comprometer as operações; igualmente catastrófico pode ser a realização muito abaixo/acima do previsto para uma variável-chave para o investidor.
Para usar um exemplo real, vamos considerar a mesma variável utilizada no post anterior: o núcleo do IPCA EX-3. A amostra vai de julho de 2006 a maio de 2019 e contém 155 observações. Os modelos utilizados foram ARIMA, ETS, CES e DOTM -- todos vistos naquela ocasião -- e os erros de previsão um passo à frente computados a partir de validação-cruzada com uma janela móvel de 60 observações -- o que totalizou cerca de 95 pontos de erro para cada modelo. As densidades dos erros de previsão para cada modelo são apresentadas abaixo:
<img src="2019-07-29-tales-from-tails-analisando-o-risco-em-previsões_files/figure-html/unnamed-chunk-1-1.png" width="672" />
De modo geral, todas as distribuições apresentam maior ocorrência em torno do zero, o que sugere que medidas que computam a tendência central devem ter desempenho mais ou menos parecido. Porém, cabe notar que a cauda das distribuições têm formatos bem diferentes: os erros do modelo ETS têm a cauda da direita maior que a do modelo DOTM e CES, por exemplo. É justamente neste aspecto que reside a ideia de risco: **a probabilidade de eventos extremos é maior ou menor de acordo com a área destas caudas**. Para traduzir isso de forma mais objetiva, o gráfico abaixo traz três medidas: duas de tendência central -- Root mean square forecast error (RMSFE) e Root median square forecast error (RMedSFE) -- e uma de risco: os limites da área com probabilidade de 10% à direita e à esquerda -- esta última em valor absoluto para ficar mais fácil de visualizar com as demais.

<img src="2019-07-29-tales-from-tails-analisando-o-risco-em-previsões_files/figure-html/unnamed-chunk-2-1.png" width="1152" />

O primeiro ponto a notar é que existe diferença na classificação quando consideramos a média ou a mediana. Pelo RMSFE, a escolha seria pelo DOTM, ao passo que pelo RMedSFE o modelo escolhido seria o CES. Por outro lado, se quiséssemos reduzir as chances de valores extremos -- tanto para cima quanto para baixo -- o DOTM seria a escolha inequívoca. Fica claro, portanto, a relevância de utilizar medidas aderentes aos objetivos da previsão. Adicionalmente, é sempre uma boa ideia comparar medidas alternativas para cada objetivo.

Por fim, uma questão interessante que se coloca é: **não é possível ter uma única medida capaz de sumarizar tanto a acurácia como o risco em um modelo de previsão?** A resposta parece ser positiva. No artigo [*Tales from tails: On the empirical distributions of forecasting errors and their implication to risk*](https://www.sciencedirect.com/science/article/pii/S0169207018301547), os autores propõem uma medida chamada Risk measure (RM). A ideia é relativamente simples: aplica-se uma transformação do tipo Box-Cox sobre a distribuição dos erros de previsão a fim de normalizá-los e em seguida calcula-se a medida que é a soma da média com o desvio-padrão da distribuição transformada. Ao fim, a transformação é revertida. A tabela abaixo computa uma versão da medida RM para os erros de previsão dos modelos (transformados via Box-Cox) em conjunto com o p-valor associado ao teste Shapiro de normalidade.


|modelo |  MAE|   SD|    RM| shapiro_bc|
|:------|----:|----:|-----:|----------:|
|dotm   | 0.94| 0.11|  2.03|       0.02|
|ces    | 1.31| 0.26|  2.79|       0.93|
|arima  | 1.41| 0.32|  3.09|       0.06|
|ets    | 3.02| 1.23| 13.48|       0.00|

O problema é que nem sempre a distribuição transformada é normal. O modelo DOTM apresentou a menor RM, porém os resultados não são significativos uma vez que somente a distribuição dos erros do modelo CES -- e com alguma "boa vontade" a do ARIMA -- se aproximou de uma distribuição normal. **De fato, em termos de p-valor, a distribuição original dos erros apresentou resultados melhores para o teste de normalidade: respectivamente 0.14, 0.15, 0.11 e 0.02.** Ainda assim, o ideal seria ter p-valores maiores para dar mais segurança.

Os códigos dos exercícios encontram-se disponíveis no [repositório do blog no github.](https://github.com/leripio/blog)

**Aviso legal**: Todo o conteúdo desta página é de responsabilidade pessoal do autor e não expressa a visão da instituição a qual o autor tem vínculo profissional.

