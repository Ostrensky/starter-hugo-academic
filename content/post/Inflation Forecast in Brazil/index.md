---
title: Forecasting inflation in Brazil
subtitle: ""

# Summary for listings and search engines
summary: This is a monthly inflation forecasting model for Brazil (as measured by the IPCA).

# Link this post with a project
projects: []

# Date published
date: "2021-10-18T00:00:00Z"

# Date updated
date: "2021-10-18T00:00:00Z"

# Is this an unpublished draft?
draft: false

# Show this page in the Featured widget?
featured: false

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: ""
  focal_point: ""
  placement: 2
  preview_only: false

authors:
- admin

tags:
- Forecasting

categories:
- Time Series

bibliography: 'ref.bib'
---


# Introdução

Este documento demonstra o passo-a-passo do modelo de previsão do IPCA.



```r
library(gtrendsR)
library(ipeadatar)
library(tsibble)
library(zoo)
library(forecast)
library(tidyverse)
library(sidrar)
library(lubridate)
library(tidyverse)
library(randomForest)
library(timetk)
library(HDeconometrics) 
library(knitr)
library(rbcb)

set.seed(1984)
```


# Extração dos dados

Procuramos restringir as fontes de dados na obtenção das variáveis a serem utilizadas, de forma a simplificar o processo de extração. Assim, as variáveis econômicas foram obtidas junto às bases de dados do IBGE/Sidra e IPEA, por meio dos pacotes sidrar e ipeadatar, que utilizam as API´s de ambos os órgãos. Além disso, por meio do pacote gtrendsR, também incluímos na análise uma variável obtida do Google Trends. O termo de busca foi para "Seguro desemprego", que tende a ser uma boa antecipação da tendência do mercado de trabalho. Esse é um tipo de dado que vem ganhando bastante relevância nas previsões econômicas [@choi2012predicting; @naccarato2018combining]. Por último, também selecionamos a mediana da previsão do IPCA do sistema de expectativas Focus, que trataremos mais detalhadamente adiante. 

A escolha das variáveis preditoras se baseou em estudos que buscavam prever inflação, como @baybuza2018inflation, e @mandalinci2017forecasting e, principalmente para o Brasil, como @garcia2017real e @medeiros2021forecasting. Buscou-se utilizar dados que representassem diferentes áreas da economia, como monetária, fiscal, financeiro, mercado de trabalho e produção. As variáveis utilizadas, as transformações realizadas e as respectivas fontes estão descritas na tabela 1, no apêndice. 



