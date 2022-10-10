# Practical Machine Learning - Coursera
---
title: "Practical Machine Learning - Coursera"
author: "Rafi Amir-ud-Din"
date: "October 9, 2022"
output: html_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
library(lattice)
library(ggplot2)
library(caret)
library(rpart)
library(rpart.plot)
library(corrplot)
library(rattle)
library(randomForest)
library(RColorBrewer)
library(caret)
library(randomForest)

```

# Practical Machine Learning Project

## Background

Using devices such as Jawbone Up, Nike FuelBand, and Fitbit it is now possible to collect a large amount of data about personal activity relatively inexpensively. These type of devices are part of the quantified self movement – a group of enthusiasts who take measurements about themselves regularly to improve their health, to find patterns in their behavior, or because they are tech geeks. One thing that people regularly do is quantify how much of a particular activity they do, but they rarely quantify how well they do it. In this project, your goal will be to use data from accelerometers on the belt, forearm, arm, and dumbell of 6 participants. They were asked to perform barbell lifts correctly and incorrectly in 5 different ways. More information is available from the website here: http://web.archive.org/web/20161224072740/http:/groupware.les.inf.puc-rio.br/har (see the section on the Weight Lifting Exercise Dataset).

## Data

The training data for this project are available here:

https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv

The test data are available here:

https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv

The data for this project come from this source: http://web.archive.org/web/20161224072740/http:/groupware.les.inf.puc-rio.br/har. If you use the document you create for this class for any purpose please cite them as they have been very generous in allowing their data to be used for this kind of assignment.

## Analysis

We load the two data sets and name them as trn and tst, corresponding with training and testing. 

```{r load data, warning=FALSE, message=FALSE, echo=TRUE}
trn = read.csv("https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv",na.strings=c("NA","#DIV/0!",""))
tst = read.csv("https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv",na.strings=c("NA","#DIV/0!",""))
# Data dimensions
dim(trn)
dim(tst)
```

```{r header, warning=FALSE, message=FALSE, eval= FALSE}
# First look at the data
head(trn)
head(tst)
```

## Splitting the data into training-validation groups

Distinction between validation data and test data is generally fuzzy in the context of developing machine learning (ML) models. Although, both data sets broadly serve  the same objectives, there are some differences between the two. The validation data set generally carries the same labels so that data scientist can better train the model. Tuning the parameters of the model using the validation data set is essentially the part of training process. However, the test data set confirms that the trained model actually works. 

Following the standard practice, we split the training data set `pml-training.csv` into *sub*-training and validation data sets according to 70:30 ratio. Finally, we shall use the best predicted model on the `pml-testing.csv` dataset.

```{r cross-validation, warning=FALSE, message=FALSE, echo=TRUE}
# load packages

# Index for training dataset (70%) and testing dataset (30%) 
# from the pml-training data set
set.seed(1)
dp = createDataPartition(y=trn$classe, p=0.7, list=FALSE)
# training dataset
train_ds = trn[dp,]
# testing dataset
valid_ds = trn[-dp,]
```


## Removing variables with low explanatory power and large missing values

Training and testing data consist of 160 variables. As many predictors have little explanatory powers, and some other variables have large number of missing values, we remove both types of variables from the data set. The nearZeroVar command greatly facilitates the process of removing variables with little explanatory power. We drop all variables in which the ratio of missing values is greater than 95%.


```{r clean data, warning=FALSE, message=FALSE, echo=TRUE}
nearzero <- nearZeroVar(train_ds)

train_ds <- train_ds[ , -nearzero]
valid_ds  <- valid_ds [ , -nearzero]

dim(train_ds)
dim(valid_ds)


missing <- sapply(train_ds, function(x) mean(is.na(x))) > 0.95
train_ds <- train_ds[ , missing == FALSE]
valid_ds  <- valid_ds [ , missing == FALSE]

