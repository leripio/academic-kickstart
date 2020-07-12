---
title: Como fazer um tracking eficiente das suas previsões
author: Renato Leripio
date: '2020-05-24'
slug: como-fazer-um-tracking-eficiente-das-suas-previsões
categories:
  - Previsão
  - Data Science
tags:
  - Previsão
  - tracking
  - rds
subtitle: ''
summary: ''
authors: []
lastmod: '2020-05-24T17:00:05-03:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---

Quem realiza projeções que precisam ser atualizadas com frequência deve se preocupar em manter seus resultados armazenados, de modo que seja possível recuperá-los a qualquer momento a fim de acompanhar sua performance. Por exemplo, se o trabalho é projetar diariamente o valor de uma variável para os próximos 15 dias, manter uma cópia dos resultados a cada dia permite verificar o comportamento das trajetórias ao longo do tempo. 

Além disso, esse *backup* também torna possível computar métricas de acurácia para cada horizonte de previsão -- o que é muito importante para ter noção dos pontos fortes e fracos do modelo. Afinal, um modelo pode ser muito bom para projetar os próximos 7 dias mas descolar bastante depois disso. De maneira alternativa, o modelo pode ser capaz de capturar com precisão períodos mais longos e falhar nos horizontes curtos. 

Entendida a necessidade de manter guardados os resultados, o próximo passo consiste em fazer a tarefa de maneira simples e eficiente. Para isso, eu sugiro uma abordagem em dois passos. Primeiro, criar um `frame` com três colunas -- a data **na** qual as projeções foram geradas; a data **para a qual** as projeções foram feitas; e, por fim, os valores projetados. É possível também adicionar informações extras, como a especificação do modelo utilizado -- caso este seja atualizado a cada rodada, por exemplo. Mas para manter as coisas simples, vamos focar apenas nas informações esssenciais. 

O segundo passo é armazenar fisicamente a informação para que seja possível acessá-la mais tarde. Aqui, a melhor maneira é utilizar o objeto `RDS` -- que é, basicamente, a estrutura para armazenagem de objetos do R. Uma boa ideia neste caso é exportar um arquivo `RDS` contendo o `frame` a cada nova rodada de projeções e mantê-los em uma pasta. Isto garante que você trabalhará sempre com arquivos pequenos (em geral de poucos kilobytes) e que tenha controle do espaço em disco usado para armazenar o seu *tracking*.

Estando claro o objetivo, vamos a um exemplo prático de como fazer isso. Para utilizar um contexto atual, vamos assumir que nosso objetivo seja projetar todos os dias o número total de casos de Covid-19 no Brasil para os próximos 7 dias a partir de um modelo univariado bem simples -- um `ets` do pacote `fable` (a versão `tidy` do `forecast`). Os dados serão obtidos através do pacote `datacovidbr` (https://github.com/Freguglia/datacovidbr), desenvolvido pelo Victor Freguglia e que reúne informações sobre Covid-19 a partir de diversas fontes.

### Passo 1: carregar pacotes e obter os dados 


```r
library(tidyverse)
library(lubridate)
library(fable)
library(tsibble)
library(datacovidbr)

dadosBR <- datacovidbr::brMinisterioSaude() %>% 
  
  dplyr::filter(regiao == "Brasil") %>%
  
  dplyr::select(date, casosAcumulado)

tail(dadosBR, 3)
```

```
## # A tibble: 3 x 2
##   date       casosAcumulado
##   <date>              <dbl>
## 1 2020-07-10        1800827
## 2 2020-07-11        1839850
## 3 2020-07-12        1864681
```

### Passo 2: ajustar o modelo ETS, realizar as projeções e salvar objeto RDS

Aqui, vamos utilizar a função `Sys.Date()` para introduzir a coluna com o dia no qual foram feitas as projeções, isto é, o dia atual.


```r
proj <- dadosBR %>%
  
  as_tsibble() %>%
  
  model(ets = ETS(casosAcumulado)) %>%
  
  forecast(h = "7 days") %>%
  
  dplyr::mutate(dateFrom = Sys.Date())

tail(proj, 3)
```

```
## # A fable: 3 x 5 [1D]
## # Key:     .model [1]
##   .model date       casosAcumulado .distribution       dateFrom  
##   <chr>  <date>              <dbl> <dist>              <date>    
## 1 ets    2020-07-17       2068636. N(2068636, 1.7e+08) 2020-07-12
## 2 ets    2020-07-18       2106768. N(2106768, 2.5e+08) 2020-07-12
## 3 ets    2020-07-19       2131422. N(2131422, 3.6e+08) 2020-07-12
```

Em seguida, basta salvar o `frame`. Uma boa ideia é usar a data em que foram geradas as projeções no nome do arquivo.


```r
saveRDS(proj, paste0("tracking_", Sys.Date(), ".rds"))
```

Para termos uma noção do que seria o resultado ao longo de um período maior, reproduzi a ação anterior para os últimos 15 dias. Assim, existem na pasta 15 arquivos `.rds` correspondendo à trajetória de projeção gerada em cada um dos últimos 15 dias. Esses arquivos podem ser importados de uma só vez utilizando como identificador o nome ou a extensão, empilhados e utilizados para análises diversas.


```r
proj_labs <- list.files()[str_detect(list.files(), ".rds")]

proj_files <- purrr::map_dfr(.x = proj_labs, .f = readRDS)

proj_files <- proj_files %>% 
  
  as_tibble() %>%
  
  dplyr::select(date, previsto = casosAcumulado, dateFrom)
```

Para finalizar, devemos adicionar os dados que foram realizados. Isto pode ser feito dando um `join` com o nosso objeto `dadosBR`, utilizando a coluna de data como identificador. Para que o `frame` fique mais claro, vamos renomear a coluna.


```r
proj_tracking <- dplyr::left_join(proj_files, 
                                  dadosBR %>% 
                                    dplyr::rename("realizado" = "casosAcumulado"))
```

Podemos utilizar `proj_tracking` para gerar o gráfico abaixo, em que os pontos representam os valores realizados e as linhas são as trajetórias projetadas em cada dia. 

<img src="2020-05-24-como-fazer-um-tracking-eficiente-das-suas-previsões_files/figure-html/plot-1.png" width="672" style="display: block; margin: auto;" />

Também vale analisar a acurácia do modelo para cada horizonte preditivo, conforme o gráfico abaixo:

<img src="2020-05-24-como-fazer-um-tracking-eficiente-das-suas-previsões_files/figure-html/acc-1.png" width="672" style="display: block; margin: auto;" />

**Aviso legal**: Todo o conteúdo desta página é de responsabilidade pessoal do autor e não expressa a visão da instituição à qual o autor tem vínculo profissional.
