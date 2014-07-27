# Exercise Quality Prediction

restrictions: 2k words, 5 figures, gh-pages, compiled html version if DL the repo

Using devices such as Jawbone Up, Nike FuelBand, and Fitbit it is now possible to collect a large amount of data about personal activity relatively inexpensively. These type of devices are part of the quantified self movement – a group of enthusiasts who take measurements about themselves regularly to improve their health, to find patterns in their behavior, or because they are tech geeks. One thing that people regularly do is quantify how much of a particular activity they do, but they rarely quantify how well they do it. In this project, my goal is to use data from accelerometers on the belt, forearm, arm, and dumbell of 6 participants. They were asked to perform barbell lifts correctly and incorrectly in 5 different ways.

This document describe the model I developed to predict the "classe" of how each exercise was completed. It has been created using literate programming for maximum reproducibilty and transparency.

## Loading and processing the data

The data for this project come from this source: http://groupware.les.inf.puc-rio.br/har. The training data for this project are available here: https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv. The test data are available here: https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv


```r
#setwd("/Users/yakich/vrygit/machine-learning")

library(lattice)
library(caret)
```

```
## Loading required package: ggplot2
```

```r
library(gtools)

#download data
if(!file.exists("./data")){
    dir.create("./data")
}
fileUrl = "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv"
filepath = "./data/pml-training.csv"
if(!file.exists(filepath)){
    download.file(fileUrl,destfile=filepath,method="curl")
}
fileUrl = "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv"
filepath = "./data/pml-testing.csv"
if(!file.exists(filepath)){
    download.file(fileUrl,destfile=filepath,method="curl")
}

#load data into R, treat "testing set" as validation
training = read.csv("./data/pml-training.csv",
    comment.char = "#",
    header = TRUE,
    sep = ",",
    colClasses = c(NA,"factor","integer",
                   "integer","Date","factor","integer",
        rep(NA,152),"factor"),
    as.is = TRUE,
    na.strings = c("NA","#DIV/0",'""',""))
validation = read.csv("./data/pml-testing.csv",
    comment.char = "#",
    header = TRUE,
    colClasses = c(NA,"factor","integer",
                   "integer","Date","factor","integer",
        rep(NA,152),"factor"),
    as.is = TRUE,
    na.strings = c("NA","#DIV/0",'""',""))

#get rid of empty columns (all NA)
validation = validation[, which(as.numeric(colSums(is.na(training)))==FALSE)]
training = training[, which(as.numeric(colSums(is.na(training)))==FALSE)]
```

Find highly correlated variables:  

```r
nzv = nearZeroVar(training[,8:59],saveMetrics=T)
nzv[nzv$percentUnique < 1,]
```

```
##                      freqRatio percentUnique zeroVar   nzv
## total_accel_belt         1.063        0.1478   FALSE FALSE
## gyros_belt_x             1.059        0.7135   FALSE FALSE
## gyros_belt_y             1.144        0.3516   FALSE FALSE
## gyros_belt_z             1.066        0.8613   FALSE FALSE
## accel_belt_x             1.055        0.8358   FALSE FALSE
## accel_belt_y             1.114        0.7288   FALSE FALSE
## total_accel_arm          1.025        0.3364   FALSE FALSE
## total_accel_dumbbell     1.073        0.2191   FALSE FALSE
## total_accel_forearm      1.129        0.3567   FALSE FALSE
```

Results seem to suggests getting rid of total_* predictors, which makes sense as these are likely derived values.  

```r
dropcols = grepl("total",colnames(training))
training = training[, dropcols == FALSE]
validation = validation[, dropcols == FALSE]
```

## Create Random Forest model
Other models were attempted with cross validation within the testing set, but RF seems best. Now creating a Random Forest model with 4 folds on a random set of 4 out of the 6 users in the training set.  


