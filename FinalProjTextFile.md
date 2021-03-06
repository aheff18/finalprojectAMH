title: "Untitled"
output: html_document
---

```{r}
#install.packages("haven")
library(package = "haven")
pacman::p_load(survival, survminer, tidyverse, readr, lmtest, table1, ggplot2, odds.n.ends, dplyr, blorr, lmtest, car)

```


## Importing the data

```{r chunk2}
# 2015 Behavioral Risk Factor Surveillance Survey data
temp <- tempfile(fileext = ".zip")
download.file(url  = "https://www.cdc.gov/brfss/annual_data/2020/files/LLCP2020XPT.zip", destfile = temp)
BRFSS_2020 <- read_xpt(file = temp)
```
``` {r}
BRFSS <- BRFSS_2020 %>%
  select('_BMI5CAT','_AGE_G', SEXVAR,'_MICHD', '_BMI5')

BRFSS$hdbinary<-ifelse( BRFSS$'_MICHD'== 2| BRFSS$'_MICHD' =="Did not report having MI or CHD", 0, ifelse(BRFSS$'_MICHD'== 1|BRFSS$'_MICHD'=="Reported having MI or CHD", 1, NA))
BRFSS$hdbinary <-factor(BRFSS$hdbinary, levels=c(0,1), labels=c("No Heart Disease", "Heart Disease"))
table(BRFSS$hdbinary, useNA="always")
BRFSS<-BRFSS[which(!is.na(BRFSS$hdbinary)),]

BRFSS$bmi <- BRFSS$'_BMI5CAT'
BRFSS$bmi<- ifelse(BRFSS$bmi == 1, "Underweight", ifelse(BRFSS$bmi == 2, "Normal Weight", ifelse(BRFSS$bmi == 3, "Overweight", ifelse(BRFSS$bmi == 4, "Obese", NA))))
BRFSS<-BRFSS[which(!is.na(BRFSS$bmi)),]
class(BRFSS$bmi)
BRFSS$bmi<-as.factor(BRFSS$bmi)

BRFSS$agecode <- ifelse(BRFSS$'_AGE_G' == 1, "18-24",
                      ifelse(BRFSS$'_AGE_G' == 2, "25-34",
                      ifelse(BRFSS$'_AGE_G' == 3, "35-44",
                      ifelse(BRFSS$'_AGE_G' == 4,"45-54", 
                      ifelse(BRFSS$'_AGE_G' == 5, "55-64", 
                      ifelse(BRFSS$'_AGE_G' == 6, "65 or older", NA))))))
BRFSS<-BRFSS[which(!is.na(BRFSS$agecode)),]
BRFSS$agecode <-as.factor(BRFSS$agecode)
class(BRFSS$agecode)

BRFSS$sexbinary<-ifelse( BRFSS$SEXVAR == 1| BRFSS$SEXVAR =="Male", 1, ifelse(BRFSS$SEXVAR == 2|BRFSS$SEXVAR =="Female", 0, NA))
BRFSS$sexbinary <-factor(BRFSS$sexbinary, levels=c(0,1), labels=c("Female", "Male"))

BRFSS<-drop_na(BRFSS)
library(table1)
brfsstable1 <- table1(~ sexbinary + agecode + bmi | hdbinary, data=BRFSS)
brfsstable1

library(odds.n.ends)
bmiLogit <- glm(hdbinary ~bmi + sexbinary + agecode, data=BRFSS, family="binomial")
  summary(bmiLogit)
odds.n.ends(bmiLogit)
```
```{r}
#Test assumptions
#Box Tidwell not needed to test for linearity assumption, since predictor variable is not continuous

#Cook's Distances to check for influential data

plot(bmiLogit, which=4, id.n=5, col="red") 

#Cook's D cutoff=0.0015 Or you can define use the rules described in the lecture
cutoff <- (4/(357637))
obs_no <- as.data.frame(cooks.distance(bmiLogit)) %>%
  mutate(obs_no=row_number()) %>%
  filter(`cooks.distance(bmiLogit)` > cutoff)
bmiLogit.modex <- update(bmiLogit,subset=c(-obs_no$obs_no))
summary(bmiLogit.modex)
compareCoefs(bmiLogit, bmiLogit.modex) 

#from the comparison results above, I conclude that removing influential data largely affect the coefficients

#Model Fit Assumption
#Various pseudo R squares, log likelihood, deviance, AIC, BIC
blr_model_fit_stats(bmiLogit)
#Hosmer lemeshow goodness of fit test: a significant p value indicates a bad fit
blr_test_hosmer_lemeshow(bmiLogit)
#P<0.05, so this assumption is violated 

#Multicollinearity assumption tested using VIF
vif(bmiLogit)

```