```r
IPCA = sidrar::get_sidra(118, period = "all") %>% #9
  select(`Mês (Código)`, Valor) %>%
  mutate(date =   yearmonth(ymd(as.character(paste0(`Mês (Código)`, "01"))))) %>%
  rename(ipca = Valor) %>%
  select(-`Mês (Código)`)


cambio = ipeadata("PAN12_ERV12") %>% 
  select(date, value) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(cambio = value) 


seguro = gtrends(keyword = c("seguro desemprego"),
                 geo = "BR", time='all', onlyInterest=TRUE)

seguro = seguro$interest_over_time %>% 
  select(date, hits) %>%
  head(., nrow(.) -1) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(seguro = hits) 


civil_price = sidrar::get_sidra(2296, period = "all", variable = 1198) %>% #31
  select('Mês (Código)', Valor) %>%
  separate('Mês (Código)', into = c("pre", "post"), sep = -2) %>% 
  mutate(date = paste0(pre, "-", post)) %>% 
  select(-pre,-post) %>%
  mutate(date =   yearmonth(ymd(as.character(paste0(date, "-01"))))) %>%
  rename(civil_price = Valor) 



interestrate  = ipeadatar::ipeadata("ANBIMA12_TJTLN1212") %>% 
  select(date, value ) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(interest = value) 


economic_activity  = ipeadatar::ipeadata("BM12_PIB12") %>%
  select(date, value ) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(economic_activity = value) 


IGPM  = ipeadatar::ipeadata("IGP12_IGPMG12") %>%
  select(date, value ) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(IGPM = value)

IGPDI  = ipeadatar::ipeadata("IGP12_IGPDIG12") %>%
  select(date, value ) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(IGPDI = value)

 
M1  = ipeadatar::ipeadata("BM12_M1MN12") %>%
  select(date, value ) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(M1 = value) 


M2  = ipeadatar::ipeadata("BM12_DEPOUCNY12") %>%
  select(date, value ) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(M2 = value) 

M3 = ipeadatar::ipeadata("BM12_FRFCN12") %>%
  select(date, value ) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(M3 = value) 

M4 = ipeadatar::ipeadata("BM12_M4NCN12") %>%
  select(date, value ) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(M4 = value) 

manu_production = sidrar::get_sidra(3651, variable = 4139, classific = "c543", category =  list(129300), period = "all") %>% #9
  select('Mês (Código)', Valor) %>%
  separate('Mês (Código)', into = c("pre", "post"), sep = -2) %>% 
  mutate(date = paste0(pre, "-", post)) %>% 
  select(-pre,-post) %>%
  mutate(date =   yearmonth(ymd(as.character(paste0(date, "-01"))))) %>%
  rename(manu_production = Valor) 

caged = ipeadatar::ipeadata("CAGED12_SALDO12") %>%
  select(date, value ) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(caged = value) %>%
  bind_rows(ipeadatar::ipeadata("CAGED12_SALDON12") %>%
              select(date, value ) %>%
              mutate(date =   yearmonth(ymd(as.character(date)))) %>%
              rename(caged = value)) 


trade_balance = ipeadatar::ipeadata("PAN12_SBC12") %>%
  select(date, value ) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(trade_balance = value) 

income_tax_revenue = ipeadatar::ipeadata("SRF12_IR12") %>%
  select(date, value ) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename( income_tax_revenue = value) 

bovespa = ipeadatar::ipeadata("ANBIMA12_IBVSP12") %>%
  select(date, value ) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(bovespa = value) 

sales_commerce = ipeadatar::ipeadata("PAN12_IVVRG12") %>%
  select(date, value ) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(sales_commerce = value)   


sales_vehicles = ipeadatar::ipeadata("FENABRAVE12_VENDVETOT12") %>%
  select(date, value ) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(sales_vehicles = value) 

public_debt = ipeadatar::ipeadata("PAN12_DTSPY12") %>%
  select(date, value ) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(public_debt  = value) 

energy_consumption = ipeadatar::ipeadata("ELETRO12_CEET12") %>%
  select(date, value ) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(energy_consumption  = value) 


oil_price = ipeadata("EIA366_PBRENT366") %>% 
  select(date, value) %>%
  mutate(date =   yearmonth(ymd(as.character(date)))) %>%
  rename(oil_price = value) 

oil_price =  oil_price[!duplicated(oil_price$date, fromLast = T), ]
```

## Focus

Os dados do sistema de expectativas do Banco Central (Focus) mercem uma atenção especial. Essa base é formada por expectativas de diversas instituições cadastradas no Banco central. Nossa estratégia é estimar a inflação todo dia 15, quando a inflação do mês anterior já foi divulgada. Assim, todos as variáveis utilizadas são as que estiverem disponíveis no dia 15. Como os dados do Focus são diários, escolhemos sempre os últimos dados disponíveis até o dia 15 de cada mês. Optamos por apenas selecionar a mediana das expectativas para cada um dos próximos 12 meses, incluindo o corrente, ou seja t até t+11. 

Há dois motivos muito importantes para utilizarmos os dados do Focus. O primeiro deles é o uso deles como benchmarking de performance. A previsão do IPCA por meio do Focus tem custo 0, já que é facilmente obtida. Além disso, sua performance é melhor que a de um passeio aleatório. Portanto, para um modelo de previsão ser útil, é preciso que ele tenha um desempenho acima do Focus. O segundo motivo é até mais importante. Nem toda informação disponível no mercado é capturada pelos dados macroeconômicos. Assim, quando utilizamos previsões externas no nosso modelo, estamos, na prática, utilizando informações que não temos, mas que outros players do mercado tem, o que melhora o desempenho do nosso modelo.




