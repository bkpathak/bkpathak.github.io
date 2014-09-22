---
layout: post
title: "Human Activity Recognition"
comments: true
permalink: "human-activity-recognition"

### Introduction

This projects uses the accelerometer measurement of 6 people over the time. The data contains the accelerometer measurement mesurement for different type of acctivities and label identifying the quality of the activity.The goal of the project is to create the prediction model to predict the label for the test data sets given.

The project describes each steps taken to build the model and all the preprocessing done in data sets to reach the mode.The data for the project is taken from [1]

### Data Preprocessing and Preparation 

The ```caret``` package is used for this project.Training and test data sets are read and test data is not used until the model is built.

```{r}
library(caret)
ptrain <- read.csv("data/pml-training.csv")
ptest <- read.csv("data/pml-testing.csv")
```

To estimate the *out-of-sample* error, the training set ```ptrain```  is splitted into training and validation set: ```ptrain1``` and ```validation```.  

```{r}
set.seed(1000)
inTrain <- createDataPartition(y = ptrain$classe, p = 0.7, list = FALSE)
ptrain1 <- ptrain[inTrain, ]
validation <- ptrain[-inTrain, ]
```

We will now anlyse the each features of the ```ptrain1``` and remove feauters which do not contribute much to the final model.The features with almost *zero and NA values* are not useful for building model.Also, the feature having almost *zero variance* do not contribute.Other feature variable such as name do not contribute the model but make the prediction model more complex.Now we will analyze the ```ptrain1``` and remove those features and we will apply same to the ```validation``` set also.

```{r}
# remove features with nearly with zero variance using 
nzVar <-  nearZeroVar(ptrain1)
ptrain1 <- ptrain1[, -nzVar]

# Applying same to validation set
validation <- validation[, -nzVar]

# check for features with mostly NAs value and remove if any
mostlyNAs <- sapply(ptrain1 , function(x) mean(is.na(x))) > 0.95
ptrain1 <- ptrain1[, mostlyNAs == F]
validation <- validation[, mostlyNAs == F]

# remove variables which are not relevant for building the model. The first 5 feature varible * X, user_name, raw_timestamp_part_1, raw_timestamp_part_2, cvtd_timestamp* are not relevant.
ptrain1 <- ptrain1[, -(1:5)]
validation <- validation[, -(1:5)]
```

### Model Construction

First, I planned to build the model using ```neural network```.Since the problem is supervised learning and neural nets are good at classification task.To make the model more robust I will use 3 - fold cross validation to build the model.

```{r}
# Use 3-fold cross validation to build the model
controlPara <- trainControl(method = "cv", number = 3, verboseIter = F,)

#build model using ptrain1
modelFit1 <- train(classe ~ ., data = ptrain1, method = "nnet", trControl = controlPara,trace = F)
```


### Model Evaluation
The fitted model is used to predict tje label for our validation set and find the *out-of-sample* error.Confusion matrix is used to compare the output from the our model to the actual labels of the data.

```{r}
# uses modelFit1 to predict classe for our validation data sets
prediction <- predict(modelFit1, newdata = validation)

# Show the confusion matrix with other relevant information to evaluate our model
confusionMatrix(validation$classe, prediction)
```

The performance of this model is less than satisfactory with just 38 % accuracy.The matrix shows there are lots of misclassification. The nnet is feed forward model so the neural network with back propagation may give more accuracy then just feed forward model. 

### Second Model Construction

Let's try to build the second model with ```Random Forest``` and evaluate the accuracy of model on the validation data sets.The model is build with 3-fold CV.

```{r}
# Use 3-fold cross validation to build the model
controlPara <- trainControl(method = "cv", number = 3, verboseIter = F,)

#build model using ptrain1
modelFit2 <- train(classe ~. , data = ptrain1 ,method = "rf", trControl = controlPara )
```

### Second Model Evaulation

The fitted model is used to predict tje label for our validation set and find the *out-of-sample* error.Confusion matrix is used to compare the output from the our model to the actual labels of the data.

```{r}
# uses modelFit1 to predict classe for our validation data sets
prediction <- predict(modelFit2, newdata = validation)

# Show the confusion matrix with other relevant information to evaluate our model
confusionMatrix(validation$classe, prediction)
```

The ```modelFit2``` build with ```Random Forest``` is far better then our previos model ```modelFit1```.The ```out-of-sample``` error for the model is less 1 %. This is the excellent model with 99 % accuarcy. We will use this model as our final model and used this to predict the classes on our test data sets.

### Final Model

Since we have divided our origan training data sets ```ptrain``` into ```ptrain1``` and ```validation``` for finding the *out-of-sample* error.Now we will build the model using complete trianing data sets ```ptrain```.We also apply the same data pre-processing steps on both ```ptrain``` and ```ptest``` data sets.

#### Preprocessing on traing and test data sets 

```{r}
# remove features with nearly with zero variance using 
nzVar <-  nearZeroVar(ptrain)
ptrain <- ptrain[, -nzVar]

# Applying same to test data set
ptest <- ptest[, -nzVar]

# check for features with mostly NAs value and remove if any
mostlyNAs <- sapply(ptrain , function(x) mean(is.na(x))) > 0.95
ptrain <- ptrain[, mostlyNAs == F]
ptest <- ptest[, mostlyNAs == F]

# remove variables which are not relevant for building the model. The first 5 feature varible * X, user_name, raw_timestamp_part_1, raw_timestamp_part_2, cvtd_timestamp* are not relevant.
ptrain <- ptrain[, -(1:5)]
ptest <- ptest[, -(1:5)]
```

#### Building the Final Model
Now we fit the model on the complete training data sets.

```{r}
# Use 3-fold cross validation to build the model
controlPara <- trainControl(method = "cv", number = 3, verboseIter = F)

#build model using ptrain
modelFinal <- train(classe ~. , data = ptrain ,method = "rf", trControl = controlPara )

```

### Prediction on Test Data Set
Now we will use our final model ```modelFinal``` to predict the class of our test data set.We have applied the same data preprocessing methods to test data sets as to training data sets.

The test data sets ```ptest``` contains 20 observations. The prediction output for each observation is written to the separate file.

```{r}
# predict the class of test data
prediction <- predict(modelFinal, newdata = ptest)

# create the character vector for prediction class
preds <- as.character(prediction)

# write prediction for each observation to the files
write_to_files <- function(x) {
    n <- length(x)
    for(i in 1:n) {
        filename <- paste0("output/problem_id_", i, ".txt")
        write.table(x[i], file=filename, quote=F, row.names=F, col.names=F)
    }
}

# create prediction files to submit
write_to_files(preds)
```

1. Velloso, E.; Bulling, A.; Gellersen, H.; Ugulino, W.; Fuks, H. Qualitative Activity Recognition of Weight Lifting Exercises. Proceedings of 4th International Conference in Cooperation with SIGCHI (Augmented Human '13) . Stuttgart, Germany: ACM SIGCHI, 2013. "1"