---
title: "Logs 2020-04-27: Learning Kaggle"
date: 2020-04-27T06:29:02-07:00
draft: false
---

This past week I tried my hand at the [Titanic Kaggle competition](https://www.kaggle.com/c/titanic/). The goal of this contest is to predict whether a given passenger on the _Titanic_ survived the disaster based on data collected about the attributes of passengers.

Even though I've spent a lot of time studying machine learning and statistics throughout my degree, I've only had a tiny amount of experience doing Kaggle competitions. In doing this one, I feel like I've managed to find a work process and compare different models to each other that's worked somewhat well, and in coming up with it I didn't need to use a tutorial or even any sort of recommendation as to what model to use. I just had the `scikit-learn` and `pandas` documentation and the data itself. As of now, I'm not incredibly high on the leaderboard yet, but coming up with this process was in and of itself quite rewarding for me as I've finally had the time to apply in practice a lot of what I've learned in theory.

(As of now, I'm not using a deep neural network framework like Tensorflow or Torch or Flux.jl or anything like that, just because the Titanic problem seems small enough that I wouldn't need to use them.)

## Starting out

The first thing I thought to do when starting a competition is to try to find out what the data means. Luckily enough, the competition's page has documented the semantics of each column, but we aren't always going to be so lucky. Even before looking at the competition page I went and loaded the `csv` file (fortunately as well, it wasn't too large) and looked at what the columns were.

We start here by using `pandas` to read the csv and just output the column names in an `ipython` interpreter:

```
In [1]: import pandas                                                            
pan
In [2]: pandas.read_csv("./train.csv")                                           
Out[2]: 
     PassengerId  Survived  Pclass  ...     Fare Cabin  Embarked
0              1         0       3  ...   7.2500   NaN         S
1              2         1       1  ...  71.2833   C85         C
2              3         1       3  ...   7.9250   NaN         S
3              4         1       1  ...  53.1000  C123         S
4              5         0       3  ...   8.0500   NaN         S
..           ...       ...     ...  ...      ...   ...       ...
886          887         0       2  ...  13.0000   NaN         S
887          888         1       1  ...  30.0000   B42         S
888          889         0       3  ...  23.4500   NaN         S
889          890         1       1  ...  30.0000  C148         C
890          891         0       3  ...   7.7500   NaN         Q

[891 rows x 12 columns]

In [3]: trainset = pandas.read_csv("./train.csv")                                

In [4]: trainset.columns                                                         
Out[4]: 
Index(['PassengerId', 'Survived', 'Pclass', 'Name', 'Sex', 'Age', 'SibSp',
       'Parch', 'Ticket', 'Fare', 'Cabin', 'Embarked'],
      dtype='object')
```

Okay, so everything seems to match up with the documentation of the competition. In the documentation we see that `Pclass` is a categorical value, while `Embarked` and `Sex` can be obviously pegged as categorical just by a cursory look at the dataset.

Models in `scikit-learn` deal primarily with numbers and don't work particularly well with strings (at least, not always, and not automatically), so it's necessary here to change these categorical values to a different encoding. Because there are few enough classes I decided to opt for dummy encoding, which is accessible in Pandas using the [`.get_dummies()`](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.get_dummies.html) method. Dummy encoding turns a single categorical column with \\( k \\) categories into \\( k \\) columns with values either 1 or 0 for whether or not the category is equal.

I also decided to drop a few columns here when I noticed that there's not enough data present in that column or when it looked like the variable couldn't really have a bearing on the chance of survival, like the passenger's name.

At this point, I wasn't done yet with preprocessing; I still needed to be able to pass the data into an sklearn model's `.fit()` call.

When I worked with Jupyter notebooks before I'd rapidly test stuff out and often find myself in a tangle of out-of-order code executions. When we're trying to preprocess our data but we're not sure at what preprocessing stage the data that currently exists in the `X` variable in the global state is at, the notebook's environment actually starts to become taxing to use.

My own pet solution to avoid that potential entanglement is to write a couple of functions where we'd place all of the logic to preprocess each piece of data, converting between the `DataFrame` and whatever form that would need to get passed to the models. In this case, as we're just using `scikit-learn` models for now, we're transforming them to `np.array` objects.


```python
def transform(dataset):
    d = pandas.get_dummies(dataset, columns=['Embarked', 'Pclass', 'SibSp', 'Sex'])
    d = d.drop(columns=['Cabin', 'Name', 'Ticket'])
    return d

def for_model_input(dataset, test=False):
    d = dataset.drop(columns=['PassengerId'])
    if test:
        y = None
        X = d.values
    else:
        y = np.ravel(d[['Survived']].values) # * 2 - 1
        X = d.drop(columns=['Survived']).values
    X = preprocessing.scale(X)
    return (X, y)
```

Doing things this way is really just following basic software engineering principles that seem a lot like common sense but always escape me if I start off developing on a Jupyter notebook.

* Any transforms we do to the training and test datasets are guaranteed to be exactly the same, and changing the process only requires me to change code in one place.
* Putting the preprocessing stages in just one or two functions means that we minimize the number of intermediate steps where things can get left halfway and confused.


## Missing data

I couldn't even train a single model at this point because some of the rows had missing values. Showing that we need to deal with this is part of the instructive value of this competition, I guess. I've learned from some classes in the past that there are different ways to impute missing data in a dataset &ndash; using the [EM Algorithm](http://www.stats.ox.ac.uk/~steffen/teaching/fsmHT07/fsm407c.pdf) for instance. `scikit-learn` has its own `sklearn.impute` model which contains a number of imputers, and it looks like the most "stable" of these is the [`KNNImputer`](https://scikit-learn.org/stable/modules/generated/sklearn.impute.KNNImputer.html). According to the documentation, it computes missing values by taking the nearest neighbors of that data row (along other covariates); it then takes the mean of these rows' values for the covariate of interest and sets the missing value to that mean.

If this process sounded at all complicated, I'm delighted to inform we that doing this is as simple as doing another `fit_transform()` call.

```python
knn_imputer = KNNImputer()
Xtrain = knn_imputer.fit_transform(Xtrain)
```

## Trying out models

At this point the first major hurdle was already over. Maybe one important lesson from these struggles is that we're always going to spend a lot of time cleaning and preprocessing data.

Now I was ready to start testing models. This is a classification problem and the most basic form of classifier model is a [logistic regression](https://en.wikipedia.org/wiki/Logistic_regression) classifier. Scikit-learn is magic, of course, and provides a logistic regression classifier like this out of the box. Our `X` and `y` values correspond to the training data and the `Survived` column respectively; here `Survived` is the variable we want to be able to predict.

```python
logit_model = LogisticRegressionCV(fit_intercept=True)
logit_model.fit(Xtrain, ytrain)
logit_scores = cross_val_score(logit_model, Xtrain, ytrain, cv=5)
```

I then make predictions using this model on the test set:

```python
predictions_logit = logit_model.predict(Xtest)
predictions_logit = predictions_svc.astype('int64')
```

It's amazing that one can do all of this in a five-liner. Where we need to be slightly more involved is writing to a `.csv` file in the proper format for Kaggle submission. Kaggle expects a format of `PassengerId`, `Survived` where `Survived` is the predicted outcome of survival from our model.

```python
pred_logit_df = pandas.DataFrame(predictions_logit, columns=['Survived'])
fin_ans_logit = pandas.DataFrame(testset['PassengerId']).join(pred_logit_df)
with open('predictions_logit.csv', 'w') as f:
    f.write((fin_ans_logit.to_csv(index=False)))
```

And we're good! I submitted this to Kaggle and got a score of 0.66985. This means that I got about two-thirds of my predictions right, which I don't think is half-bad! Now the only remaining thing to do is to test out more models and see if they get better submission scores, and we're golden.

![My first submission to the Titanic contest](/images/kaggle_submission_1.png)

## Hold up, you're not validating

If our benchmark for testing our models is using Kaggle's evaluation on the test set after we submit them, then we're committing quite a few errors. Doing this is almost like using the test set as part of training (here, as part of model selection), which is a big no-no that we'd learn in the first couple of lectures of a machine learning course. Consider this thought experiment: there are a finite number of possible submissions that someone can make to Kaggle's system. This number is astronomically huge, but even if it were feasible to submit every single possible combination until we get a perfect score on the contest, this still isn't the right idea. The point of the competition is to create a model that's able to predict outcomes based on data, which we're definitely not doing if we just submit every possible combination, and we're not doing well enough if our models are just optimized for doing well on the test set. Kaggle incentivizes this idea by limiting the number of times we can submit predictions in a given time period (I believe it's 5 times in 24 hours for the Titanic competition).

What we want instead is to be able to have some measure of effectiveness of a model based on only the training set. Training error doesn't work here because models are subject to overfitting; we need a measure that estimates the **out-of-sample error** of our model; that is, the error our model's predictions have when evaluating data outside of the training set. You may have heard of the term _validation error_ before, which is a kind of error calculated by a process wherein we'd split our training set \\( X \\) into \\( V \subset X \\) and \\( T = X \\setminus V \\), train the models on \\( T \\), and compare their errors when making predictions on \\( V \\); and whichever model minimizes the error is then trained on \\( V \\). Validation sets aren't perfect, though; if we use the same validation set on way too many models, it's entirely possible that we "get lucky" and do well in predicting the validation set even though our model might perform worse in general when predicting things out of sample. In the machine learning course that I took this was known as optimization bias. (Aside: I think optimization bias maps on pretty nicely to the concept of "Type I error" in statistics, which is the probability that we conclude statistical significance, rejecting the null hypothesis, when in fact the null hypothesis is true. But that's its own investigation.)

We can try to do something better than just a single validation set by doing something called _cross-validation_. In cross-validation, we divide the training data into some set number \\( k \\) subsets called _folds_, which consist of a \\( \frac{k+1}{k} \\) portion of the training data, with each fold being created by excluding a different \\( \frac{1}{k} \\) portion. We then train the model \\( k \\) times on each of the folds, and test their error on predicting the excluded portion of each fold. When we have a bigger model that relies of much more training data, training becomes expensive (sometimes incredibly so, in the case of larger neural network-based models in particular), so training each model \\( k \\) times (in this case and in general people set \\( k = 5 \\) ) is infeasible, but because our training set is small and our models are relatively simple, it's definitely possible to do cross-validation.

Gee, doing all this folding sounds like a lot of work, I wonder how we're going to implement it:

```python
from sklearn.model_selection import cross_val_score

# ...

logit_scores = cross_val_score(logit_model, Xtrain, ytrain, cv=5)

# ...
print("Logit CV scores:\n", logit_scores, np.mean(logit_scores))
```

That wasn't that hard at all! Now we just have to apply this to any model that we try out next. 

```python
(Xtrain, ytrain) = for_model_input(trainset)
ytrain_svc = ytrain * 2 - 1

knn_imputer = KNNImputer()
Xtrain = knn_imputer.fit_transform(Xtrain)

svc_model = SVC()
svc_scores = cross_val_score(svc_model, Xtrain, ytrain_svc, cv=5)
svc_model.fit(Xtrain, ytrain_svc)

logit_model = LogisticRegressionCV(fit_intercept=True)
logit_scores = cross_val_score(logit_model, Xtrain, ytrain, cv=5)
logit_model.fit(Xtrain, ytrain)

selector_logit = RFE(logit_model)
selector_logit_scores = cross_val_score(selector_logit, Xtrain, ytrain_svc, cv=5)
selector_logit.fit(Xtrain, ytrain)

boosted_model = GradientBoostingClassifier()
boosted_scores = cross_val_score(boosted_model, Xtrain, ytrain, cv=5)
boosted_model.fit(Xtrain, ytrain)
```

Here we've fit each of the following models to the data:

* Logistic regression model
* Support vector machine model (note: I transformed the output values for this since an SVM expects -1 and 1 instead of 0 and 1, which is why here the value for the outputs is a different variable) 
* Logistic regression model with [recrusive feature elimination](https://scikit-learn.org/stable/modules/generated/sklearn.feature_selection.RFE.html), a form of feature selection
* [Gradient boosting classifier](https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.GradientBoostingClassifier.html)

I added these one-by-one and saved the predictions on the test set for each model in different files. I'd only submit a new set of predictions if the cross-validation score of a new model I tried (including the same kinds of models after removing or transforming certain features) exceeded or was about the same as the maximum CV score I encountered so far. This wouldn't necessarily translate to better scores on the Kaggle submissions, but in this case it did:

![Progression of my Kaggle submissions to the Titanic contest](/images/kaggle_submission_2.png)

Currently, submitting the gradient-boosting classifier's predictions has yielded my best score to date at about 0.77; my next goal is to break 80%. It's just a few ranks higher than the example model that the competition description provides, but hey, it's something, and we have a clear path to improvement.

## Conclusion

(All of my code for this is [available](https://github.com/emsal1863/kaggle_code/blob/master/titanic/analyze.py) on Github)

This isn't really a post about how to win at Kaggle. There's probably a tutorial out there teaching you exactly what the right model is for the Titanic problem, and you can copy and paste that model into a Python file or run it on Jupyter and it'll give you predictions that'll get 100\% in the contest. I don't yet have that model, but I've come up with a process to be able to validate new models that I decide to try out where I can know whether or not it's improving on the previous ones. So rather, this is more of a post about how to come up with your own method to use in Kaggle.

Despite all the code here pretty much amounting to of a bunch of library calls yielding a bunch of three- and five-liners, the entire process is something that requires quite a bit of refining and relies on quite a bit of background knowledge about working with data. In here we had to deal with:

* Handling and preprocessing data in a clean way
* Dealing with missing data values
* The semantics of fitting a model and using it to predict on the test data
* Comparing models against each other using cross-validation

The hardest part out of all of these for me was the preprocessing, as that required familiarity with `sklearn` and `pandas` that I didn't have at the outset. But without having the intuition for the other parts it's perilously easy to get lost or continue making mistakes; this is especially true of what I discussed in the cross-validation section. For me, knowing what to do at each of these situations wasn't entirely obvious and there were times when I mainly relied on stuff I picked up from various courses in statistics and machine learning that I took in the past.

I hope this post helped out if you're in a similar situation, stuck at any of the places that I previously got stuck on or are similarly starting out! I honestly still consider myself a noob at all of this so if there's something that needs fixing in this post or you have any other words whatsoever about it, please don't hesitate to [contact me](/about)!