```r
  focus = get_monthly_market_expectations("IPCA") %>%
    mutate(monthyear = paste0(year(date),"-", month(date))) %>%
    group_by(monthyear) %>% 
    filter(day(date) < 16) %>%
    filter(abs(day(date) - 15) == min(abs(day(date) - 15))) %>%
    ungroup() %>%
    mutate(date =  yearmonth(ymd(as.character(date))),
           monthyear =  as.yearmon(paste0(monthyear, "-01"), "%Y-%m-%d"),
           reference_month = as.yearmon(paste0(reference_date, "-01"), "%Y-%m-%d"),
           horizon = round((reference_month - monthyear)*12 +1)) %>%
    filter(horizon > 0 & horizon < 13 & base == 0 ) %>%
    distinct(date, horizon, .keep_all = T) %>% 
    select(date,horizon, median) %>%
    pivot_wider(id_cols = date,  names_from = horizon, 
                values_from = median, names_prefix = "median_h_")
```


## Juntando as variáveis

Depois de reunidas todas as variáveis, elas são colocadas em um só dataframe.


```r
data = list(IPCA, economic_activity, IGPM, interestrate, civil_price, seguro, 
            cambio, M1,M2,M3,M4, manu_production, caged, trade_balance,
            income_tax_revenue, bovespa, IGPDI, sales_commerce, sales_vehicles,
            public_debt, energy_consumption,  oil_price) %>%
  reduce(left_join, by = "date")

data <- full_join(data, focus)
```


# Transformando as variáveis 

## Tornando as séries estacionárias

Quando estamos tratando de problemas de séries temporais, é importante que se trabalhe com séries estacionárias. Alguns modelos são robustos a presenção de não-estacionariedade, mas não é o caso dos que a gente irá utilizar para modelar. A dependência temporal das variáveis pode acarretar alguns problemas estatísticos. No caso da previsão, muito provavelmente o principal seria o chamado overfitting, que acontece quando o modelo tem boa perfomance no treino, mas má performance quando o testamos [@adhikari2013introductory; @friedman2001elements]. Assim, as séries que não são estacionárias se tornaram por meio da diferenciação de 1 ou 2 ordens, conforme a estatística do teste Dickey-Fuller aumentado.  



```r
data <- data %>%
  mutate(cambio = (cambio/lag(cambio,1) -1) *100,
         seguro = difference(seguro,1),
         civil_price = difference(civil_price,12),
         interest = difference(interest,12),
         economic_activity = difference(log(economic_activity),12),
         M1 = difference(log(M1),12, differences = 2),
         M2 = difference(log(M2),12, differences = 1),
         M3 = difference(log(M3),12, differences = 1),
         M4 = difference(log(M4),12, differences = 1),
         caged = difference(caged, 12, differences = 1),
         trade_balance = difference(trade_balance,12, differences = 1),
         income_tax_revenue = difference(income_tax_revenue,12, 2),
         sales_commerce = difference(sales_commerce,12),
         sales_vehicles = difference(sales_vehicles,12,2),
         public_debt = difference(public_debt,12,2),
         energy_consumption = difference(energy_consumption,12),
         oil_price = difference(oil_price,12))
```

## "Publication lag" e inclusão de lags nas variáveis

Como mencionado anteriormente, nosso modelo retornará as previsões todo dia 15, utilizando os dados do mês anterior (t-1), assim excluímos as os dados do mês presente, com exceção dos dados do boletim focus. Porém, nem todas as variáveis são divulgadas ao mesmo dia. Assim, temos que considerar esse lag de publicação para algumas variáveis na hora de estimarmos o modelo, a fim de não incorrermos no chamado "look-ahead bias", ou seja, fazer previsão com dados que você não teria no mundo real. Para isso, utilizamos os dados de dois meses antes (t-2). As variáveis em que isso ocorre são: M1,M2,M3,M4, Caged, produção manufatureira, atividade econômica, receita com impostos, vendas do comércio, dívida pública e consumo de energia.

