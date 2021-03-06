---
title: "Comparing Modeling Techniques with Kaggle's Titanic Data"
output: github_document
---



```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
library(class)
library(tidyverse)
library(naivebayes)
library(pROC)
library(rpart)
library(rpart.plot)
library(randomForest)
library(skimr)
library(caret)
library(dataPreparation)
```

## Loading and Cleaning the data

The first step is loading the data and inspecting it to see what needs to be cleaned up and rangled. Kaggle provides a description of the data [here](https://www.kaggle.com/c/titanic/data)

```{r loading the data, message=FALSE}
titanic_train <- read_csv("data/train.csv")
titanic_submission <- read_csv("data/test.csv")
```

Inspecting the data we can see that generally it looks pretty good, but there are some areas of concern.  The first is that a lot of the data on cabins is missing. In fact we only have cabin data on about 23% of our records.We also notice that we have a significant amount of records where age is missing.  

It looks like there is a correlation between our dependent variable and whether or not we have cabin data on the person. We will want to make sure we include these records and handle them in a way that takes advantage of that correlation.

```{r inspecting the data, echo=FALSE}
skim(titanic_train)

titanic_train %>%
  group_by(Survived) %>%
  summarize(has_cabin = sum((!is.na(Cabin))), no_cabin = sum(is.na(Cabin))) %>%
  gather("cabin", "count", -Survived) %>%
  ggplot(aes(x = cabin, y = count, fill = factor(Survived))) +
  geom_bar(position="fill", stat="identity") +
  ylab("percent")+
  xlab(NULL) +
  scale_fill_discrete(name=NULL,
                         labels=c("didn't survive", "survived"))
```

Looking at Age, we can see that there also seems to be some correlation with wheter or not the passenger survived, albeit weaker than the cabin data. We will likely want to make sure not to exclude individuals with NA in their age field.

```{r check age, echo=FALSE}
titanic_train %>%
  group_by(Survived) %>%
  summarize(has_age = sum((!is.na(Age))), no_age = sum(is.na(Age))) %>%
  gather("age", "count", -Survived) %>%
  ggplot(aes(x = age, y = count, fill = factor(Survived))) +
  geom_bar(position="fill", stat="identity") +
  ylab("percent")+
  xlab(NULL) +
  scale_fill_discrete(name=NULL,
                         labels=c("didn't survive", "survived"))
```

Embarked is the only other field with missing values and it only has two. There should be no significant effect on the model if we keep them.

We will begin with the cabin data. We will convert all the NA's in this column to "unknown". Because the submission data set will be the data the model is used on, any changes made to the train data should also be made on the submission data.

```{r unknown cabin}
sum(is.na(titanic_train$Cabin))
titanic_train <- titanic_train %>%
  mutate(Cabin = replace_na(Cabin, "unknown"))
sum(is.na(titanic_train$Cabin))

titanic_submission <- titanic_submission %>%
  mutate(Cabin = replace_na(Cabin, "unknown"))
```

Now that the NAs in Cabin are taken care of, we want to take a closer look at the cabin data to see if there is any additional information we can glean from it. Not including unknown, there are 147 unique values for Cabin and they all seem to follow the pattern of having a letter followed by a number. 

```{r cabin pattern}
length(unique(titanic_train$Cabin))
head(titanic_train$Cabin, 25)
```

We learn from this [link](https://triangleinequality.wordpress.com/2013/09/08/basic-feature-engineering-with-the-titanic-data/) that the letter at the beginning indicates the deck. We will follow the work from this link and use this to create a new column indicating the deck of the passenger. First we want to check and make sure that the passengers assigned to multiple cabins are all on the same deck. we can confirm that they are. This [link](https://www.kaggle.com/c/titanic/discussion/22049) helps explain some of the more confusing values.

```{r deck}
titanic_train <- titanic_train %>%
  mutate(deck = str_sub(Cabin, end = 1))
titanic_submission <- titanic_submission %>%
  mutate(deck = str_sub(Cabin, end = 1))
```

We could assume it's obvious that any "u" stands for "unknown", but we want it to be more clear than that so we change them back to "unknown".

```{r deck unkown}
titanic_train <- titanic_train %>%
  mutate(deck = recode(deck, "u" = "unknown"))

titanic_submission <- titanic_submission %>%
  mutate(deck = recode(deck, "u" = "unknown"))
```

Now we move on to deal with the NA values in age. There doesn't seem to be a significant difference in age between genders, although the age for men seems to be slightly more skewed to the left than for women. We do have two variables that may help indicate age a little more though.  SibSp indicates how many siblings or spouses the passenger has on board, while Parch indicates how many parents or children the passenger has on board.

The Parch seems to give some pretty good insight into age at around 4 and above.  Th SibSp seems to be a little more telling as mores of the time in the Anglo world would make if very unlikely for anyone to be traveling with more than one spouse, but children would could easily be traveling with more than one sibling.

```{r age histogram, echo=FALSE, warning=FALSE}
ggplot(titanic_train, aes(x = Age)) +
  geom_histogram(aes(y = ..density..)) +
  facet_wrap(~SibSp) +
  geom_density() +
  labs(title = "Siblings or Spouses")
  
ggplot(titanic_train, aes(x = Age)) +
  geom_histogram(aes(y = ..density..)) +
  facet_wrap(~Parch) +
  geom_density() +
  labs(title = "Parents or Children")
```

With this knowledge we will impute age for missing values based on the median value by SibSp. Because we are adding values for age rather than marking them unknown, we will create a new column to keep track of which records had age and which were missing age.  

```{r impute missing age}
titanic_train <- titanic_train %>%
  group_by(SibSp) %>%
  mutate(age_original = !is.na(Age),
         Age = ifelse(is.na(Age), round(median(Age, na.rm = TRUE)), Age)) %>%
  ungroup()

titanic_submission <- titanic_submission %>%
  group_by(SibSp) %>%
  mutate(age_original = !is.na(Age),
         Age = ifelse(is.na(Age), 
                      ifelse(SibSp == 0, 29,
                             ifelse(SibSp == 1, 30,
                                    ifelse(SibSp == 2, 23,
                                           ifelse(SibSp == 3, 10,
                                                  ifelse(SibSp == 4, 6,
                                                         ifelse(SibSp == 5, 11,
                                                                Age)))))), Age))%>%
  ungroup()
```

unfortunately there are in the train data set we all of the records where SibSp is 8 have NA for age, so we can't calculate a median age for them.  They all seem to be from the same family.  This is kind of a real tossup as far as what to do.  It doesn't really make sense to assign them all to one age because we know they are from the same family, but at the same time, we can't assume that all future passengers in the submission set will be from the same family, so we need to have a consistent policy. The decision has been made to just impute those NA values with the same median age for SibSp == 5, which is 11.

```{r NA age 2}
titanic_train <- titanic_train %>%
  mutate(Age = ifelse(is.na(Age) & SibSp == 8, 11, Age))
  
titanic_submission <- titanic_submission %>%
  mutate(Age = ifelse(is.na(Age) & SibSp == 8, 11, Age))
```

Finally we will replace NA in embarked with "unknown" and since we're coming to the end of cleaning up our data we will also use the janitor package to clean the names.  We will also convert some of the character variables into factor variables.

```{r embarked NA}
titanic_train <- titanic_train %>%
  mutate(Embarked = replace_na(Embarked, "unknown"))

titanic_submission <- titanic_submission %>%
  mutate(Embarked = replace_na(Embarked, "unknown"))

titanic_train <- janitor::clean_names(titanic_train) %>%
  mutate(sex = factor(sex),
         cabin = factor(cabin),
         embarked = factor(embarked),
         deck = factor(deck),
         age_original = factor(age_original),
         survived = factor(survived))

titanic_submission <- janitor::clean_names(titanic_submission) %>%
  mutate(sex = factor(sex),
         cabin = factor(cabin),
         embarked = factor(embarked),
         deck = factor(deck),
         age_original = factor(age_original))
```

Now we have a value for every variable and every record, Even if that value is only there to indicate unknown.  Now let's take a look at the relationships between the Survived variable (our dependent variable) and all other numeric variables. We can see that the only numeric variable Survived seems to have any significant relationship with is the Pclass variable.  

```{r pairs graph, echo = FALSE}
titanic_train %>%
  select_if(is.numeric) %>%
  pairs(lower.panel = panel.smooth)
```

Looking at the relationship with some of the non-numeric variables we can see that there is certainly a strong relationship beween Sex and Survived and there also seem to be relationships with Embarked and deck as well.

```{r non-numeric variables graph, echo = FALSE}
p1 <- titanic_train %>%
  ggplot(aes(x = sex, group = survived, fill = survived)) +
  geom_bar(position = "fill") +
  theme(legend.position = "left")

p2 <- titanic_train %>%
  ggplot(aes(x = embarked, group = survived, fill = survived)) +
  geom_bar(position = "fill") +
  theme(legend.position = "none")

p3 <- titanic_train %>%
  ggplot(aes(x = deck, group = survived, fill = survived)) +
  geom_bar(position = "fill") +
  theme(legend.position = "none")

p4 <- titanic_train %>%
  ggplot(aes(x = age_original, group = survived, fill = survived)) +
  geom_bar(position = "fill") +
  theme(legend.position = "none")

gridExtra::grid.arrange(p1, p2, p3, p4, nrow = 2)
```

Now that we have a pretty good looking dataframe we can remove a few of the variables we won't be using in the models.  Name, Ticket, and PassengerID all seem to be forms of unique identifiers that probably won't be as helpful in these models.  In one of the links above, another model builder used Name to create a new varibale called title, but we will not be doing that.

```{r remove variables}
titanic_train <- titanic_train %>%
  select(-name, -ticket, -passenger_id, -cabin)

titanic_submission <- titanic_submission %>%
  select(-name, -ticket, -cabin)
```

## Building Models

The first thing we're going to want to do is split the train data set so that we have a test set to test our models on before making predictions on the data to be submitted. 

```{r split train set}
set.seed(3)

ttnc_smp_size <- floor(0.8 * nrow(titanic_train))

ttnc_train_ind <- sample(seq_len(nrow(titanic_train)), size = ttnc_smp_size)

ttnc_model_train <- titanic_train[ttnc_train_ind, ]
ttnc_model_test <- titanic_train[-ttnc_train_ind, ]

```

### Logistic Model

The first model we will build is a logistic model. This generalized linear model may be one of the simplest models for classification, but it will likely still be powerful and provide a good baseline for us to compare other models to.

We train the model using caret and calli summary on the model to see how it turned out.  Looking at this the model seems to show that sex, pclass, and age are all variables with very significant coefficients.  the coefficients for sib_sp and passengers who embarked out of Southampton both seem to be signficant at the 0.01 level.

```{r logisitic model, message=FALSE, warning=FALSE}
glm_pred <- train(survived ~ ., ttnc_model_train, method = "glm",
                  family = "binomial")

summary(glm_pred)
```

Now we check to see how this first model does on new data. We get the predicited values for the new data and see how often they are correct. Looks like this model predicts correctly about 82% of the time on the new data! However, this isnt the best measure. We could, for get above a 50% correct prediction rate if we just guessed everyone did not survive. Because of this we'll want to plot an ROC curve and calculate the area under the curve. A 1 would mean a perfect model predicitng correctly every time. We have an AUC of 0.8957!! Our baseline model seems to be very strong. We'll have to see if we can improve on this model at all.

```{r inspecting logistic model results}
ttnc_preds <- predict(glm_pred, ttnc_model_test)

mean(ttnc_model_test$survived == ttnc_preds)

ttnc_preds <- predict(glm_pred, ttnc_model_test, type = "prob")

log_ROC <- roc(ttnc_model_test$survived, ttnc_preds$`1`)

plot(log_ROC, col = "blue")

log_auc <- auc(log_ROC)
log_auc
```

### Naive Bayes Model

The next model we'll use to try and predicted survival on the titanic is Naive Bayes. 

```{r naive bayes model}

nb_predict <- train(survived ~ ., ttnc_model_train, method = "naive_bayes",
                    tuneGrid = data.frame(laplace = 0, usekernel = TRUE, 
                                          adjust = TRUE))
```

After training the model we check it's results on new data. Not only is this model less accurate than the logisitic model on the new data, but it also has a lower AUC.  Looks like the Naive Bayes model doesn't perform as well on new data as the Logistic model.

```{r inspecting Naive Bayes model results}
ttnc_preds <- predict(nb_predict, ttnc_model_test)

mean(ttnc_model_test$survived == ttnc_preds)

ttnc_preds <- predict(nb_predict, ttnc_model_test, type = "prob")

nb_ROC <- roc(ttnc_model_test$survived, ttnc_preds$`1`)

plot(nb_ROC, col = "blue")

nb_auc <- auc(nb_ROC)
nb_auc
```

### Random Forest Model

For our third model we will create a random forest model using the ranger package.  In order to include the probabilites using this model we will need to make some changes to the coding of the survived variable. 

```{r Random Forest model}
ttnc_model_train <- ttnc_model_train %>%
  mutate(survived = recode(survived, `0` = "didnt_survive",
                           `1` = "survived"))
rf_predict <- train(survived ~ ., ttnc_model_train, method = "ranger",
                    tuneLength = 15, 
                    trControl = trainControl(classProbs = TRUE))

```

Again, we check the model's performance on new data. This model looks really good! Outperforming even our logisitic regression score in both accuracy and auc. Where approaching 84% accuracy and an auc of 0.9279. That's 0.074 better than our logisitic model's auc.

```{r inspecting Random Forest model results}
ttnc_model_test <- ttnc_model_test %>%
  mutate(survived = recode(survived, `0` = "didnt_survive",
                           `1` = "survived"))

ttnc_preds <- predict(rf_predict, ttnc_model_test)

mean(ttnc_model_test$survived == ttnc_preds)

ttnc_preds <- predict(rf_predict, ttnc_model_test, type= "prob")

rf_ROC <- roc(ttnc_model_test$survived, ttnc_preds$survived)

plot(rf_ROC, col = "blue")

rf_auc <- auc(rf_ROC)
rf_auc
```

### Gradient Boost Model

Finally, we build a gradient boost model.

```{r gbm model, message=FALSE, include=FALSE}
ttnc_model_train <- ttnc_model_train %>%
  mutate(survived = recode(survived, `0` = "didnt_survive",
                           `1` = "survived"))
gbm_predict <- train(survived ~ ., ttnc_model_train, method = "gbm",
                    tuneLength = 15, 
                    trControl = trainControl(classProbs = TRUE))
```

Looking at the results of this final model on new data we see that it performs very well, but not quite as well as our random forest model. It's accuracy is 83% and auc is 0.9114. It looks like the model we will be using on our submission data will be the random forest model.

```{r results of gbm model}
ttnc_model_test <- ttnc_model_test %>%
  mutate(survived = recode(survived, `0` = "didnt_survive",
                           `1` = "survived"))

ttnc_preds <- predict(gbm_predict, ttnc_model_test)

mean(ttnc_model_test$survived == ttnc_preds)

ttnc_preds <- predict(gbm_predict, ttnc_model_test, type= "prob")

gbm_ROC <- roc(ttnc_model_test$survived, ttnc_preds$survived)

plot(gbm_ROC, col = "blue")

gbm_auc <- auc(gbm_ROC)
gbm_auc
```

### Building our final model and submission

So now that we know our random forest model performed the best on new data we will be using that model on our submission data. First, we need to train it on all of our training data.  Because we used a tunelength to have caret use multiple setting for the random forest model we need to inspect it to see what its final setting were. We see that the setting were mtry = 4, splitule = gini and min.node.size = 1. We set these in the tuneGrid in caret and retrain this model on the entire dataset we have.

```{r inspect random forest}
rf_predict

rf_predict <- train(survived ~ ., titanic_train, method = "ranger",
                    tuneGrid = data.frame(mtry = 4, splitrule = "gini",
                                          min.node.size = 1))
```

Now that we have the model trained we will use it to make predictions on the final submission data. We find that while there were no NAs for fare in the train dataset, the submission dataset has one value where fare is NA. For this we impute it with the median fare before running predict.

```{r predict on submission data}
titanic_submission <- titanic_submission %>%
  mutate(fare = replace_na(fare, median(fare, na.rm = TRUE)))

prediction <- predict(rf_predict, titanic_submission)

titanic_submission <- titanic_submission %>%
  bind_cols(prediction = prediction) %>%
  mutate(PassengerId = passenger_id, Survived = prediction) %>%
  select(PassengerId, Survived) 
  
```

After submitting these prediction my accuracy was 0.75598, which ranked me at 19,516 on the leaderboard for this contest on Kaggle.com. Thank you very much for view my work and I look forward to any feedback or advice anyone would like to provide.