```r
#initialize model results (there must be a more elegant way)
results = data.frame(user_combo = 0,num_folds = 0, method = "rf",
            users = paste(0,0,0,0),
            in.accuracy = 0.0,out.accuracy = 0.0
            )
four.user.combos = dim(subtest.user.index)[1]
uniqueusers = unique(training$user_name)
numtrainusers = length(uniqueusers)

#get all combinations of 30% of the users (index only)
subtest.user.index = combinations(numtrainusers,round(numtrainusers*.3))

#for(j in 3:5){ #try different numbers of folds in the RF cross validation
#    for(i in 1:four.user.combos){ #try different combinations of users
        
        #choose a random partition of test users from the training set
        i = sample.int(dim(subtest.user.index)[1],1)
        
        #set the number for cross validation folds to 4
        j = 4

        #use the combination list as an index to factor levels for user names
        in.test = 
            training$user_name == levels(training$user_name)[subtest.user.index[i,1]] | 
            training$user_name == levels(training$user_name)[subtest.user.index[i,2]]
        train.sub = training[-in.test,c(5,7:56)]
        test.sub = training[in.test,c(5,7:56)]
        
        mod = train(classe~., data = train.sub, method="rf",
                    trControl = trainControl(method="cv"),number=j)
```

```
## Loading required package: randomForest
## randomForest 4.6-10
## Type rfNews() to see new features/changes/bug fixes.
## 
## Attaching package: 'e1071'
## 
## The following object is masked from 'package:gtools':
## 
##     permutations
```

```r
        in.pred = predict(mod,train.sub)
        out.pred = predict(mod,newdata = test.sub)

        users = unique(training$user_name[-in.test])
        users = paste(users[1],users[2],users[3],users[4])
    
        results = rbind(results,
                        data.frame(user_combo = i,
                            num_folds = j,
                            method = "rf",
                            users = users,
                            in.accuracy = sum(train.sub$classe == in.pred) / length(in.pred),
                            out.accuracy = sum(test.sub$classe == out.pred) / length(out.pred)
                    ))
        #print(results)
#    }
#}
print(results[2:dim(results)[1],])
```

```
##   user_combo num_folds method                         users in.accuracy
## 2          6         4     rf carlitos pedro adelmo charles           1
##   out.accuracy
## 2            1
```

## Examine error (confusion matrix)
In sample error is zero!
This describes how well the model predicts within the training set of the random 4 users.

```r
    confusionMatrix(in.pred,train.sub$classe)$table
```

```
##           Reference
## Prediction    A    B    C    D    E
##          A 5579    0    0    0    0
##          B    0 3797    0    0    0
##          C    0    0 3422    0    0
##          D    0    0    0 3216    0
##          E    0    0    0    0 3607
```

Out of sample error is zero!  
This describes how well the model predicts for the other 2 users.

```r
    confusionMatrix(out.pred,test.sub$classe)$table
```

```
##           Reference
## Prediction    A    B    C    D    E
##          A 1733    0    0    0    0
##          B    0 1435    0    0    0
##          C    0    0 1032    0    0
##          D    0    0    0 1128    0
##          E    0    0    0    0 1320
```

looks perfect, now to try it on the validation set (once-and-only-once) to answer and submit

## Predict answers for the 20 problems


```r
validation.sub = validation[,c(5,7:56)]
answers = data.frame(problem_id = 1:20,pred = NA)
for(i in 1:20){
    val = predict(mod,newdata=validation.sub[validation$problem_id == as.character(i),])
    answers[i,2] = levels(val)[val]
}
print(answers)
```

```
##    problem_id pred
## 1           1    B
## 2           2    A
## 3           3    B
## 4           4    A
## 5           5    A
## 6           6    E
## 7           7    D
## 8           8    B
## 9           9    A
## 10         10    A
## 11         11    B
## 12         12    C
## 13         13    B
## 14         14    A
## 15         15    E
## 16         16    E
## 17         17    A
## 18         18    B
## 19         19    B
## 20         20    B
```

## Write answers to files  

```r
pml_write_files = function(x,outdir){
  n = length(x)
  for(i in 1:n){
    filename = paste0(outdir,"/problem_id_",i,".txt")
    write.table(x[i],file=filename,quote=FALSE,row.names=FALSE,col.names=FALSE)
  }
}

if(!file.exists("./answers")){
    dir.create("./answers")
}
 
pml_write_files(answers[,2],"./answers")
```


Success!