Outra importante transformação é a inclusão de lags nas variáveis. Decidimos acrescentar 4 defasagens para cada variável (incluindo a própria variável dependente). Assim, utilizamos os dados xt-1, xt-2, xt-3 e xt-4. No caso das variáveis proveniente do Focus, também incluímos xt. Incluir variáveis defesadas como potencial preditoras do IPCA é importante, pois o impacto não é simultâneo. Mais defasagens poderiam ser incluídas, mas com o risco de se aumentar o ruído nos dados. 

Na última parte do código filtramos a base para ficar com os dados apenas a partir de Novembro de 2004.



```r
nlags <- 4

variables <- data[-c(1:2)] %>%
  select(-c(median_h_1,median_h_2,median_h_3,median_h_4,median_h_5,median_h_6,median_h_7,
            median_h_8,median_h_9,median_h_10,median_h_11,median_h_12)) %>%
  names()
```


```r
data <- data  %>%
  mutate(M1 = lag(M1),#Accounting for publication lag
         M2 = lag(M2),
         M3 = lag(M3),
         M4 = lag(M4),
         caged = lag(caged),
         manu_production = lag(manu_production),
         economic_activity = lag(economic_activity),
         income_tax_revenue = lag(income_tax_revenue),
         sales_commerce = lag(sales_commerce),
         public_debt = lag(public_debt),
         energy_consumption = lag(energy_consumption,2)) %>%
  tk_augment_lags(-one_of("date"), .lags = 1:nlags) %>%
  select(!variables) %>%
  as_tsibble() %>%
  filter_index("2004 nov" ~ .) %>%
  relocate(date) 
```

## Dividindo a base

Para avaliarmos a performance do modelo é preciso separar a base de dados entre treino e teste. Assim, separamos até o fim de 2014 como treino do modelo e de 2015 até março de 2021 como período de teste. Uma outra abordagem poderia ser a utilização de cross-validation para avaliação de performance e otimização dos hiperparâmetros dos modelos. Entretanto essa é uma forma mais intensiva computacionalmente. 


```r
end_train <- "2014 dec"

start_test <- "2015 jan"


train <-data %>% 
  filter_index(~ end_train) %>%
  as.data.frame()

test  <- data %>% 
  filter_index(start_test ~ .) %>%
  as.data.frame()

forecast_row <- data %>% #última linha
  filter(row_number() == n())

data_list = list(train, test, forecast_row)


train <- data_list[[1]]
test <- data_list[[2]]
forecast_row <- data_list[[3]]

train_date <- train$date
test_date <- test$date

x_train <- as.matrix(train[,c(3:ncol(train))])
y_train <- train$ipca

x_test <- as.matrix(test[,c(3:ncol(test))])
y_test <- test$ipca
```

# Modelos

Como estamos utilizando 34 variáveis exógenas, sendo 12 delas com 5 lags e 22 com 4, totalizando 148 variáveis (k). Nossa base de treino tem 10 anos, portanto são 120 observações (n). Portanto estamos com uma alta dimensionalidade (k>n) e essa é uma questão que traz implicações estatísticas bastante relevantes, como baixa generalização dos modelos [@friedman2001elements].  Por isso, precisamos utilizar métodos que sejam específicos para tratar esse problema de dimensionalidade. Selecionamos três deles que têm fácil aplicação, pois não necessitam de muita otimização de hiperparâmetros. São eles; Complete subset regression, Bagging e Lasso. Cabe mencionar que os três modelos são variações da regressão linear.

Assim, formamos um ensemble destes três modelos com a média de suas previsões. O motivo de utilizarmos os três modelos juntos é por causa do fato, que há muito se sabe na literatura de previsão, que quando juntamos diferentes modelos, temos resultados em média melhores do que os resultados individuais de suas previsões. Formas mais rebuscadas de combinar as previsões também podem ser aplicadas [@kolassa2011combining]. 

Além disso, nossa abordagem para realizarmos previsões para diferentes horizontes temporais será estimar um modelo para cada um deles. Assim, teremos um modelo para estimarmos o ipca no mês corrente (t), outro para o mês seguinte (t+1), seguindo assim até t+11.




## Complete subset regression (CSR)

