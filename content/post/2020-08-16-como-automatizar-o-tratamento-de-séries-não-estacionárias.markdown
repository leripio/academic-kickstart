---
title: Como automatizar o tratamento de séries não-estacionárias
author: Renato Leripio
date: '2020-08-16'
slug: como-automatizar-o-tratamento-de-séries-não-estacionárias
categories:
  - Data Science
tags:
  - unit roots
  - time series
  - séries temporais
  - raiz unitaria
subtitle: ''
summary: ''
authors: []
lastmod: '2020-08-16T19:29:27-03:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---

Alguns métodos para séries temporais exigem que as séries sejam estacionárias. Além das questões estatísticas associadas aos estimadores, regredir séries integradas coloca em dúvida a validade do modelo -- o problema da regressão espúria. 

Um procedimento padrão ao trabalhar com séries temporais é, portanto, checar se as séries são estacionárias e, caso não sejam, aplicar algum tratamento -- geralmente isso envolve tomar a primeira diferença da série.

Apesar de simples, este procedimento envolve certo trabalho. Em particular quando temos um conjunto grande de variáveis, não parece razoável repetir o processo para cada uma delas. 

A maneira mais geral de automatizar esta tarefa é criar uma função que aplica os testes de raiz unitária sobre a série, compara a estatística com o valor crítico do teste e diferencia a série caso exista evidência de não-estacionaridade. 

O pacote `forecast` resolveu boa parte desse fluxo com a função `ndiffs`. A função aplica os testes mais comuns -- ADF, PP e KPSS -- e retorna o número de vezes que a série precisa ser diferenciada. Nosso trabalho se reduz, portanto, a reter esta informação e usá-la para computar as diferenças. 

Vamos a um exemplo de como fazer isso a partir de um conjunto simulado de séries temporais estacionárias e não-estacionárias.

### Passo 1: Carregar pacotes e simular os dados


```r
# 1. Carregar pacotes

library(tidyverse)
library(forecast)
library(timetk)

# 2. Simular séries

set.seed(123)

mu <- list(-2, 0, 2, 4, 6) # média das séries

sigma <- list(1, 2, 3, 4, 5) # variância das séries

series <- purrr::map2_dfc(mu, sigma, rnorm, n = 100) %>% 
  
  dplyr::mutate(date = seq.Date(from = as.Date("2010-01-01"), 
                                length.out = 100, by = "month")) %>%
  
  dplyr::mutate_at(vars(-date), ~ timetk::tk_ts(.)) %>%
  
  magrittr::set_colnames(c(paste0("Serie", seq_along(mu)), "date"))

# 3. Tornar as séries Serie1 e Serie3 não-estacionárias

series_aux <- series %>% 
  
  dplyr::mutate_at(vars(Serie1, Serie3), ~ cumsum(.))
```

<img src="/post/2020-08-16-como-automatizar-o-tratamento-de-séries-não-estacionárias_files/figure-html/graf-1.png" width="672" />

### Passo 2: checar o número de diferenças necessárias

Aqui vamos utilizar a função `ndiffs` para determinar o número de diferenças necessárias para tornar a série estacionária, porém com alguns avanços. Primeiro, vamos utilizar os três testes disponíveis: ADF, PP e KPSS. 

Em seguida, vamos criar uma regra para definir se a série será ou não diferenciada a partir dos resultados anteriores. Como esse é um exercício simulado, sabemos que as séries não-estacionárias -- `Serie1` e `Serie3` -- são I(1), isto é, só precisam de uma diferença. 

Portanto, os testes devem retornar 1 para `Serie1` e `Serie3` e 0 para as demais. Assim, nossa regra será a seguinte: se a soma dos resultados for superior a 2, ou seja, mais de um teste retornar necessidade de uma diferença, esta série deverá ser diferenciada. Vamos usar uma coluna `Diferenciar` = "SIM" neste caso e "NAO" para o caso contrário.

A ressalva aqui é que esta é uma regra bem simples para atender ao nosso exercício. Os interessados podem adequá-la aos seus próprios objetivos.


```r
testes <- list(kpss = "kpss", pp = "pp", adf = "adf")

ndiffs_testes <- function(x) purrr::map(testes, function(y){
  
  forecast::ndiffs(x, alpha = 0.05, y)
  
  })

series_ndiffs <- series_aux %>%
  
  dplyr::select(-date) %>%
  
  purrr::map(.f = ndiffs_testes) %>%
  
  plyr::ldply(bind_rows) %>%
  
  dplyr::mutate(Diferenciar = ifelse(kpss + adf + pp >= 2, "SIM", "NAO"))


series_ndiffs
```

```
##      .id kpss pp adf Diferenciar
## 1 Serie1    1  1   1         SIM
## 2 Serie2    0  0   0         NAO
## 3 Serie3    1  1   1         SIM
## 4 Serie4    0  0   0         NAO
## 5 Serie5    0  0   0         NAO
```

### Passo 4: Diferenciar as séries não-estacionárias

O último passo consiste em diferenciar aquelas séries consideradas não-estacionárias que receberam "SIM" na etapa anterior.


```r
# Reter o id das séries não-estacionárias

series_labs <- series_ndiffs %>% 
  
  dplyr::filter(Diferenciar == "SIM") %>% 
  
  dplyr::pull(.id)

# Diferenciar 

series_dif <- series_aux %>% 
  
  dplyr::mutate_at(vars(all_of(series_labs)), ~ . - dplyr::lag(., 1))

head(series_dif, 5)
```

```
## # A tibble: 5 x 6
##   Serie1 Serie2 Serie3 Serie4 Serie5 date      
##    <dbl>  <dbl>  <dbl>  <dbl>  <dbl> <date>    
## 1 NA     -1.42  NA      1.14   5.63  2010-01-01
## 2 -2.23   0.514  5.94   0.989  0.157 2010-02-01
## 3 -0.441 -0.493  1.20   0.246  2.83  2010-03-01
## 4 -1.93  -0.695  3.63  -0.210  5.86  2010-04-01
## 5 -1.87  -1.90   0.757  2.25   9.35  2010-05-01
```

Por fim, o pacote `forecast` também dispõe da função `nsdiffs` que realiza o mesmo procedimento para séries que necessitam de diferenças sazonais.

**Aviso legal**: Todo o conteúdo desta página é de responsabilidade pessoal do autor e não expressa a visão da instituição a qual o autor tem vínculo profissional.
