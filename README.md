# CampaignEvaluation_PSM
Propensity score matching application for marketing case

```
#datathought.blog
#Liubo Bregman 
#liubomyrb@gmail.com
#v1   03/03/2019
#v1.1 06/03/2019 added prediction on sample data, corrected typo
```
Load Libraries
```
library(MatchIt)
library(stats)
library(data.table)
library(cobalt)
library(ggplot2)
```

set.seed(314)
sample_size = 50000

Load data 
```
df1 <- fread('criteo-uplift.csv')
```

Review data and sample
```
str(df1)
summary(df1)

df2 <- df1[sample(nrow(df1), sample_size),]
summary(df2)
```

Simple test if there is a significan difference between sample and populaiton exposures, you may claim there is significan difference between sample and population of test results >1.95(95% confidence), or according to z score distribution table. Perfect sampling means the score goes to 0. >0.01 you can claim there is no difference between sample and populaiton 
```
(mean(df2$exposure) - mean(df1$exposure))/sqrt(sd(df2$exposure)*sd(df1$exposure))
```
=> no significant difference between distribution of sample and distribution of population
```
(mean(df2$conversion) - mean(df1$conversion))/sqrt(sd(df2$conversion)*sd(df1$conversion))
```
=> no significant difference between distribution of sample and distribution of population


PS matching model 
Propensity score formula definition
```
PS_formula <- as.formula('exposure ~ f0 + f1 + f2 + f3 +  f4 + f5 + f6 + f7 + f8 + f9 + f10 + f11')
```
Train the matching model 
```
st <- Sys.time()
match_model = matchit(PS_formula,
                data = df2, 
                method = "nearest",  
                distance = "logit")
Sys.time() - st

summary(match_model)
```
Match the data 
```
df3<- match.data(match_model)
##Review the data 
dim(df3)
summary(df3)
```
Visualization of the maching model 

QQplot
```
plot(match_model)
plot(match_model, type = 'jitter', interactive = FALSE)

```
Review distribution feature by feature distribution.
```
bal.plot(match_model, 'f0')
bal.plot(match_model, var.name = "f1", which = "both",
         type = "histogram", mirror = TRUE)
bal.plot(match_model, "distance", which = "both")
```

Review all features difference in adjusteed and unadjusted samples at once using love chart 
```
love.plot(bal.tab(match_model), threshold = .1)
```

CASUAL EFFECT 
Simple T test
```
with(df3, t.test(conversion ~ exposure))
```
Linear model 
Causal effect defined by coefficient OLS of simple/full model.
simple model 
```
casual_effect_linear <- lm(conversion ~ exposure, df3)
summary(casual_effect_linear)
```
full model 
```
full_formula <-as.formula('conversion ~ exposure + f0 + f1 + f2 +f3 + f4 + f5 +f6 +f7+f8+f9+f10+f11')
casual_effect_linear_full <- lm(full_formula, df3)
summary(casual_effect_linear_full)
```
Testing joint sigfinicance of f0-f11 ( F test )
anova(casual_effect_linear_full, casual_effect_linear)
-> the f0, f11 are joinly significan 

Fitting the model on sample. 
```
df2$fitted_conversion <- (predict(casual_effect_linear_full, df2))
#fitting linear model for binary requires assuming binary probability bellongs to [0,1]
df2$fitted_conversion <- ifelse(df2$fitted_conversion <0,0,df2$fitted_conversion) 
```
Distribution of fitted values
```
qplot() + geom_density(aes(df2$fitted_conversion), fill = 'blue')
```