Esse é um método relativamente novo e simples, apresentado por @elliott2013complete. Ele é bastante interessante para problemas com alta dimensionalidade. Nele, são estimadas todas as regressões possíveis de combinações entre as variáveis preditivas, o que auxilia no controle do problema da multicolinearidade. Entretanto, estimar todas as combinações possíveis em 148 variáveis demanda muito computacionalmente. Assim, seguimos o procedimento adotado por X @garcia2017real, em que há um pré teste com cada uma das variáveis estimadas em uma regressão simples contra o IPCA. Assim, são selecionadas as 20 variáveis com melhores valores na estatística t. Com elas são estimadas todas as combinações possíveis de regressões com 15 parâmetros, totalizando 15.504 regressões. Tomando a média das previsões dessas regressões, temos a previsão do modelo.

## Bagging

O Bagging é uma técnica que visa diminuir a instabilidade de modelos [@friedman2001elements]. A idéia é utilizar um processo de reamostragem aleatória com substituição chamado bootstrap. Assim, para cada amostra é estimada uma regressão linear com as variáveis. Em seguida é tomada a média dos coeficientes estimados. O principal efeito desse modelo é evitar que ruídos deixem de fora variáveis importantes para a previsão, melhorando capacidade do modelo de generalização (bom desempenho em dados futuros). Optamos por escolher 500 replicações de bootstrap, mas esse valor é arbitrário. 



## Lasso

Lasso (Least Absolute Shrinkage and Selection Operator) é um método desenvolvido por @tibshirani1996regression que tem como objetivo reduzir o número de parâmetros, principalmente em casos de multicolineariedade. Assim, é adicionado a minimização dos mínimos quadrados uma penalização baseada na magnitude absoluta dos coeficientes:


\begin{equation}
\sum_{i=1}^{n}\left(y_{i}-\sum_{j} x_{i j} \beta_{j}\right)^{2}+\lambda \sum_{j=1}^{p}\left|\beta_{j}\right|
\end{equation}

Fica evidente com base na equação acima que o valor de $\lambda$ é extremamente relevante para o resultado do modelo. Se esse parâmetro aumenta, o viés do estimador também aumenta. Já se ele diminui, quem aumenta é a variância. Caso ele seja 0, ele se torna igual ao estimador OLS. A forma usual de encontrar o valor ótimo de $\lambda$ é por meio da validação cruzada (CV). Entretanto, como não optamos por realizar CV por questões computacionais, seguiremos a abordagem de [@zou2007degrees],  utilizando o valor do Critério de informação Bayesiano (BIC) para achar o melhor $\lambda$.



```r
pred_csr = list()
pred_lasso = list()
pred_rw = list()
pred_focus = list()
pred_bagging = list()
pred_ensemble = list()
acc_csr = list()
acc_lasso = list()
acc_rw = list()
acc_focus = list()
acc_bagging = list()
acc_ensemble = list()

for(i in 1:12){
  
  csr_model = csr(x_train,y_train,K=20,k=15)
  pred_csr[[i]] = predict(csr_model, newdata = x_test) 
  
  lasso_model = ic.glmnet(x_train,y_train,crit = "bic")
  pred_lasso[[i]] = as.numeric(predict(lasso_model,newdata = x_test))
  
  bagging_model = bagging(x_train, y_train, R = 500)
  pred_bagging[[i]] = as.numeric(predict(bagging_model, newdata = x_test))
  
  pred_rw[[i]] = x_test[,13]
  
  pred_ensemble[[i]] = rowMeans(cbind(pred_csr[[i]], pred_lasso[[i]], pred_bagging[[i]]))
  
  y_train = y_train[-1] 
  x_train = x_train[-nrow(x_train), ] 
  
  lead_y <- lead(y_test,i-1)
  
    
  acc_rw[[i]] <- accuracy(pred_rw[[i]], lead_y)
  acc_csr[[i]] <- accuracy(pred_csr[[i]], lead_y)
  acc_lasso[[i]] <- accuracy(pred_lasso[[i]], lead_y)
  acc_bagging[[i]] <- accuracy(pred_bagging[[i]], lead_y)
  
  acc_ensemble[[i]] <- accuracy(pred_ensemble[[i]], lead_y)
  
  
  horizon <- paste0("median_h_", i)
  pred_focus[[i]] <- x_test[,colnames(x_test) == horizon]
  acc_focus[[i]] <-  accuracy(pred_focus[[i]], lead_y)
}
```


