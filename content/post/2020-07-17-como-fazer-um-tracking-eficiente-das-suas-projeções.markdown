---
title: Como fazer um tracking eficiente das suas projeções
author: Renato Leripio
date: '2020-07-23'
categories:
  - Data Science
  - Previsão
tags:
  - Previsão
  - tracking
slug: como-fazer-um-tracking-eficiente-das-suas-projeções
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
---

Quem realiza projeções deve manter seus resultados armazenados, de modo que seja possível recuperá-los a qualquer momento. Isso permite, por exemplo, acompanhar o desempenho do modelo ao longo do tempo. 

A tarefa pode ser feita em dois passos. Primeiro, criar um `tibble` com três colunas: a data **na** qual as projeções foram geradas; a data **para a qual** as projeções foram feitas; e, por fim, os valores (ou distribuição) projetados. É possível também adicionar informações extras, como a especificação do modelo utilizado -- caso este seja atualizado regularmente.

O segundo passo é armazenar fisicamente a informação para que seja possível acessá-la mais tarde. Aqui, eu prefiro utilizar o objeto `RDS` -- que é, basicamente, a estrutura para armazenagem de objetos do R. Uma boa ideia neste caso é exportar um arquivo `RDS` contendo o `tibble` a cada nova rodada de projeções e mantê-los em uma pasta.

Vamos a um exemplo prático de como fazer isso. Para utilizar um contexto atual, vamos assumir que nosso objetivo seja projetar todos os dias o número de novos casos de Covid-19 no Brasil para os próximos 7 dias a partir de um modelo univariado bem simples -- um `ETS` automático do pacote `fable` (a versão `tidy` do pacote `forecast`). Os dados são importados do repositório público do Wesley Cota (https://github.com/wcota/covid19br/).

### Passo 1: carregar pacotes e obter os dados 


```r
library(tidyverse)
library(lubridate)
library(fable)
library(tsibble)
library(datacovidbr)

dados_url <- "https://raw.githubusercontent.com/wcota/covid19br/master/cases-brazil-states.csv"

dadosBR <- readr::read_csv(dados_url) %>%
  
  dplyr::filter(country == "Brazil", state == "TOTAL",
                date >= max(date) %m-% days(60)) %>%
  
  dplyr::select(date, newCases)

tail(dadosBR, 3)
```

```
## # A tibble: 3 x 2
##   date       newCases
##   <date>        <dbl>
## 1 2020-07-21    46927
## 2 2020-07-22    62943
## 3 2020-07-23    56111
```

### Passo 2: ajustar o modelo ETS, realizar as projeções e salvar objeto RDS




```r
  proj <- dadosBR %>%
  
  as_tsibble() %>%
  
  model(ets = ETS(newCases)) %>%
  
  forecast(h = "7 days") %>%
  
  dplyr::mutate(dateFrom = max(dadosBR$date))

tail(proj, 3)
```

```
## # A fable: 3 x 5 [1D]
## # Key:     .model [1]
##   .model date       newCases .distribution     dateFrom  
##   <chr>  <date>        <dbl> <dist>            <date>    
## 1 ets    2020-07-28   46116. N(46116, 7.8e+07) 2020-07-23
## 2 ets    2020-07-29   47461. N(47461, 8.2e+07) 2020-07-23
## 3 ets    2020-07-30   47584. N(47584, 8.3e+07) 2020-07-23
```

Em seguida, basta salvar o objeto. Uma boa ideia é incluir alguma referência no nome do arquivo: a data em que foram geradas as projeções ou a data da última observação disponível, por exemplo.


```r
saveRDS(proj, paste0("tracking/forecast_", unique(proj$dateFrom), ".rds"))
```

Para termos uma noção do que seria o resultado ao longo de um período maior, reproduzi a ação anterior por um período de 30 dias. Assim, existem na pasta 30 arquivos `.rds` correspondendo à trajetória de projeção gerada em cada um dos 30 dias. Esses arquivos podem ser importados de uma só vez -- utilizando como identificador um padrão do nome ou simplesmente a extensão -- e empilhados para produzir as análises.


```r
proj_labs <- list.files("tracking/")[str_detect(list.files("tracking/"), ".rds")]

proj_files <- purrr::map_dfr(.x = proj_labs, 
                             .f = function(x) readRDS(paste0("tracking/", x)))
```

Para finalizar, devemos adicionar os dados que foram realizados. Isto pode ser feito dando um `join` com o nosso objeto `dadosBR`, utilizando a coluna de data como identificador. Para que o `tibble` fique mais claro, vamos renomear a coluna.


```r
proj_tracking <- dplyr::left_join(
  
  proj_files %>% 
    
    as_tibble() %>%
    
    dplyr::rename("previsto" = "newCases"),
  
  dadosBR %>% dplyr::rename("realizado" = "newCases")
  
  )
```

Podemos utilizar `proj_tracking` de diversas formas para acompanhar o desempenho do nosso modelo. O gráfico abaixo, por exemplo, traz a evolução das projeções um passo à frente. Apesar de o modelo não ser tão aderente à magnitude, ele é capaz de capturar a sazonalidade da série. Vale lembrar que o modelo utilizado é bem simples e serve apenas para ilustração.

<img src="/post/2020-07-17-como-fazer-um-tracking-eficiente-das-suas-projeções_files/figure-html/plot-1.png" width="672" style="display: block; margin: auto;" />

Por fim, também podemos utilizar os dados gerados anteriormente para computar métricas de acurácia para cada horizonte de previsão. Para isso, vamos utilizar a função `mape` do pacote `yardstick` para obter o erro absoluto percentual médio (MAPE). O pacote conta com diversas outras métricas conhecidas: `rmse`, `mae`, `mase`, etc. 


```r
library(yardstick)

proj_acc <- proj_tracking %>%
  
  dplyr::group_by(dateFrom) %>%
  
  dplyr::mutate(step_ahead = 1:n()) %>%
  
  dplyr::group_by(step_ahead) %>%
  
  yardstick::mape(truth = realizado, estimate = previsto, na.rm = TRUE)
```

<img src="/post/2020-07-17-como-fazer-um-tracking-eficiente-das-suas-projeções_files/figure-html/acc_plot-1.png" width="672" />
  

**Aviso legal**: Todo o conteúdo desta página é de responsabilidade pessoal do autor e não expressa a visão da instituição a qual o autor tem vínculo profissional.
