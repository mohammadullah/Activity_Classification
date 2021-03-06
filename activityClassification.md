---
title: "Predicting Activity Class from Wearable Sensors"
author: "Lisa Marie Pritchett"
date: 'January 2, 2018'
output:
  html_document:
    keep_md: yes
---




Wearable sensors are popular for tracking amount of activity throughout the day, but can these sensors also be used to help users improve activity techniques? This analysis examines the feasibility of creating a product that gives online feedback about activity class using machine learning and accelerometers. I also completed an exploratory analysis of the data, it is published as a separate report [here](http://rpubs.com/lisamariepritchett/345942).

To show that this idea is feasable, I examine data from a study where participants performed bicep curls while accelerometers were attached to their body and equipment. They each completed one set of 10 standard Unilateral Dumbbell Biceps Curls in 5 different fashions:

__Dumbbell Lifting Classes:__

- Class A: Exactly according to the specification 
- Class B: Throwing the elbows to the front
- Class C: Lifting the dumbbell only halfway 
- Class D: Lowering the dumbbell only halfway 
- Class E: Throwing the hips to the front

The accelerometers were mounted on the user's belt, arm, forearm, and dumbell. The data are the readings from the accelerometers, and are described by the authors of the original study as follows:

_"For data recording we used four 9 degrees of freedom Razor inertial measurement units (IMU), which provide three-axes acceleration, gyroscope and magnetometer data at a joint sampling rate of 45 Hz"_(Vellosso, 2013)

## Summary

### Part 1. Predicting Activity Class from Accelerometer Data

Here I train a random forest machine learning algorithm to predict activity class using the accelerometer data.
I use 60% of the data as training and 40% as cross-validation, with the separate set of 20 for testing. 
I find in- and out-of-sample error rates under 1%. 
The algorithm takes almost 5 minutes to train even with parallel processing enabled. This training time may be a barrier to use, so I explore ways of speeding training time.

I train on the Principal Components, which cuts the training time in half, but also results in higher error rates (~3%).

 I think the algorithm is finding a different solution for each user and that the user-specific solutions are optimal. I find that if I train the algorithm for each user separately, accuracy is still above 99% and each training completes in under 30 seconds. This demonstrates the feasibility of this product, as long as the users can train the algorithm before using (each time?).

### Part 2. New Users, Feature in Development:

It would be nice if we could train the algorithm once and then apply that model to new users. This would mean that users could purchase the product off the shelf and use it without consulting with an expert or conducting their own training session first.

I test the feasability of this idea by training the algorithm on data from 5 of the 6 users and then testing it on the 6th. 
In this case I find poor performance when using the raw data (average 37% accuracy). 
I find that centering the data by user before training improves performance (average 53% accuracy). 
Peformance is very test-user dependent, varying between barely better than chance performance (16.67%) for 2 users and nearly 70% accuracy for another. 
The poor performance is likely due to overfitting to the very limited number of training-users. There is also some missing information that if included would probably improve peformance. A larger sample of training users should be used and information about heights, weights, and arm lengths should be collected to further develop the new user product feature.


## Get & Select Data

First I downloaded the data and partitioned it into training, cross-validation, and testing sets. I added a column to the data (_Partition_) to track the assignment to these seporate sets. I combined all of the observations so that I can preprocess them all in the same way, while still keeping track of which partition they are assigned to. I take care to only visualize and run statistical analysis on the training set. 

I remove the columns that are not useful, including any with missing values or near zero variance. I remove columns that only hold aggregate rather than raw data. I also add dummy-coded columns for the user names, then I partition into separate training, cross-validation, and testing data sets.


```r
# Automatically download the data csv file if not already in working directory
train_url <- 'https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv'
testing_url <- 'https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv'
train_filename <- 'human_activity_training.csv'
testing_filename <- 'human_activity_testing.csv'
if (!file.exists(train_filename)){
    download.file(train_url,train_filename)}
if (!file.exists(testing_filename)){
    download.file(testing_url,testing_filename)}

# Read in Data from file
data_raw <- read.csv(train_filename, na.strings=c('','NA','#DIV/0!'))
test_raw <- read.csv(testing_filename, na.strings=c('','NA','#DIV/0!'))

# Partition Data into Train and Cross-Validation Sets
trainp <- 0.6
set.seed(420)
InTrain <- createDataPartition(y=data_raw$classe, p = trainp, list=FALSE)
data_raw[InTrain,'Partition'] <- 'train'
data_raw[-InTrain,'Partition'] <- 'cross'

# Read in Testing Set
test_raw <- test_raw %>% select(-problem_id) %>% mutate(
        Partition = 'test',
        classe = 'A')

# Combine Train, Cross, and Test to preprocess in the same way
combined <- rbind(data_raw,test_raw) %>% as.tibble()
combined$X <- as.character(combined$X)

# not useful columns
combined <- combined %>% select(-raw_timestamp_part_1, -raw_timestamp_part_2,
                                -cvtd_timestamp, -new_window, -num_window)
# remove variables with any NA
hasNAs <- apply(combined,2, anyNA)
combined <- combined[!hasNAs]

# remove variables with near zero variance
nearzerovariance <- nearZeroVar(combined, saveMetrics = TRUE)
combined <- combined[, !nearzerovariance$nzv]

# remove aggregating columns
combined <- combined %>% select(-starts_with('total'))

# add dummy variables for users
dummyusers <- predict(dummyVars('~ user_name', combined), newdata=combined ) 
combined <- cbind(dummyusers, combined) %>% arrange(desc(classe)) %>% as.tibble()

# repartition
my_train <- combined %>% filter(Partition=='train')
my_cross <- combined %>% filter(Partition=='cross')
my_test <- combined %>% filter(Partition=='test')
```



## Exploratory Analysis

Before training, I did an exploratory analysis of the data using Pair Plots and Principal Components. You can find that analysis [here](http://rpubs.com/lisamariepritchett/345942).

## Random Forest Training

In this report I train the random forest algorithm on data preprocessed in several ways:

- Raw: raw uprocessed data 
- Principal Components: The Principal Components Explaining 95% of the variance in the raw data
- User Self-trained: Train and Test on each user seporately
- New User: Train on other users and test on new user
- New User, Centered: Data is centered for each user and then tested on new user

I create custom functions to carry out the fitting and cross-validation so that I can easily pass different datasets later to examine the performance. 

### My Training Function

This function is used throughout this analysis to initiate training of the random forest algorithm and test the results. I used the caret package to train a random forest algorithm and cross-validate. I set the random forest to use 5-fold cross validataion using the train control feature in caret.  I used the parallel and doparallel packages to take advantage of multiple core processors and speed up training time. 
The function returns the training results and a summary dataframe with in- and out-of-sample accuracy rates and training time in seconds.


```r
# create my training function using the caret package to train and test a random forest algorithm using 5-fold cross validation. Uses the parallel and doparallel packages to implement parallel processing to speed training
run_training <- function(train,cross,label='mydata') {
        # choose training options
        traincontrol_cv5 <- trainControl(method = 'cv', #  'cv' cross validation
                                         number = 5,  #number of resampling itteration
                                         allowParallel = TRUE)
        # setup parallel processing
        cluster <- makeCluster(detectCores() -1) # leave 1 for OS
        registerDoParallel(cluster)
        
        # train using caret package
        starttime = strftime(Sys.time(),'%s') %>% as.numeric()
        result <- train(classe ~ . , data = train,
                        method = 'rf',
                        trControl = traincontrol_cv5)
        seconds.train = strftime(Sys.time(),'%s') %>% as.numeric() - starttime
        
        # release cluster
        stopCluster(cluster)
        registerDoSEQ()
        
        # check results on cross-validation set
        cross$rawpredict <- predict(result,cross)
        cross$rawcorrect <- cross$rawpredict == cross$classe
        accuracy.cross = mean(cross$rawcorrect)
        accuracy.train = max(result$results$Accuracy)

        # create result summary as a row to a dataframe
        train_summary <- data.frame(label, seconds.train, accuracy.train, 
                                    accuracy.cross)
        # return both the caret result and the summary as list 
        myreturn <- list(result, train_summary)
        myreturn
}
```







### Part 1: Predict Activity Class from Accelerometer Data

Train the random forest using the raw data and the Principal Components:


```r
#### Train on Raw Data
set.seed(420)
myreturn <- run_training(train = select(my_train, -Partition, -user_name, -X), 
              cross = my_cross, 
              label = 'Raw')

rf_raw <- myreturn[[1]]
train_summary <- myreturn[[2]]
my_summary <- train_summary
rf_raw
```

```
## Random Forest 
## 
## 11776 samples
##    54 predictor
##     5 classes: 'A', 'B', 'C', 'D', 'E' 
## 
## No pre-processing
## Resampling: Cross-Validated (5 fold) 
## Summary of sample sizes: 9422, 9420, 9420, 9420, 9422 
## Resampling results across tuning parameters:
## 
##   mtry  Accuracy   Kappa    
##    2    0.9872628  0.9838853
##   28    0.9884515  0.9853909
##   54    0.9811495  0.9761484
## 
## Accuracy was used to select the optimal model using the largest value.
## The final value used for the model was mtry = 28.
```



```r
#### Train on Principal Componets

## Use 99%
# create preprocessing object for PCA
preProc <- preProcess(select(my_train, roll_belt:classe), 
                      method=c("pca"), 
                      thresh = 0.99, # portion of variance to retain
                      pcaComp = NULL) # number of components)

# Apply PCA trained on only training data to all the data to get PC scores 
pca_train <- predict(preProc, select(my_train, -X, -user_name, -Partition))
pca_cross <- predict(preProc, my_cross)
myreturn <- run_training (train = pca_train, 
                          cross = pca_cross, 
                          label = 'PC 99%')
rf_PC99 <- myreturn[[1]]
train_summary <- myreturn[[2]]
my_summary <- rbind(my_summary,train_summary)
rf_PC99
```

```
## Random Forest 
## 
## 11776 samples
##    40 predictor
##     5 classes: 'A', 'B', 'C', 'D', 'E' 
## 
## No pre-processing
## Resampling: Cross-Validated (5 fold) 
## Summary of sample sizes: 9420, 9420, 9420, 9423, 9421 
## Resampling results across tuning parameters:
## 
##   mtry  Accuracy   Kappa    
##    2    0.9693453  0.9612016
##   21    0.9645896  0.9551874
##   40    0.9565224  0.9449744
## 
## Accuracy was used to select the optimal model using the largest value.
## The final value used for the model was mtry = 2.
```

```r
## Use 95%
# create preprocessing object for PCA
preProc <- preProcess(select(my_train, roll_belt:classe), 
                      method=c("pca"), 
                      thresh = 0.95, # portion of variance to retain
                      pcaComp = NULL) # number of components)

# Apply PCA trained on only training data to all the data to get PC scores 
pca_train <- predict(preProc, select(my_train, -X, -user_name, -Partition))
pca_cross <- predict(preProc, my_cross)
myreturn <- run_training (train = pca_train, 
                          cross = pca_cross, 
                          label = 'PC 95%')
rf_PC95 <- myreturn[[1]]
train_summary <- myreturn[[2]]
my_summary <- rbind(my_summary,train_summary)
rf_PC95
```

```
## Random Forest 
## 
## 11776 samples
##    29 predictor
##     5 classes: 'A', 'B', 'C', 'D', 'E' 
## 
## No pre-processing
## Resampling: Cross-Validated (5 fold) 
## Summary of sample sizes: 9420, 9422, 9420, 9422, 9420 
## Resampling results across tuning parameters:
## 
##   mtry  Accuracy   Kappa    
##    2    0.9624660  0.9525140
##   15    0.9609379  0.9505894
##   29    0.9540591  0.9418886
## 
## Accuracy was used to select the optimal model using the largest value.
## The final value used for the model was mtry = 2.
```

```r
my_summary
```

```
##    label seconds.train accuracy.train accuracy.cross
## 1    Raw           283      0.9884515      0.9920979
## 2 PC 99%           212      0.9693453      0.9738720
## 3 PC 95%           152      0.9624660      0.9663523
```


#### Performance for Each Observer

Train the algorithm on each user's data separately:


```r
set.seed(2)
userlist <- c('adelmo','carlitos','charles','eurico','jeremy','pedro')
user_summary <- data.frame()

# Train for each user
for (user in userlist) {
        # select data
        userdata <- my_train %>% filter(user_name==user) %>% select(roll_belt:classe)
                
        # remove variables with near zero variance
        nearzerovariance <- nearZeroVar(userdata, saveMetrics = TRUE)
        userdata <- userdata[, !nearzerovariance$nzv]
        
        # train on user's raw data
        myreturn <- run_training (train = userdata, 
              cross = filter(my_cross, user_name==user), 
              label = 'Mean Self Trained')
        
        rf_user <- myreturn[[1]]
        train_summary <- myreturn[[2]]
        train_summary <- cbind(user,train_summary)
        user_summary <- rbind(user_summary,train_summary)
}
user_avg_summary <- cbind(data.frame(label = 'Self Trained'), summarise_if(user_summary,'is.numeric','mean'))
my_summary <- rbind(my_summary,user_avg_summary)
user_summary
```

```
##       user             label seconds.train accuracy.train accuracy.cross
## 1   adelmo Mean Self Trained            33      0.9857944      0.9891651
## 2 carlitos Mean Self Trained            25      0.9924961      0.9903537
## 3  charles Mean Self Trained            25      0.9915647      0.9978617
## 4   eurico Mean Self Trained            24      0.9935469      0.9934048
## 5   jeremy Mean Self Trained            26      0.9890485      0.9942529
## 6    pedro Mean Self Trained            21      0.9861158      0.9834146
```

```r
my_summary
```

```
##          label seconds.train accuracy.train accuracy.cross
## 1          Raw     283.00000      0.9884515      0.9920979
## 2       PC 99%     212.00000      0.9693453      0.9738720
## 3       PC 95%     152.00000      0.9624660      0.9663523
## 4 Self Trained      25.66667      0.9897611      0.9914088
```





```r
my_summary %>% ggplot(aes(x=label,y=seconds.train)) + 
        geom_bar(stat='identity', fill = 'navy', alpha=0.7) +
        labs(x='Traing Data',y='Seconds',
             title='Training Time in Seconds by Training Type ') +
        theme_gray(base_size = 15)
my_summary %>% ggplot(aes(x=label,y=accuracy.cross)) + 
        geom_bar(stat='identity', fill = 'darkgreen', alpha= 0.7)+
        labs(x='Class Prediction Traing Data',y='Accuracy',
             title='Prediction Accuracy by Training Type')+
        theme_gray(base_size = 15)
```

<img src="activityClassification_files/figure-html/summaryplot-1.png" width="50%" /><img src="activityClassification_files/figure-html/summaryplot-2.png" width="50%" />



The algorithm achieved better than 99% accurcay when trained on tested on the raw data. When Principal Componets were used accuracy was reduced to about 97%, but training time was cut in half. When trained and tested by each user individually accuracy was again over 99%, and each training was between 21 and 34 seconds. This deomonstrates the feasbility of the product as long as each new user can collect a training set.


### Part 2: New Users

Train the random forest algorithm using data from all but one user who becomes the testing set.


```r
#train on 5 users and test on 1
for (user in userlist) {
        #Select Data 
        newuser_train <- combined %>% filter(user_name != user) %>% 
                select(roll_belt:classe)
        newuser_cross <- combined %>% filter(user_name == user)
        
        # Train on User's data
        set.seed(420)
        myreturn <- run_training(train = newuser_train, 
                                   cross = newuser_cross, 
                                   label =  'New User')
        rf_newuser <- myreturn[[1]]
        train_summary <- myreturn[[2]]
        train_summary <- cbind(user,train_summary)
        user_summary <- rbind(user_summary,train_summary)
}
user_summary
```

```
##        user             label seconds.train accuracy.train accuracy.cross
## 1    adelmo Mean Self Trained            33      0.9857944      0.9891651
## 2  carlitos Mean Self Trained            25      0.9924961      0.9903537
## 3   charles Mean Self Trained            25      0.9915647      0.9978617
## 4    eurico Mean Self Trained            24      0.9935469      0.9934048
## 5    jeremy Mean Self Trained            26      0.9890485      0.9942529
## 6     pedro Mean Self Trained            21      0.9861158      0.9834146
## 7    adelmo          New User           337      0.9943489      0.1810943
## 8  carlitos          New User           385      0.9950991      0.4770465
## 9   charles          New User           376      0.9937907      0.5535765
## 10   eurico          New User           364      0.9940848      0.1909564
## 11   jeremy          New User           408      0.9949484      0.5472141
## 12    pedro          New User           407      0.9941277      0.2916188
```

When I test on a user who provided no training data performance is very poor. It is at or below chance (20%) for 3 of the 6 new users, and no better than 56% accuracy for any of the users.

#### New Users, Centered Data

In my [exploratory analysis](url) I noticed that there was a lot of between-user varaibility. Now I examine whether removing that variability by centering each user's data could improve the performance.

Below, I center the data for each user. This is a preprocessing step and needs to be done for the test, cross-validation, and train data before being sent to the machine learning algorithm, so I combine those sets into one and center by user before repartitioning,

I found that after removing the between-subjects variance, a single outlying point accounted for a very large portion of the variance and the second Principal Component. I suspect the observation may have been an error or anomaly, and so I removed that point (X = 5373). Finally I plot the centered and uncentered data to visualize the effect of user-centering. 


```r
## center the data by user
combined_centered <- combined %>% select(X:Partition)%>% group_by(user_name) %>% 
        mutate_if(is.numeric, function(x) {x - mean(x)}) %>% ungroup()# %>% 

## add dummy variables back
combined_centered <- cbind(dummyusers,combined_centered) %>% as.tibble()
        
#update to removeoutlier here
# remove  one outlier
combined_centered <- combined_centered %>% filter(X != 5373)

# partition
my_train_centered <- combined_centered %>% filter(Partition=='train')
my_cross_centered <- combined_centered %>% filter(Partition=='cross')
my_test_centered <- combined_centered %>% filter(Partition=='test')

my_train %>% ggplot(aes(x = accel_dumbbell_x, y = accel_dumbbell_y, color = user_name)) + 
        geom_point(alpha = .3) +
        labs(title='Raw Dumbbell Acceleration', color='User') +
        theme_gray(base_size = 15)

my_train_centered %>% ggplot(aes(x = accel_dumbbell_x, y = accel_dumbbell_y, color = user_name)) + 
        geom_point(alpha = .3) +
        labs(title='User-Centered Dumbbell Acceleration', color='User')+
        theme_gray(base_size = 15)
```

<img src="activityClassification_files/figure-html/user_center-1.png" width="50%" /><img src="activityClassification_files/figure-html/user_center-2.png" width="50%" />

The plots above show the effect of user centering. In the raw data, each user's data was centered around a different point. After centering data for all of the users is centered around zero. This centering was done for every variable. The point removed from the training is visable in the plot on the left, it is the single point below -400 on accel_dumbbedll_x, the point was an outlier on many other dimensions as well.

Train with centered data:



```r
#train on 5 users and test on 1, with centered
for (user in userlist) {
        newuser_train_centered <- my_train_centered %>% 
                filter(user_name != user) %>% 
                select(roll_belt:classe)
        
        newuser_cross_centered <- my_cross_centered %>% 
                filter(user_name == user)
        
        # Train on all Other Users, test on newuser
        set.seed(319)
        myreturn <- run_training(train = newuser_train_centered, 
                                            cross = newuser_cross_centered, 
                                            label = 'New User, Centered')
        rf_newuser_centered <- myreturn[[1]]
        train_summary <- myreturn[[2]]        
        train_summary <- cbind(user,train_summary)
        user_summary <- rbind(user_summary,train_summary)
}
user_summary
```

```
##        user              label seconds.train accuracy.train accuracy.cross
## 1    adelmo  Mean Self Trained            33      0.9857944      0.9891651
## 2  carlitos  Mean Self Trained            25      0.9924961      0.9903537
## 3   charles  Mean Self Trained            25      0.9915647      0.9978617
## 4    eurico  Mean Self Trained            24      0.9935469      0.9934048
## 5    jeremy  Mean Self Trained            26      0.9890485      0.9942529
## 6     pedro  Mean Self Trained            21      0.9861158      0.9834146
## 7    adelmo           New User           337      0.9943489      0.1810943
## 8  carlitos           New User           385      0.9950991      0.4770465
## 9   charles           New User           376      0.9937907      0.5535765
## 10   eurico           New User           364      0.9940848      0.1909564
## 11   jeremy           New User           408      0.9949484      0.5472141
## 12    pedro           New User           407      0.9941277      0.2916188
## 13   adelmo New User, Centered           186      0.9886798      0.4442320
## 14 carlitos New User, Centered           198      0.9840529      0.6864952
## 15  charles New User, Centered           207      0.9863095      0.4112616
## 16   eurico New User, Centered           199      0.9849788      0.2926628
## 17   jeremy New User, Centered           196      0.9891454      0.7169540
## 18    pedro New User, Centered           210      0.9848875      0.5404878
```



#### Results Summary


```r
user_summary
```

```
##        user              label seconds.train accuracy.train accuracy.cross
## 1    adelmo  Mean Self Trained            33      0.9857944      0.9891651
## 2  carlitos  Mean Self Trained            25      0.9924961      0.9903537
## 3   charles  Mean Self Trained            25      0.9915647      0.9978617
## 4    eurico  Mean Self Trained            24      0.9935469      0.9934048
## 5    jeremy  Mean Self Trained            26      0.9890485      0.9942529
## 6     pedro  Mean Self Trained            21      0.9861158      0.9834146
## 7    adelmo           New User           337      0.9943489      0.1810943
## 8  carlitos           New User           385      0.9950991      0.4770465
## 9   charles           New User           376      0.9937907      0.5535765
## 10   eurico           New User           364      0.9940848      0.1909564
## 11   jeremy           New User           408      0.9949484      0.5472141
## 12    pedro           New User           407      0.9941277      0.2916188
## 13   adelmo New User, Centered           186      0.9886798      0.4442320
## 14 carlitos New User, Centered           198      0.9840529      0.6864952
## 15  charles New User, Centered           207      0.9863095      0.4112616
## 16   eurico New User, Centered           199      0.9849788      0.2926628
## 17   jeremy New User, Centered           196      0.9891454      0.7169540
## 18    pedro New User, Centered           210      0.9848875      0.5404878
```

```r
user_summary %>% group_by(label) %>% summarise_if('is.numeric','mean')
```

```
## # A tibble: 3 x 4
##                label seconds.train accuracy.train accuracy.cross
##               <fctr>         <dbl>          <dbl>          <dbl>
## 1  Mean Self Trained      25.66667      0.9897611      0.9914088
## 2           New User     379.50000      0.9943999      0.3735844
## 3 New User, Centered     199.33333      0.9863423      0.5153489
```

```r
user_summary %>% ggplot(aes(y=accuracy.cross, x = user, fill = label)) + 
        geom_bar(stat='identity', position = "dodge") + 
        geom_hline(yintercept = 1/6, alpha=.7, size=1) +
        labs(title='Accuracy Predicting a Single User, Self-Trained or New', 
             y='Accuracy', x='User',fill='') + 
        theme_gray(base_size = 12)
```

<img src="activityClassification_files/figure-html/userplots2-1.png" width="100%" height="50%" />



The plots above show the accuracy of the random forest algorithm for predicting each user's activity class depending on whether it was tested and trained on the same (self-trained) or different (new) users. It also shows the improvement in accuracy from user-centering.

As we saw before when trained and tested on the same user (self-train) the algorithm does very well (mean 99% accurate), but when tested on a new user performance is poor (mean 37% accurate). It is somewhat better when user-centering is applied for all users, but is still low (mean 53% accurate).


The algorithm is probably overfitting to the small number of users. A larger sample of training-users should be gathered to further develop this product. More measurements could also be made, especially the users heights, weights and arm lengths.


## Reference:

Velloso, E., Bulling, A., Gellersen, H., Ugulino, W. & Fuks, H. (2013). Qualativie Activity Recognition of Weight Lifting Exercises. _Proceedings of the 4th Augemented Human (AH) International Conference._ Stugttgart, Germany.




-----

This report is the work of Lisa Marie Pritchett and was knitted on 2018-01-02 for Practical Machine Learning, Data Science Specialization, Johns Hopkins University, Coursera.

```