```r
rmse_fore <- data.frame(cbind(unlist(map(acc_focus, 2)), unlist(map(acc_rw, 2)), unlist(map(acc_csr, 2)),
                             unlist(map(acc_lasso, 2)), unlist(map(acc_bagging, 2)), unlist(map(acc_ensemble, 2))))

mae_fore <- data.frame(cbind(unlist(map(acc_focus, 3)), unlist(map(acc_rw, 3)), unlist(map(acc_csr, 3)),
                             unlist(map(acc_lasso, 3)), unlist(map(acc_bagging, 3)), unlist(map(acc_ensemble, 3))))

names(rmse_fore) <- c("focus", "rw", "csr", "lasso", "bagging", "ensemble")
```

```r
rmse_fore$h <- seq(1:12)
```

```r
mae_fore$h <- seq(1:12)
```


```r
forecast_12months <- data.frame(cbind(unlist(lapply(pred_csr, tail, 1)),
                                      unlist(lapply(pred_lasso, tail, 1)),
                                      unlist(lapply(pred_bagging, tail, 1)),
                                      unlist(lapply(pred_ensemble, tail, 1)),
                                      unlist(lapply(pred_focus, tail, 1))))

names(forecast_12months) <- c("csr", "lasso", "bagging", "ensemble", "focus")
```

```r
forecast_12months$h <- seq(1:12)
```



```r
pred_test <- list(pred_focus, pred_ensemble, y_test)

list_acc_pred <- list(rmse_fore, mae_fore, forecast_12months, pred_test)
```


# Métricas

Com os modelos estimados e os valores previstos para o período de teste, precisamos avaliar os modelos conforme alguma métrica de erro. Para fins de simplicidade, escolhemos as duas que provavelmente são as mais comuns para modelos de regressão: o erro médio absoluto (MAE) e a raíz do erro quadrático médio (RMSE). A diferença prática entre os dois é que o RMSE coloca pesos maiores para erros maiores (outliers). Como citado anteriormente, utilizaremos dois benchmarkings, o passeio aleatório e a mediana das previsões do Focus. 

Assim, podemos dispor a comparação entre o modelo de ensemble e os dois benchmarkings para cada um dos doze horizontes de previsão. Nota-se que o desempenho do nosso modelo é sistemáticamente superior à mediana do Focus. 


```r
rmse<- list_acc_pred[[1]]
mae<- list_acc_pred[[2]]
pred <- list_acc_pred[[3]]
pred_test <- list_acc_pred[[4]]

rmse$h <- seq(1:12)
```


```r
mae$h <- seq(1:12)
```


```r
pred$h <- as.Date(append_row(forecast_row,11)$date)
```


```r
rmse$metric <- "RMSE"
mae$metric <- "MAE"

error_metric <- rbind(rmse, mae) %>% select(rw, ensemble, focus, h, metric)
```

```r
error_metric %>% 
    gather("modelo", "value", -h, - metric) %>%
    ggplot(aes(x = as.factor(h), y = value, color = modelo)) +
    geom_line(size = 1) +
    geom_point() +
    theme_minimal() + 
    scale_x_discrete() +
    ylab("MAE") +
    xlab("Horizon") +
    facet_wrap(~ metric)
```



# Exportando os dados para o shiny


```r
write.csv(error_metric, "Inflation_app/error_metric.csv")
```



```r
write.csv(pred, "Inflation_app/pred.csv")
```

```r
save(pred_test, file="Inflation_app/pred_test.RData")
```


```r
test_date <- test$date

save(test_date, file="Inflation_app/test_date.RData")
```


# Apêndice


```r
apend <- read.csv2("variaveis.csv")
```


```r
apend$X <- NULL

kable(apend)
```



