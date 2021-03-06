---
title: Como generalizar ações de maneira eficiente
author: Renato Leripio
date: '2020-07-30'
categories:
  - Data Science
  - Previsão
tags:
  - loops
  - map
  - nest
slug: como-generalizar-ações-de-maneira-eficiente
lastmod: '2020-07-29T16:18:13-03:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
---

É muito comum no dia-a-dia precisarmos generalizar alguma ação sobre diversos conjuntos de dados. Por exemplo, quando desenvolvemos uma especificação para um modelo e queremos ajustá-la sobre várias unidades diferentes. Uma maneira eficiente de pensar essa tarefa é considerar dois objetos. 

O primeiro é uma **função** que realiza todas as ações pretendidas de forma genérica. O segundo é uma **lista** cujos elementos correspondem ao conjunto de informações referente a cada unidade. No fim das contas, o objetivo será aplicar a função a cada um dos elementos da lista.

Existem algumas formas de alcançar este objetivo, seja através de *loops* convencionais ou usando as variações da família `apply`. O conjunto de pacotes `tidyverse` fornece uma solução bem elegante e prática através das funções `nest` e `map`. 

Imagine que precisamos projetar a Produção Industrial (PIM-PF/IBGE) para todas as UF's disponíveis na pesquisa do IBGE. Como seria?

## Passo 1: carregar pacotes e importar os dados

Para importar os dados, vamos acessar a API do IBGE através do pacote `sidrar`.


```r
library(sidrar)
library(tidyverse)
library(fable)
library(tsibble)

dados_pim <- "/t/3653/n3/all/v/3135/p/all/c544/129314/d/v3135%201"

pim_ibge <- sidrar::get_sidra(api = dados_pim)

pim_ibge <- pim_ibge %>%
  
  dplyr::select(date = `Mês (Código)`,
                uf   = `Unidade da Federação`,
                valor = Valor) %>%
  
  dplyr::mutate(date = lubridate::ymd(paste0(date, "01")))
```

## Passo 2: colocar os dados no formato nest


```r
pim_ibge_nest <- pim_ibge %>%
  
  dplyr::group_by(uf) %>%
  
  tidyr::nest()

pim_ibge_nest
```

```
## # A tibble: 14 x 2
## # Groups:   uf [14]
##    uf                data              
##    <chr>             <list>            
##  1 Amazonas          <tibble [221 × 2]>
##  2 Pará              <tibble [221 × 2]>
##  3 Ceará             <tibble [221 × 2]>
##  4 Pernambuco        <tibble [221 × 2]>
##  5 Bahia             <tibble [221 × 2]>
##  6 Minas Gerais      <tibble [221 × 2]>
##  7 Espírito Santo    <tibble [221 × 2]>
##  8 Rio de Janeiro    <tibble [221 × 2]>
##  9 São Paulo         <tibble [221 × 2]>
## 10 Paraná            <tibble [221 × 2]>
## 11 Santa Catarina    <tibble [221 × 2]>
## 12 Rio Grande do Sul <tibble [221 × 2]>
## 13 Mato Grosso       <tibble [221 × 2]>
## 14 Goiás             <tibble [221 × 2]>
```

Repare no `tibble` criado pela função `nest`. Diferente das estruturas convencionais, ele permite armazenar objetos dentro das células -- aqui serão os `tibble`'s contendo os dados para cada UF. Na prática, isso funciona como uma *lista* sobre a qual podemos usar o `map` para aplicar a nossa função.

Vale ressaltar que seria equivalente utilizar uma lista. Para isso, poderíamos recorrer à função `dlply` do pacote `plyr` conforme apresentado a seguir. A vantagem do formato `nest` é que você pode ir adicionando novas colunas com outras transformações aos dados sem a necessidade de criar novos objetos.


```r
library(plyr)

pim_ibge_list <- pim_ibge %>%
  
  plyr::dlply(.variables = "uf")
```

## Passo 3: criar a função

Nossa função não precisa ser tão geral neste caso, dado que os nomes das colunas são iguais para todas as UF's -- `date` e `valor` -- e o modelo também será o mesmo. Basicamente, a ação aqui é pegar o argumento `x` que será um `tibble` e retornar as projeções de 1 a 6 períodos à frente utilizando um `ARIMA`. 


```r
fc_arima <- function(x){
    
    x %>%
      
      dplyr::mutate(date = tsibble::yearmonth(date)) %>%
      
      as_tsibble() %>%
      
      model(modelo = ARIMA(valor)) %>%
      
      forecast(h = 6)
    
  }
```

## Passo 4: estimar os modelos

De posse dos dados devidamente segmentados e da função, podemos usar a função `map` para finalizar o trabalho.


```r
pim_ibge_modelo <- pim_ibge_nest %>% 
  
  dplyr::mutate(modelos = purrr::map(.x = data, .f = fc_arima))

pim_ibge_modelo
```

```
## # A tibble: 14 x 3
## # Groups:   uf [14]
##    uf                data               modelos        
##    <chr>             <list>             <list>         
##  1 Amazonas          <tibble [221 × 2]> <fable [6 × 4]>
##  2 Pará              <tibble [221 × 2]> <fable [6 × 4]>
##  3 Ceará             <tibble [221 × 2]> <fable [6 × 4]>
##  4 Pernambuco        <tibble [221 × 2]> <fable [6 × 4]>
##  5 Bahia             <tibble [221 × 2]> <fable [6 × 4]>
##  6 Minas Gerais      <tibble [221 × 2]> <fable [6 × 4]>
##  7 Espírito Santo    <tibble [221 × 2]> <fable [6 × 4]>
##  8 Rio de Janeiro    <tibble [221 × 2]> <fable [6 × 4]>
##  9 São Paulo         <tibble [221 × 2]> <fable [6 × 4]>
## 10 Paraná            <tibble [221 × 2]> <fable [6 × 4]>
## 11 Santa Catarina    <tibble [221 × 2]> <fable [6 × 4]>
## 12 Rio Grande do Sul <tibble [221 × 2]> <fable [6 × 4]>
## 13 Mato Grosso       <tibble [221 × 2]> <fable [6 × 4]>
## 14 Goiás             <tibble [221 × 2]> <fable [6 × 4]>
```

Cada célula da coluna `modelos` contém o resultado da função, isto é, as informações sobre a distribuição das projeções para cada UF. Isto permite acessar tais informações como se fossem uma lista e, por exemplo, plotar gráficos como estes abaixo.

<img src="/post/2020-07-29-como-generalizar-acoes-de-maneira-eficiente_files/figure-html/step7-1.png" width="672" />

Para funções com mais de um argumento, a função `map` dispõe de variantes: `map2` para dois argumentos e `pmap` para múltiplos argumentos. Também seria factível usar um modelo para cada UF. O ponto aqui era mostrar que a abordagem **função + lista** elimina em grande medida repetições desnecessárias de código, além da facilidade em escalar o trabalho apenas adicionando novos elementos à lista.