dim(train_ds)
dim(valid_ds)
```

Removing the identification information from the data sets
```{r}
train_ds = train_ds[, -c(1:6)]
valid_ds  = valid_ds[,  -c(1:6)]
```

## Pairwise correlation matrix

A picture is worth one thousand words! Pairwise correlation generally give useful information about the covariance structure of the data set. Pairwise correlation among all the numerical variables in the training data set suggest generally low pairwise correlation, suggesting limited scope of using dimension reduction techniques.

```{r corrplot}
library(corrplot)
train_ds1 = train_ds[,-c(53)]
corr_matrix <- cor(train_ds1)
corrplot(corr_matrix, order = "FPC", method = "square", type = "upper", tl.cex = 0.5, tl.col = rgb(0, 0, 0))
```


# Predictive models

We shall use three most frequently used methods, namely, decision tree, support vector machine (SVM), generalized boosted model (GBM), and random forest model to predict five human activities (A: sitting-down, B: standing-up, C: standing, D: walking, and E: sitting) collected on eight hours of activities of four healthy subjects.

## Decision tree

We use caret package to fit decision tree model.


```{r Decision Tree}
set.seed(1)
ftdt <- rpart(classe ~ ., data = train_ds, method="class")
fancyRpartPlot(ftdt)
```
### Prediction of decision tree model on validation data

We estimate how well the trained decision model data fits the test data set. 

```{r predict decision tree on validation data}
prdt <- predict(ftdt, newdata = valid_ds, type="class")
cmdt <- confusionMatrix(prdt, as.factor(valid_ds$classe))
cmdt
```


```{r predictive accuracy}
plot(cmdt$table, col = cmdt$byClass, main = paste("Decision Tree: Predictive Accuracy =", round(cmdt$overall['Accuracy'], 3)))
```

## Support Vector Machine

```{r}
tsvm   <- trainControl(method = "cv", number = 2)
ftsvm <- train(classe~., data=train_ds, method="svmLinear", trControl = tsvm, tuneLength = 5, verbose = F)
```
### Prediction of SVM on validation data

```{r}
prsvm <- predict(ftsvm, valid_ds)
cmsvm <- confusionMatrix(prsvm, factor(valid_ds$classe))
cmsvm
```



## Generalized Boosted Model (GBM)

```{r GBM}
set.seed(1)
tgbm   <- trainControl(method = "repeatedcv", number = 2, repeats = 1)
ftgbm  <- train(as.factor(classe) ~ ., data = train_ds, method = "gbm", trControl = tgbm, verbose = FALSE)
ftgbm$finalModel
```



### Prediction of GBM on validation data

```{r gbm test}
prgmb <- predict(ftgbm, newdata = valid_ds)
cmgbm <- confusionMatrix(prgmb, factor(valid_ds$classe))
cmgbm
```

## Random forest model

```{r Random Forest}
set.seed(1)
ftrf = randomForest(as.factor(classe) ~ ., data = train_ds, ntree = 250)
plot(ftrf, main = "Random Forest")
```

### Prediction of Random Forest on validation data

```{r Random Forest Prediction}
prrf = predict(ftrf, valid_ds[,-ncol(valid_ds)])
# Get results (Accuracy, etc.)
confusionMatrix(prrf, as.factor(valid_ds$classe))
```

The accuracy of the random forest model (99.39%) is significantly higher than the decision tree accuracy (75.04%) and support vector machine (SVM) (78.16), but  Generalized Boosting Model (GBM) comes close to random forest model with 95.85% accuracy. The **out-of-sample error** is 24.96%, 21.84%, 4.15%, and 0.61% in decision tree, SVM, GBM, and Random Forest models respectively.



# Applying ML algorithms to the 20 test cases

Applying RF ML algorithm, which is the best model among the four algorithms with 99.39% accuracy to the 20 test cases predicts the following classes:

```{r confusion matrix with RF}
# Random forest model
predict(ftrf, tst)
```

# Conclusion
Both Random Forest model and Gradient Boosting Model could be the best predictive models. However, given a significantly smaller amount of time Random Forest model takes to train compared to the Generalized Boosting Model, Random Forest model is the best model to predict 5 classes (sitting-down, standing-up, standing, walking, and sitting) in HAR data set.
