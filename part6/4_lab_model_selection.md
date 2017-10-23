Install ISLR
~~~R
library(ISLR)
summary(Hitters)
~~~

There are some missing values here, so before we processed we will remove them:
~~~R
Hitters=na.omit(Hitters)
sum(is.na(Hitters$Salary))
~~~

## Best subset regression
We will now use the package `leaps` to evaluate all the best-subset models:
~~~R
library(leaps)
regfit.full=regsubset(Salary~., data=Hitters)
summary(regfit.full)
~~~

It gives by default best-subset up to size 8; let's increases that to 19, i.e. all the variables
~~~R
regfit.full=regsubsets(Salary~。， data=Hitters, nvmax=19)
reg.summary=summary(regfit.full)
names(reg.summary)
plot(reg.summary$cp, xlab="Number of Variables", ylab="Cp")
which.min(reg.summary$cp)
~~~

There is a plot method for the `regsubsets` object
~~~R
plot(regfit.full, scale="Cp")
coef(regfit.full, 10)
~~~


## Forward Stepwise Selection
Forward stepwise is a greedy algorithm.Each time it includes the next best variable. But it produces a nested sequence.So it's a much less adventurous search.

Here we use the `regsubsets` function but specify the **method="forward"** option:
~~~R
regfit.fwd=regsubset(Salary~., data=Hitters, nvmax=19, method="forward")
summary(regfit.fwd)
plot(regfit.fwd, scale=“Cp”)
~~~

## Model Selection Using a Validation Set
Lets make a training and validation set, so that we can choose a good subset model.We will do it using a slightly different approach from what was done in the book.
~~~R
dim(Hitters)
263 20

set.seed(1)
train=sample(seq(263), 180, replace=FALSE)
regfit.fwd=regsubset(Salary~., data=Hitters[train,], nvmax=19, method="forward")
~~~

Now we will make predictions on the observations not used for training.

We know there are 19 models, so we set up some vectors to record the errors, we have to do a bit of work here, because there is no predict method for `regsubset`
~~~R
val.errors=rep(NA, 19)
x.test=model.matrix(Salary~., data=Hitters[-train,])  # notice the -index!
for(i in 1:19){
    coef=coef(regfit.fwd, id=i)
    pred=x.test[, names(coefi)]%*%coefi
    val.errors[i]=mean((Hitters$Salary[-train]-pred)^2)
}
plot(sqrt(val.erros), ylab="Root MSE", ylim=c(300, 400), pch=19, type="b")
points(sqrt(regfit.fwd$rss), col="blue", pch=19, type="b")
legend("topright", legend=c("Trainging","Validation"), col=c("blue", "black"), pch=19)
~~~

> As we expect, the training error goes down monotonically as the model gets bigger, but not so for the validation error.

This was a little tedious - not having a predict method for `regsubset`, so we will write one!
~~~R
predict.regsubset=function(object, newdata, id,...){
  form=as.formula(object$call[[2]])
  mat=model.matrix(form.newdata) # the test matrix
  coefi=coef(object, id=id)
  mat[, names(coefi)]%*%coefi
}
~~~

## Model Selection by Cross-Validation
We will do 10-fold cross-validation, its really easy!
~~~R
set.seed(1)
folds=sample(rep(1:10), length=nrow(Hitters))
tables(fold)

1  2  3  4  5  6  7  8  9 10
27 27 27 26 26 26 26 26 26 26

cv.errors=matrix(NA, 10, 19)
for(k in 1:10){
  best.fit=regsubset(Salary~., data=Hitters[fold!=k,], nvmax=19, method="foward")
  for(i in 1:19){
    pre=predict(best.fit, Hitters[fold==k,], id=i)
    cv.errors[k, i]=mean((Hitters$Salary[fold==k]-pred)^2)
  }
}

#use the apply function to the columns, average down the columns
rmse.cv=sqrt(apply(cv.errors,2,mean))
plot(rmse.cv, pch=19, type="b")
~~~

## Ridge Regression and the Lasso
We will use the package `glmnet`, which does not use the module formula language, so we will set up an `x` and `y`.
~~~R
library(glmnet)
x=model.matrix(Salary~.-1, data=Hitters)
y=Hitters$Salary
~~~
First we will fit a ridge-regression model. This is achieved by calling `glmnet` with `alpha=0`.

There is also a `cv.glmnet` function which will do the cross-validation for us.
~~~R
fit.ridge=glmnet(x,y,alpha=0)
plot(fit.ridge, xvar="lambda",label=TRUE)
cv.ridge=cv.glmnet(x,y,alpha=0)
plot(cv.ridge)
~~~

Now we fit a lasso model; for this we use the default `alpha=1`
~~~R
fit.lasso=glmnet(x,y)
plot(fit.lasso, xvar="lambda", label=TRUE)
#plot(fit.lasso, xvar="dev", label=TRUE)
cv.lasso=cv.glmnet(x,y)
plot(cv.lasso)
coef(cv.lasso)
~~~

Suppose we want to use our earlier train/validation division to select the `lambda` for the lasso.

~~~R
lasso.tr=glmnet(x[train], y[train])
lasso.tr
pred=predict(lasso.tr, x[-train,])
dim(pred)
rmse=sqrt(apply((y[-train]-repd)^2, 2, mean))
plot(log(lasso.tr$lambda), rmse, type="b", xlab="Log(labmda)")
lam.best=lasso.tr$lambda[order(rmse[1])]
lam.best
coef(lasso.trm s=lam.best)
~~~