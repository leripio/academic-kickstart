---
title: Como obter rolling windows de maneira rápida e eficiente
author: Renato Leripio
date: '2020-08-10'
slug: como-obter-rolling-windows-de-maneira-rápida-e-eficiente
categories:
  - Data Science
tags:
  - rolling window
  - rolling
  - rstats
subtitle: ''
summary: ''
authors: []
lastmod: '2020-08-10T17:30:10-03:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---

No post [anterior](https://www.rleripio.com.br/post/como-generalizar-acoes-de-maneira-eficiente/), apresentei uma forma bastante conveniente de generalizar ações dispensando *for-loops* tradicionais. A abordagem permite eliminar indexações trabalhosas e organizar os objetos de maneira bem prática.

Outra tarefa muito presente no dia-a-dia de quem trabalha com dados é obter versões em janela móvel para alguma função. O exemplo mais simples seria a média móvel, a qual conta com implementações eficientes em diversos pacotes -- eu recomendo o `RcppRoll` que, além da média móvel, também computa outras estatísticas.

Entretanto, na maioria dos casos não existe uma versão *rolling* da função desejada -- sobretudo para aquelas criadas por nós para atender a alguma demanda específica. É possível obter resultados *rolling* aplicando a função desejada sobre subconjuntos dos dados e, novamente, o post [anterior](https://www.rleripio.com.br/post/como-generalizar-acoes-de-maneira-eficiente/) pode ajudar nisso -- pense em cada subconjunto como um elemento de uma lista.

Porém, existe uma forma ainda mais prática de fazer isso através da função `rollify` do pacote `tibbletime`. A ideia da função é criar versões *rolling* da função desejada e a integração com `tibble`'s torna possível retornar as saídas de maneira organizada.

A sintaxe é bem simples e, para ilustrar, vamos computar uma *rolling regression* -- isto é, obter o coeficiente de uma regressão em janela móvel. Isso é bem útil nas aplicações em que o coeficiente de interesse varia no tempo. No exemplo, vou retornar o coeficiente autoregressivo de um modelo **SARIMA(1,0,0)(1,0,0)[12]** ajustado à série histórica do IPCA como forma de medir a inércia da inflação ao longo do tempo.

### Passo 1: carregar pacotes e importar os dados

Para baixar a série do IPCA dos preços livres, vou utilizar o pacote `rbcb` do Wilson Freitas. Este pacote não está mais disponível no CRAN, mas pode ser instalado a partir do [GitHub](https://github.com/wilsonfreitas/rbcb) do autor através do seguinte comando: 


```r
devtools::install_github('wilsonfreitas/rbcb')
```


```r
library(rbcb)
library(tidyverse)
library(tibbletime)

ipca_livres <- rbcb::get_series(code = list("ipca" = 11428))
```

### Passo 2: criar a função que ajusta o modelo e retorna o coeficiente AR e seu erro padrão


```r
get_ar1 <- function(x){
  
x_ts <- ts(x, frequency = 12)  
  
fit_sar1 <- arima(x_ts, order = c(1,0,0), seasonal = list(order = c(1,0,0)))

out <- list("beta" = fit_sar1[["coef"]]["ar1"],
            "se"   = fit_sar1[["var.coef"]]["ar1", "ar1"] %>% sqrt())

}
```

### Passo 3: criar a versão *rolling* da função

O argumento `window` controla o tamanho da janela. O argumento `unlist = FALSE` garante que os resultados ficarão no formato de lista. Isso é necessário aqui porque nosso *output* é uma lista com dois objetos: o coeficiente e seu erro padrão.


```r
roll_get_ar1 <- tibbletime::rollify(.f = get_ar1, window = 60, unlist = FALSE)
```

Na sequência, usamos a função criada acima para gerar um `tibble` contendo uma coluna com o coeficiente AR(1) e outra com o erro-padrão. O `map` é utilizado para extrair os elementos da lista. 


```r
inercia <- ipca_livres %>%
  
  dplyr::mutate(out = roll_get_ar1(ipca)) %>%
  
  dplyr::filter(!is.na(out)) %>%
  
  dplyr::mutate(ar1 = purrr::map_dbl(.x = out, .f = function(x) x$beta),
                se  = purrr::map_dbl(.x = out, .f = function(x) x$se))
```

Por fim, podemos gerar um gráfico como este abaixo contendo a inércia estimada ao longo do tempo, bem como a significância do coeficiente $ \pm 1.96 \times \text{se} $. Também adicionamos uma curva suavizada (loess) para deixar a trajetória mais comportada.

<img src="/post/2020-08-10-como-obter-rolling-windows-de-maneira-rápida-e-eficiente_files/figure-html/plot-1.png" width="672" />

Vale ressaltar que o modelo SARIMA é bem simples e não controla os efeitos de diversos fatores sobre a inflação -- atividade, câmbio, etc. Portanto, para medir adequadamente o grau de inércia precisaríamos adicionar mais estrutura à especificação. 

**Aviso legal**: Todo o conteúdo desta página é de responsabilidade pessoal do autor e não expressa a visão da instituição a qual o autor tem vínculo profissional.
