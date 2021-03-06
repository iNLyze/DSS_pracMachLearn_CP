# Monitoring Your Weight Lifting Excercise
iNLyze  
Saturday, July 18, 2015  

This report was created as part of the module "Practical Machine Learning" of the Data Science Specialization on Coursera. The goal is to predict how well a test subject performed a weight lifting excercise using machine learning. The movement of the test subjects was recorded using wearable sensors. The underlying data was published by [Velloso et al. (2013)](http://groupware.les.inf.puc-rio.br/har). 



# Loading libraries and data

The original data was split into a [testing](https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv) and [training](https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv) set, downloaded and imported using *read.csv()*.
Note: Data exploration revealed several kinds of missing data which were omitted (see also *na.strings()* argument).




```r
library(caret)
```

```
## Loading required package: lattice
## Loading required package: ggplot2
```

```r
library(dplyr)
```

```
## 
## Attaching package: 'dplyr'
## 
## The following objects are masked from 'package:stats':
## 
##     filter, lag
## 
## The following objects are masked from 'package:base':
## 
##     intersect, setdiff, setequal, union
```

```r
if (!require(randomForest))
        install.packages("randomForest")
```

```
## Loading required package: randomForest
## randomForest 4.6-10
## Type rfNews() to see new features/changes/bug fixes.
## 
## Attaching package: 'randomForest'
## 
## The following object is masked from 'package:dplyr':
## 
##     combine
```






```r
path.testing <- paste0(my.workdir, "\\pml-testing.csv")
path.training <- paste0(my.workdir, "\\pml-training.csv")

# Read the data
training.raw <- read.csv(file = path.training, header = T, sep = ",", na.strings=c("NA","#DIV/0!",""))
testing.raw <- read.csv(file = path.testing, header = T, sep = ",", na.strings=c("NA","#DIV/0!",""))
```

# Data splitting for cross-validation

The downloaded testing set is saved for evaluating the model on 20 test cases. Hence, a second testing set, called *valset* is split off the training set for cross-validation. A third set, called *modelset* is split off as 10% of the remaining training set.  


```r
set.seed(34542134)
inValSet <- createDataPartition(training.raw$classe, p=0.15, list = F )
valset.raw <- training.raw[inValSet, ]
training.raw <- training.raw[-inValSet, ]
inModelSet <- createDataPartition(training.raw$classe, p=0.1, list = F)
modelset.raw <- training.raw[inModelSet, ]
training.raw <- training.raw[-inModelSet, ]
```
Cross-validation and out-of sample error are assessed using *valset*. The *modelset* is used for model selection. 

# Preprocessing for dimensionality reduction 

## Weeding out non-informative variables

The original 160 variables are not all informative for model building, because they have near zero variance. Only those variables are kept which are non-zero in all four data sets, i.e. *training*, *testing*, *valset* and *modelset*. The columns for numbering, user names and timestamps are omitted as well. This leaves 56 numeric variables.


```r
# Dump near zero variables from training set
nzvMetrics.training <- nearZeroVar(x = training.raw, saveMetrics = T)
nzvIndex.training <- nearZeroVar(x = training.raw, saveMetrics = F)

# Find nzv's from test set as well - no point in keeping variables I can't test for
nzvMetrics.testing <- nearZeroVar(x = testing.raw, saveMetrics = T)
nzvIndex.testing <- nearZeroVar(x = testing.raw, saveMetrics = F)

nzvMetrics.valset <- nearZeroVar(x = valset.raw, saveMetrics = T)
nzvIndex.valset <- nearZeroVar(x = valset.raw, saveMetrics = F)

nzvMetrics.modelset <- nearZeroVar(x = modelset.raw, saveMetrics = T)
nzvIndex.modelset <- nearZeroVar(x = modelset.raw, saveMetrics = F)

#Dump all vars which are zero anywhere
nonZero <- !nzvMetrics.training$zeroVar & !nzvMetrics.testing$zeroVar & !nzvMetrics.valset$zeroVar & !nzvMetrics.modelset$zeroVar

# Get rid of numbering column, user names (col 2) and cvtd timestamps (col 5) - then all vars are numeric
nonZero[1] <- FALSE
nonZero[2] <- FALSE
nonZero[5] <- FALSE

# Create training, testing and valset for actual model building
training <- tbl_df(training.raw[, nonZero])
testing <- tbl_df(testing.raw[, nonZero])
valset <- tbl_df(valset.raw[, nonZero])
modelset <- tbl_df(modelset.raw[, nonZero])

# Col index to classe
classeIndex <- sum(nonZero)
classe.train <- training.raw$classe
classe.val <- valset.raw$classe
classe.modelset <- modelset.raw$classe

# Freeing RAM
rm(training.raw)
rm(valset.raw)
```

## Reducing dimensionality to principle components

Since movements are carried out in three spatial dimensions one could expect variables to be correlated. The following chunk tests for this:


```r
# Find strongly correlated variables
M <- abs(cor(training[, -classeIndex]))
diag(M) <- 0
which(M > 0.8, arr.ind = T)
```

```
##                  row col
## yaw_belt           6   4
## total_accel_belt   7   4
## accel_belt_y      12   4
## accel_belt_z      13   4
## accel_belt_x      11   5
## magnet_belt_x     14   5
## roll_belt          4   6
## roll_belt          4   7
## accel_belt_y      12   7
## accel_belt_z      13   7
## pitch_belt         5  11
## magnet_belt_x     14  11
## roll_belt          4  12
## total_accel_belt   7  12
## accel_belt_z      13  12
## roll_belt          4  13
## total_accel_belt   7  13
## accel_belt_y      12  13
## pitch_belt         5  14
## accel_belt_x      11  14
## gyros_arm_y       22  21
## gyros_arm_x       21  22
## magnet_arm_x      27  24
## accel_arm_x       24  27
## magnet_arm_z      29  28
## magnet_arm_y      28  29
## accel_dumbbell_x  37  31
## accel_dumbbell_z  39  32
## gyros_dumbbell_z  36  34
## gyros_forearm_z   49  34
## gyros_dumbbell_x  34  36
## gyros_forearm_z   49  36
## pitch_dumbbell    31  37
## yaw_dumbbell      32  39
## gyros_forearm_z   49  48
## gyros_dumbbell_x  34  49
## gyros_dumbbell_z  36  49
## gyros_forearm_y   48  49
```
One observes lots of correlated variables. Thus, principle component analysis is carried out. At the same time data is BoxCox transformed, normalized and missing data is imputed using k-nearest-neighbours. All transformations are applied equally to the *training*, *testing* and *valset* equally. The classe variable is removed from them using *classeIndex* created above.


```r
#PCA needs further transformations, all carried out by preProcess

# BoxCox, Normalize and impute NAs, PCA
normalized.training <- preProcess(x=as.data.frame(training[,-classeIndex]), method = c("BoxCox", "center", "scale", "knnImpute", "pca"))
training <- predict(normalized.training, newdata=as.data.frame(training[, -classeIndex]))

# Apply the same trafo's to test set, validation set and modelset
testing <- predict(normalized.training, newdata=as.data.frame(testing[, -classeIndex]))

valset <- predict(normalized.training, newdata=as.data.frame(valset[, -classeIndex]))

modelset <- predict(normalized.training, newdata=as.data.frame(modelset[, -classeIndex]))
```

# Model building

The objective is a 5-class classification problem. Four different machine learning approaches are selected: A Random Forest model, a Naive Bayesian learner, a Support Vector Machine and a QDA classifier. It turned out that these have vastly different computation and memory needs. The out of sample error for each is cross-validated against *valset* and put into perspective. 10-fold resampling settings are chosen, so the models can be compared by resample() lateron. 



```r
file.rf <- paste0(my.workdir, "\\model.rf.Rds")

if (file.exists(file.rf)) {
        model.rf <- readRDS(file.rf)
} else {
        mtryGrid <- expand.grid(mtry = 20)
        ctrl <- trainControl(method = 'cv')
        model.rf <- train(classe.modelset ~ ., 
                          data = modelset, 
                          method = "rf", 
                          importance = TRUE, 
                          ntree = 10, 
                          tuneGrid = mtryGrid, 
                          trControl = ctrl )
        saveRDS(model.rf, file.rf)
}

prediction.rf <- predict(model.rf, valset)

cM.rf <- confusionMatrix(prediction.rf, classe.val)
```


```r
file.nb <- paste0(my.workdir, "\\model.nb.Rds")

if (file.exists(file.nb)) {
        model.nb <- readRDS(file.nb)
} else {
        grid <- expand.grid(fL = 0, usekernel = TRUE)
        ctrl <- trainControl(method = 'cv', 
                             number = 10)
        model.nb <- train(classe.modelset ~ .,
                          data = modelset, 
                          method="nb", 
                          trControl = ctrl, 
                          tuneGrid = grid) 
        saveRDS(model.nb, file.nb)
}

prediction.nb <- predict(model.nb, valset)

cM.nb <- confusionMatrix(prediction.nb, classe.val)
```


```r
file.svm <- paste0(my.workdir, "\\model.svm.Rds")

if (file.exists(file.svm)) {
        model.svm <- readRDS(file.svm)
} else {
        ctrl <- trainControl(method = 'cv', 
                             number = 10)
        model.svm <- train(classe.modelset ~ ., 
                           data = modelset, 
                           method = "svmRadial", 
                           trControl = ctrl) 
        saveRDS(model.svm, file.svm)
}

prediction.svm <- predict(model.svm, valset)

cM.svm <- confusionMatrix(prediction.svm, classe.val)
```


```r
file.qda <- paste0(my.workdir, "\\model.qda.Rds")

if (file.exists(file.qda)) {
        model.qda <- readRDS(file.qda)
} else {
        ctrl <- trainControl(method = 'cv', 
                             number = 10)
        model.qda <- train(classe.modelset ~ ., 
                           data = modelset, 
                           method = "qda", 
                           trControl = ctrl)   
        saveRDS(model.qda, file.qda)
}

prediction.qda <- predict(model.qda, valset)

cM.qda <- confusionMatrix(prediction.qda, classe.val)
```


# Cross-validation and out-of-sample error

The confusion matrices collected above are put into a list. 


```r
cM.list <- list(RF = cM.rf$table, 
                NB = cM.nb$table, 
                SVM = cM.svm$table, 
                QDA = cM.qda$table)
cM.list
```

```
## $RF
##           Reference
## Prediction   A   B   C   D   E
##          A 729  60  52  31  33
##          B  42 426  50  29  48
##          C  38  41 378  63  37
##          D  24  20  14 326  49
##          E   4  23  20  34 375
## 
## $NB
##           Reference
## Prediction   A   B   C   D   E
##          A 558  62  69  28  22
##          B  69 319  54  35  83
##          C 133 103 324  95  47
##          D  61  33  34 286  60
##          E  16  53  33  39 330
## 
## $SVM
##           Reference
## Prediction   A   B   C   D   E
##          A 711  43  25  24  26
##          B  17 398  58   7  38
##          C  54  73 393  55  37
##          D  50  41  35 366  41
##          E   5  15   3  31 400
## 
## $QDA
##           Reference
## Prediction   A   B   C   D   E
##          A 710  43  27  26  28
##          B  17 398  57   6  40
##          C  55  74 392  54  36
##          D  50  41  35 366  42
##          E   5  14   3  31 396
```
It turns out all four models perform rather well, but there are differences. 

In order to make statistical statements about model performance 10 resamples are created of each model and summarized below. 


```r
resamps <- resamples(list(RF = model.rf,
                          NB = model.nb, 
                          SVM = model.svm, 
                          QDA = model.qda))
summary(resamps)
```

```
## 
## Call:
## summary.resamples(object = resamps)
## 
## Models: RF, NB, SVM, QDA 
## Number of resamples: 10 
## 
## Accuracy 
##       Min. 1st Qu. Median   Mean 3rd Qu.   Max. NA's
## RF  0.7143  0.7299 0.7425 0.7419  0.7492 0.7798    0
## NB  0.5595  0.5753 0.6036 0.6072  0.6364 0.6548    0
## SVM 0.7349  0.7587 0.7695 0.7713  0.7794 0.8133    0
## QDA 0.7048  0.7616 0.7731 0.7688  0.7844 0.7964    0
## 
## Kappa 
##       Min. 1st Qu. Median   Mean 3rd Qu.   Max. NA's
## RF  0.6401  0.6588 0.6734 0.6733  0.6824 0.7201    0
## NB  0.4429  0.4646 0.5007 0.5044  0.5413 0.5629    0
## SVM 0.6652  0.6953 0.7086 0.7110  0.7224 0.7639    0
## QDA 0.6296  0.6993 0.7137 0.7081  0.7274 0.7441    0
```

```r
# Create a dotplot for hypothesis testing
trellis.par.set(caretTheme())
dotplot(resamps, metric = "Accuracy")
```

![](Monitoring_weight_lifting_files/figure-html/model_accuracy-1.png) 

The best three models are SVM, RF and QDA, while NB is performing markedly more poorly at the chosen settings for this dataset.  

# Learning efficiency 

Since the model accuracies are rather close for the three best models one could ask the question, which model provides the "biggest bang for the buck" as far as runtime complexity is concerned. The computation times and memory usage are extracted and compared for each model. A *bang.for.buck* parameter is defined as the ratio of model accuracy over computation time. 


```r
# Compare runtime complexity in relation to accuracy
accuracies <- rbind(RF = cM.rf$overall[1], 
                    NB = cM.nb$overall[1], 
                    SVM = cM.svm$overall[1],
                    QDA = cM.qda$overall[1])
timings <- resamps$timings[1]

bang.for.buck <- accuracies/timings
bang.for.buck <- cbind(bang.for.buck, rownames(bang.for.buck))
names(bang.for.buck) <- c("bang", "method")
bang.for.buck <- arrange(bang.for.buck, desc(bang))
qplot(x = method, y = bang, data=bang.for.buck,
      geom = "bar",
      stat="identity", 
      fill = method, 
      xlab = "ML Method", 
      ylab = "Accuracy / Computation Time")
```

![](Monitoring_weight_lifting_files/figure-html/model_comparison-1.png) 

```r
# Compare memory usage of each model

memuse <- as.data.frame(rbind(
                RF = as.numeric(object.size(model.rf)),
                NB = as.numeric(object.size(model.nb)), 
               SVM = as.numeric(object.size(model.svm)), 
               QDA = as.numeric(object.size(model.qda))
               ))
names(memuse) <- c("Memory [Bytes]")
memuse
```

```
##     Memory [Bytes]
## RF          925652
## NB         2065964
## SVM        1520208
## QDA        1519912
```

# Selecting the final model

Looking at the accuracy/computation time figure RF wins hands down. We also have a clear looser, naive Bayesian learner, which achieved the lowest accuracy, but used up the most memory. 
For the final model the random forest approach is picked, since it computes fast (at the parameters chosen) and achieves a very large accuracy. A bit of fine tuning is done with regard to chaning the resampling technique to *oob* and increasing the number of trees to 100. 



```r
file.final <- paste0(my.workdir, "\\model.final.Rds")

if (file.exists(file.final)) {
        model.final <- readRDS(file.final)
} else {
        mtryGrid <- expand.grid(mtry = 20)
        ctrl <- trainControl(method = 'oob')
        model.final <- train(classe.train ~ ., 
                          data = training, 
                          method = "rf", 
                          importance = TRUE, 
                          ntree = 100, 
                          tuneGrid = mtryGrid, 
                          trControl = ctrl )     
        saveRDS(model.final, file.final)       
}

prediction.final <- predict(model.final, valset)

cM.final <- confusionMatrix(prediction.final, classe.val)

cM.final
```

```
## Confusion Matrix and Statistics
## 
##           Reference
## Prediction   A   B   C   D   E
##          A 826   5   4   0   1
##          B   6 556  13   1   3
##          C   3   7 491  25   4
##          D   2   0   6 453   2
##          E   0   2   0   4 532
## 
## Overall Statistics
##                                          
##                Accuracy : 0.9701         
##                  95% CI : (0.9633, 0.976)
##     No Information Rate : 0.2841         
##     P-Value [Acc > NIR] : < 2e-16        
##                                          
##                   Kappa : 0.9622         
##  Mcnemar's Test P-Value : 0.01255        
## 
## Statistics by Class:
## 
##                      Class: A Class: B Class: C Class: D Class: E
## Sensitivity            0.9869   0.9754   0.9553   0.9379   0.9815
## Specificity            0.9953   0.9903   0.9840   0.9959   0.9975
## Pos Pred Value         0.9880   0.9603   0.9264   0.9784   0.9888
## Neg Pred Value         0.9948   0.9941   0.9905   0.9879   0.9958
## Prevalence             0.2841   0.1935   0.1745   0.1640   0.1840
## Detection Rate         0.2804   0.1887   0.1667   0.1538   0.1806
## Detection Prevalence   0.2838   0.1965   0.1799   0.1572   0.1826
## Balanced Accuracy      0.9911   0.9829   0.9696   0.9669   0.9895
```

# Conclusion

The random forest model was finally run using oob resampling and number of trees 100. Exploring more trees will improve accuracy slightly, but at high computation cost (data not shown). The total training with above settings took 20.36 seconds to achieve an accuracy of 97 +/- (96.3, 97.6)%. 
