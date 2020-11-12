---
title: Funções-perda alternativas para seleção de modelos de previsão
author: Renato Leripio
date: '2020-11-10'
slug: funções-perda-alternativas-para-seleção-de-modelos-de-previsão
categories:
  - Previsão
tags:
  - previsão
  - loss
  - seleção
  - model selection
subtitle: ''
summary: ''
authors: []
lastmod: '2020-11-10T22:52:11-03:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
markup: mmark
---

Em um [post passado](https://www.rleripio.com.br/post/tales-from-tails-analisando-o-risco-em-previsoes/), abordei uma métrica alternativa para medir a acurácia de modelos de previsão. A *risk measure* buscava eliminar a possibilidade de escolher modelos com chances de entregar valores extremos. A ideia aqui é ser um tanto mais geral.

O problema de escolher um modelo de previsão geralmente consiste em minimizar alguma métrica, sendo comum o erro absoluto médio (MAE) ou a raiz do erro quadrado médio (RMSE). Essas métricas funcionam como *funções-perda*, isto é, elas expressam um objetivo. Entretanto, muitas vezes não traduzimos de forma ampla nosso objetivo ao escolher um modelo de previsão.

Por exemplo, MAE e RMSE são funções simétricas. Além disso, ambas usam média simples para sumarizar os erros de projeção. Quando utilizamos alguma delas para avaliar qual modelo é melhor, no fundo estamos dizendo: 

> *"Eu não me importo com o sinal do erro -- 2 unidades para cima ou para baixo impactam igualmente o meu resultado --, bem como não faz diferença se os maiores erros foram cometidos recentemente ou em período mais distante."*

Seguramente essas condições não se aplicam a todos os negócios. Alguém interessado em prever demanda pode preferir errar para mais a errar para menos. Também é possível que prefira modelos cujas projeções tenham ficado mais próximas dos valores realizados nos últimos 6 meses em comparação aos primeiros 6 meses do *backtest*.

Em suma, em muitas situações é preciso definir quais condições o modelo deve atender e isso envolve criar uma **função**. Essa função deve sumarizar os erros das projeções de modo a representar nosso **objetivo**.

Considere a função abaixo em que $ x $ é o erro, isto é, a diferença entre o valor previsto e o realizado:

$$ \begin{equation}
f(x) = 
\begin{cases} 
\ \frac{x}{2} & \text{se } x \geq 0 \\
\ 2|x|         & \text{se } x < 0
\end{cases}`
\end{equation} $$

Em palavras, estamos dizendo que um erro negativo será penalizado duas vezes mais que no MAE convencional e um erro positivo duas vezes menos. Adicionalmente, podemos definir que erros mais recentes ganhem maior peso. Isso pode ser feito computando uma média ponderada em que os pesos são a posição da observação no vetor de erros. Abaixo um exemplo de como declarar esta função:




```r
alternative_loss <- function(x){
  
y <- ifelse(x >= 0, x/2, 2*abs(x))

w <- seq(from = 1, to = length(y), by = 1)

w <- w/sum(w)

output <- weighted.mean(y, w)

return(output)
  
}
```

Podemos visualizar $ f(x) $ em forma de gráfico para ganhar intuição. 

<img src="/post/2020-11-10-funções-perda-alternativas-para-seleção-de-modelos-de-previsão_files/figure-html/plot_loss-1.png" width="672" />





Na sequência, ajustamos um conjunto de modelos aos dados de vendas mensais de vinhos na Austrália -- disponível no pacote `forecast` -- e computamos os erros de projeção fora da amostra para as últimas 12 observações. Abaixo segue a classificação a partir do MAE convencional:




```
## # A tibble: 3 x 3
##   .method .metric .estimate
##   <chr>   <chr>       <dbl>
## 1 ets     mae         2205.
## 2 tbats   mae         2296.
## 3 snaive  mae         2334.
```

Na sequência, podemos utilizar nossa medida alternativa para classificar os modelos. Basicamente, estamos pegando o mesmo vetor de erros e aplicando a nova função. Os resultados são:


```
## # A tibble: 3 x 3
##   .method .metric     .estimate
##   <chr>   <chr>           <dbl>
## 1 snaive  alternative     2489.
## 2 tbats   alternative     2824.
## 3 ets     alternative     2876.
```

Os valores não são diretamente comparáveis, o que importa é a ordenação. Considerando o objetivo mais amplo, o modelo `snaive` obteve melhor performance em relação ao `ets`. 

Evidentemente, encontramos diversas características desejáveis em métricas como MAE ou RMSE e a ideia deste post não é excluí-las do jogo -- tampouco colocar a métrica adotada no exemplo como referência. A mensagem aqui é que, à medida que ganha relevância o processo de selecionar modelos, deve-se ter em mente que objetivo pretende-se alcançar e como traduzi-lo em uma métrica adequada. Neste aspecto, conhecer formas funcionais é fundamental.

**Aviso legal**: Todo o conteúdo desta página é de responsabilidade pessoal do autor e não expressa a visão da instituição a qual o autor tem vínculo profissional.
