---
title: "Exploring the Activity Classification Dataset"
author: "Lisa Marie Pritchett"
date: "January 2, 2018"
output:
  html_document:
    keep_md: yes
---




## Introduction

This is the exploratory analysis for my activity classification machine learning project. I show pair plots to visualize the raw data and biplots to visualize the Principal Components. I train a random forest algorithm using this data in a separate report [here](http://rpubs.com/lisamariepritchett/).

Wearable sensors are popular for tracking amount of activity throughout the day, but can these sensors also be used to help users improve activity techniques? 

To show that this idea is feasable, I examine data from a study where participants performed bicep curls while accelerometers were attached to their body and equipment. They each completed one set of 10 standard Unilateral Dumbbell Biceps Curls in 5 different fashions:

__Dumbbell Lifting Classes:__

- Class A: Exactly according to the specification 
- Class B: Throwing the elbows to the front
- Class C: Lifting the dumbbell only halfway 
- Class D: Lowering the dumbbell only halfway 
- Class E: Throwing the hips to the front

The accelerometers were mounted on the user's belt, arm, forearm, and dumbell. The data are the readings from the accelerometers, and are described by the authors of the original study as follows:

_"For data recording we used four 9 degrees of freedom Razor inertial measurement units (IMU), which provide three-axes acceleration, gyroscope and magnetometer data at a joint sampling rate of 45 Hz"_(Vellosso, 2013).

## Download, Read and Select Data


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

# remove aggregating columns
combined <- combined %>% select(-starts_with('total'))

# remove outlier, try running without this line to see effect of the outlier
combined <- combined %>% filter(X != 5373)

# repartition
my_train <- combined %>% filter(Partition=='train')
my_cross <- combined %>% filter(Partition=='cross')
my_test <- combined %>% filter(Partition=='test')
```





Now the data contains only the data from the sensors mounted on the weight-lifter's belt, arm, forearm, and dumbell. The data indicates the spatial orientation, location, and acceleration of the sensor using 12 measurements for each of 4 sensors at each observation. 



## Principal Components Analysis

Principal Component Analysis can be used for visualizing the data and in machine learning to reduce dimensionality and speed training time. 

Here I run principal components analysis and examine the scree plots showing the amount of explained variance for each component. 


```r
pca <- my_train %>% select(roll_belt:magnet_forearm_z) %>% 
                prcomp(center=TRUE, scale=TRUE)

pca.loadings <- as.data.frame(pca$rotation) %>% 
                rownames_to_column() %>% 
                as.tibble() %>% arrange(desc(abs(PC1)))

pca.scores <-  cbind(user = my_train$user_name, 
                     classe = my_train$classe,
                     as.data.frame(pca$x)) %>%
                as.tibble()

# Scree Plots
ggscreeplot(pca, type = "pev") + theme_gray(base_size = 15)
ggscreeplot(pca, type = "cev") + geom_hline(yintercept = 0.95) + geom_hline(yintercept = 0.99)+ theme_gray(base_size = 15)
```

<img src="activityExploration_files/figure-html/unnamed-chunk-1-1.png" width="50%" /><img src="activityExploration_files/figure-html/unnamed-chunk-1-2.png" width="50%" />

The plot on the left above shows the proportion of variance explained by each component, we see that the first two components explain more variance than the rest. To capture 95% of the variance in the data, 22 components are needed, and 99% of variance is captured by 35 components.

Next, I examine a biplot which shows the scores of each observation on the first two principal components, along with the loading of each of the variables on those components. I could use the ggbiplot to quickly make this, but found that I needed more control over the appearance of the figure to make it intelligible, so I made the plot by hand.


```r
pca.loadings2 <- pca.loadings %>% arrange(desc(abs(PC2)))
scalec <- max(pca.scores$PC1) / max(pca.loadings$PC1)
nvectors <- 15


ggplot() + geom_point(data = pca.scores, aes(x=PC1,y=PC2,color=classe), 
                       alpha = .2)  +
        theme_gray(base_size = 20)+ guides(colour = guide_legend(override.aes = list(alpha = 0.7)))

#ggbiplot(plot.data = pca, groups = my_train$user)
ggplot() + geom_point(data = pca.scores, aes(x=PC1,y=PC2,color=user), 
                       alpha = .2) + 
                geom_segment(data = pca.loadings[1:nvectors,], 
                             aes(x=0, y=0, xend=PC1*scalec, yend=PC2*scalec),
                             color = 'navy', alpha = .5) +
                geom_segment(data = pca.loadings2[1:nvectors,], 
                             aes(x=0, y =0, xend = PC1*scalec, yend = PC2*scalec),
                             color = 'red', alpha=.5) +
                geom_label(data = pca.loadings[1:nvectors,], 
                           aes(x=PC1*scalec, y=PC2 * scalec,label = rowname),
                           alpha = 0.1, nudge_y = 0) +
                geom_label(data = pca.loadings2[1:nvectors,], 
                           aes(x=PC1*scalec, y=PC2 * scalec, label = rowname),
                           alpha=0.1, nudge_x = 0) +
        ylim(c(-10,10)) + xlim(c(-10,10)) +
        theme_gray(base_size = 20)+ guides(colour = guide_legend(override.aes = list(alpha = 0.7)))
```

<img src="activityExploration_files/figure-html/biplot-1.png" width="50%" /><img src="activityExploration_files/figure-html/biplot-2.png" width="50%" />

### Principal Components Biplot
On the left I  plotted the scores from the first two principal components with color representing different classes. Although there is not nearly as much variation between classes as there is between participants (colors on the right), we do see some separation.

On the right is a biplot, which shows both component scores and variable loadings on the same axis. On this plot points of different colors are used to represent different users scores, and vectors represent the factor loadings.

####  Component Scores as Points

The first two principal components reveal several clusters of points, these do not correspond to the different activity classes as we may have hoped, but rather to different participants in the data set. If more data on participants such as heights and arm lengths were available those variables might account for much of the variation in these first two principal components. 


####  Factor Loadings as Vectors

The length of each vector indicates the magnitude of the loadings on the components, with horizontal and vertical components representing the first and second components respectively. If a variable has a high loading on PC1 and a zero loading on PC2 it would lay along the x-axis. The angles between vectors indicate the relationships between variables. Vectors that are close together represent variables that are highly positively correlated. Vectors that are orthogonal indicate variables that are orthogonal. For example, accel_belt_z and accel_belt_y appear orthogonal in the biplot, and if we examine those variables we find they have a correlation of only -0.0435795. Also, accel_belt_x and pitch_belt appear opposite, and they have a correlation of -0.9659058.

We can see that the first two components are related to many of the variables in the data set. 
We can say that the first two components are accounting much of the between-subjects variance in the data set. 

A benefit of using random forests is that it will automatically account for the differences between participants, and will even account for interactions between participant and other variables. As long as a training set can be gathered for each new participant, the same algorithm can be applied to automatically learn the differences between activity types for that user and provide feedback on demand.



### Principal Components of the Centered Data

One way to account for differences between users is to center the data for each user separately. I do that below and then perform the PCA again.


```r
# run pca on partition
my_train_centered <- my_train %>% select(X:Partition)%>% group_by(user_name) %>% 
        mutate_if(is.numeric, function(x) {x - mean(x)}) %>% ungroup() %>% 
        mutate_if(is.numeric, function(x) {x / sd(x)})

pca_centered <- my_train_centered %>% select(roll_belt:magnet_forearm_z)  %>% 
                prcomp()

pca_centered.loadings <- as.data.frame(pca_centered$rotation) %>% 
                rownames_to_column() %>% 
                as.tibble() %>% arrange((abs(PC1)))

pca_centered.scores <-  cbind(X = my_train_centered$X,
                user = my_train_centered$user_name, 
                     classe = my_train_centered$classe,
                     as.data.frame(pca_centered$x)) %>%
        arrange(desc(classe)) %>% 
                as.tibble()

#pca_centered.loadings %>% arrange(desc(abs(PC1)))
#pca_centered.loadings %>% arrange(desc(abs(PC2)))

# Scree Plots
ggscreeplot(pca_centered, type = "pev") + theme_gray(base_size = 15)
ggscreeplot(pca_centered, type = "cev") + geom_hline(yintercept = 0.95)+ geom_hline(yintercept = 0.99)+ theme_gray(base_size = 15)
```

<img src="activityExploration_files/figure-html/pca_centered-1.png" width="50%" /><img src="activityExploration_files/figure-html/pca_centered-2.png" width="50%" />



Above are scree plots for the PCA on the centered data.


```r
pca_centered.loadings2 <- pca_centered.loadings %>% arrange((abs(PC2)))
scalec <- max(pca_centered.scores$PC1) / max(pca_centered.loadings$PC1)
nvectors = 48
ggplot() + geom_point(data = pca_centered.scores, alpha = .2, 
                      aes(x=PC1,y=PC2,color=classe)) + 
        guides(colour = guide_legend(override.aes = list(alpha = 0.7))) + facet_grid(. ~ classe)
```

<img src="activityExploration_files/figure-html/centeredPCAscores-1.png" width="100%" />




The plot above shows the scores on PC1 and PC 2 for each class seporately. Below the same points overlayed in a biplot:



```r
labellist <- c('magnet_arm_x','magnet_arm_y','magnet_arm_z','accel_arm_x',
               'magnet_dumbbell_x','magnet_dumbbell_y','magnet_dumbbell_z',
               'magnet_belt_y','roll_belt','yaw_belt','pitch_belt')
dataforlabels <- pca_centered.loadings %>% filter(rowname %in% labellist)
ggplot() + geom_point(data = pca_centered.scores, 
                      aes(x=PC1,y=PC2,color=classe), 
                       alpha = .2) + 
                geom_segment(data = pca_centered.loadings[1:nvectors,], 
                             aes(x=0, y=0, xend=PC1*scalec, yend=PC2*scalec),
                             color = 'darkred', alpha =.2,size=0.75) +
                geom_label(data = dataforlabels,
                           aes(x=PC1*scalec, y=PC2 * scalec,label = rowname),
                           alpha = 0.5, nudge_y = 0) +
        xlim(c(-11,11)) +
        ylim(c(-11,11)) + guides(colour = guide_legend(override.aes = list(alpha = 0.7)))
```

<img src="activityExploration_files/figure-html/biplot_centered-1.png" width="100%" />


Above is the biplot showing scores and loadings on PC1 and PC2 for the data centerd by user. 

Removing the between user variance allows us to see the differences between the classes. We can see that if we used only the first 2 Principal Components we would have some idea whether the task was being done correctly (class = A in red) or not, but would have a lot of error. For readability, I have chosen to only show some of the variable labels on this plot but all of the vectors. 

### Notes
- There are many variables (vectors in the biplot) highly related to both PC1 and PC2. Those appear to be measurements about arm position. 
- We see that measearments about arm in y and z-coordinates are highly correlated, but are both orthogonal to measurements about arm in x-coordinates.
- Measurements related to belt position have near-zero loadings on PC1, but are related to PC2. 
- Measurements about dumbbell x, y, and z are  correlated.


## Visualizations of Raw Data

Finally, I make plots of the raw data for inspection. I loop through the different measurements to make a series of pair-plots.
For the spatial orientation data (yaw, pitch, roll, measured in degrees) I  also show plots in polar coordinates.




```r
# This creates visualizations for the entire data set!
locations <- c('_arm', '_forearm', '_dumbbell', '_belt')
sensors <- c('magnet', 'accel', 'gyros')
for (location in locations) {
        # for magnet, accel, and gyros data
        for (sensor in sensors) {
                pairplot <- my_train %>%
                        select(classe,
                               starts_with(paste(sensor, location, sep = ''))) %>%
                        arrange(desc(classe)) %>%
                        ggpairs(mapping = aes(color = classe, alpha = .1))
                print(pairplot)
        }
        
        # for spatial orientation data
        pairplot <-
                my_train %>% select(classe, ends_with(location)) %>%
                arrange(desc(classe)) %>%
                ggpairs(mapping = aes(color = classe, alpha = .1)) 
        print(pairplot)
        
        # get yaw, pitch and roll
        yaw <- my_train %>% select(starts_with('yaw')) %>%
                select(ends_with(location))
        pitch <- my_train %>% select(starts_with('pitch')) %>%
                select(ends_with(location))
        roll <- my_train %>% select(starts_with('roll')) %>%
                select(ends_with(location))
        
        polarplot1 <- my_train %>% ggplot(aes(x = roll,y = pitch, color = user_name)) +
                geom_point(alpha = .3) + coord_polar() +
                theme(legend.position = 'none')
        polarplot2 <- my_train %>% ggplot(aes(x = roll,y = yaw,color = user_name)) +
                geom_point(alpha = .3) + coord_polar() +
                theme(legend.position = 'none')
        polarplot3 <- my_train %>% ggplot(aes(x = pitch,y = yaw,color = user_name)) +
                geom_point(alpha = .3) + coord_polar() +
                theme(legend.position = 'none')
        multiplot(polarplot1, polarplot2, polarplot3, cols = 3)
}
```

<img src="activityExploration_files/figure-html/pairs-1.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-2.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-3.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-4.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-5.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-6.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-7.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-8.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-9.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-10.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-11.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-12.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-13.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-14.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-15.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-16.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-17.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-18.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-19.png" width="100%" /><img src="activityExploration_files/figure-html/pairs-20.png" width="100%" />






### Notes 

- There was a lot of between-subjects difference. I accounted for it by user-centering the data, more precise measures could be taken, for example a rotation could also be applied to further reduce variability in user data.
- Variables were often correlated or at least not independent. 
- Distributions are often bi- or multi-modal or not normally distributed
- Classes do show separation on some of the dimensions. 
- There appears to be an outlier on the gyros_dumbbell and gyros_forearm measurements (rerun code without removing outlier in first chunk to visulize with the outlier). This outlier will probably not be a problem for a random forest algorithm, but it was removed for the Principal Components analysis.

A machine learning algorithm like random forests should do well at distinguishing the different classes. 

----
This analysis is the work of Lisa Marie Pritchett. This document contains all code needed to download the data and reproduce the analysis and figures. This report was knitted on 2018-01-02.