|Variável                                                       |Transformação                                          |Fonte                                   |
|:--------------------------------------------------------------|:------------------------------------------------------|:---------------------------------------|
|IPCA                                                           |-                                                      |IBGE                                    |
|Taxa de câmbio                                                 |Crescimento em relação ao mês anterior                 |IPEA                                    |
|Custo médio m² em moeda corrente                               |Diferença em relação a 12 meses                        |IBGE                                    |
|Índice de buscas por "Seguro desemprego" no google             |Diferença em relação ao mês anterior                   |Google                                  |
|Taxa de juros pré fixada -LTN                                  |Diferença em relação a 12 meses                        |Banco Central do Brasil                 |
|Índice de atividade econômica                                  |Log e Diferença em relação a 12 meses                  |Banco Central do Brasil                 |
|Índice Geral de Preços                                         |-                                                      |FGV                                     |
|Índice Geral de Preços - Disponibilidade Interna               |-                                                      |FGV                                     |
|Meio de Pagamento - Restrito (M1)                              |Log e segunda iferença em relação a 12 meses           |Banco Central do Brasil                 |
|Meio de Pagamento - Ampliado - M2 - depósitos em poupança      |Log e Diferença em relação a 12 meses                  |Banco Central do Brasil                 |
|Meio de Pagamento - Ampliado - M3                              |Log e Diferença em relação a 12 meses                  |Banco Central do Brasil                 |
|Meio de Pagamento - Ampliado - M4                              |Log e Diferença em relação a 12 meses                  |Banco Central do Brasil                 |
|Pesquisa Industrial Mensal - Produção Física (bens de consumo) |Variação percentual ao mês anterior com ajuste sazonal |IBGE                                    |
|Saldo CAGED (novo e velho)                                     |Diferença em relação a 12 meses                        |Ministério da economia                  |
|Saldo da Balança comercial                                     |Diferença em relação a 12 meses                        |Banco Central do Brasil                 |
|Receita Bruta - Imposto de renda                               |Segunda diferença em relação a 12 meses                |Ministério da economia                  |
|Variação mensal do Índice da Ibovespa                          |-                                                      |IPEA                                    |
|Índice de volume de vendas reais no varejo                     |Diferença em relação a 12 meses                        |IBGE                                    |
|Vendas de veículos pelas concessionárias - Total               |Segunda diferença em relação a 12 meses                |Fenabrave                               |
|Dívida pública total                                           |Segunda diferença em relação a 12 meses                |Banco Central do Brasil                 |
|Consumo - energia elétrica (GWh)                               |Diferença em relação a 12 meses                        |Eletrobrás                              |
|Preço - pétroleo bruto - Brent (FOB)                           |Diferença em relação a 12 meses                        |Energy Information Administration (EIA) |


# Referências

Adhikari, R., & Agrawal, R. K. (2013). An introductory study on time series modeling and forecasting. arXiv preprint arXiv:1302. 6613.

Baybuza, I. (2018). Inflation forecasting using machine learning methods. Russian Journal of Money and Finance, 77(4), 42–59.

Choi, H., & Varian, H. (2012). Predicting the present with Google Trends. Economic record, 88, 2–9.

Elliott, G., Gargano, A., & Timmermann, A. (2013). Complete subset regressions. Journal of Econometrics, 177(2), 357–373.

Garcia, M. G. P., Medeiros, M. C., & Vasconcelos, G. F. R. (2017). Real-time inflation forecasting with high-dimensional models: The case of Brazil. International Journal of Forecasting, 33(3), 679–693.

Mandalinci, Z. (2017). Forecasting inflation in emerging markets: An evaluation of alternative models. International Journal of forecasting, 33(4), 1082–1104.

Medeiros, M. C., Vasconcelos, G. F. R., Veiga, Á., & Zilberman, E. (2021). Forecasting inflation in a data-rich environment: the benefits of machine learning methods. Journal of Business & Economic Statistics, 39(1), 98–119.

Naccarato, A., Falorsi, S., Loriga, S., & Pierini, A. (2018). Combining official and Google Trends data to forecast the Italian youth unemployment rate. Technological Forecasting and Social Change, 130, 114–122.

Zou, H., Hastie, T., Tibshirani, R., & Others. (2007). On the “degrees of freedom” of the lasso. The Annals of Statistics, 35(5), 2173–2192.


