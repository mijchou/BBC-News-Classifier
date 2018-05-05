
# BBC-News-Classifier
KNN &amp; SVM classifiers on BBC News Categories

This work aims to build a News classifier, to identify News from 5 categories: business, entertainment, politics, sport and tech. Two classifying techniques, KNN and SVM, are applied for comparisons. The classifier is built upon 2225 [BBC News Datasets](http://mlg.ucd.ie/datasets/bbc.html) from 2005-2006. Datasets can be found under the folder __bbc-fulltext__.

**R Packages used:**
* tm: Text-Mining Package
* plyr: Tools for Splitting, Applying and Combining Data
* class: Functions for Classification __(used: knn)__
* e1071: Functions for latent class analysis, fuzzy clustering, svm, bagged clustering, naive Bayes classifier... __(used: svm)__

**Sources:<br/>**

[How to Build a Text Mining, Machine Learning Document Classification System in R!](https://www.youtube.com/watch?v=j1V2McKbkLo) <br/>
[Package ‘tm’](https://cran.r-project.org/web/packages/tm/tm.pdf) <br/>
[The caret Package](https://topepo.github.io/caret/) <br/>

News Classifier
================
Miriam Chou
12 April 2018

## Setup
=======

1. Load libraries
``` r
libs <- c("tm", "plyr", "class", "e1071")
lapply(libs, require, character.only = TRUE)
```

2. Set options - do not read string as factors
``` r
options(stringAsFactors = FALSE)
```

3. Define 5 categories & specify the path of data
``` r
categories <- c("business", "entertainment",
                "politics", "sport", "tech")
pathname <- "../bbc-fulltext"
```

## Text cleaning
==========

Create **cleanCorpus()** to clean the text data

1. Remove punctuation
2. Remove white space
3. Convert all words to lowercase
4. Remove words - **stopwords()** specifies the group of predefined stopwords to be removed
<br/> i.e. **stopwords("English")** = the, is, at, which, and on

``` r
cleanCorpus <- function(corpus) {
  corpus.tmp <- tm_map(corpus, removePunctuation)
  corpus.tmp <- tm_map(corpus.tmp, stripWhitespace)
  corpus.tmp <- tm_map(corpus.tmp, tolower)
  corpus.tmp <- tm_map(corpus.tmp, removeWords, stopwords("english"))
  return(corpus.tmp)
}
```

##  Build Term-Document Matrix (TDM)
=========

A Term-Document Matrix describes the frequency of terms in the collection of documents. <br/>

1. Read in data from prespecified path <br/>
2. Apply **cleanCorpus()** created from the previous step <br/>
3. Apply **TermDocumentMatrix()** to create the resulting term-document matrix <br/>
4. *Sparsity* specifies the percentage of emptiness a word occurs in documents. <br/>
i.e. a term occurring 0 times in 70/100 documents = the term has a sparsity of 0.7 <br/>
We wish to remove words with sparsity > 0.7. <br/>
i.e. The resulting matrix retains words occuring in 30% of documents or more. ***for me to double check***

``` r
generateTDM <- function(cate, path) {
  s.dir <- sprintf("%s/%s", path, cate)
  s.cor <- Corpus(DirSource(directory = s.dir))
  s.cor.cl <- cleanCorpus(s.cor)
  s.tdm <- TermDocumentMatrix(s.cor.cl)
  
  s.tdm <- removeSparseTerms(s.tdm, 0.7) # setting sparsity threshold
  result <- list(name = cate, tdm = s.tdm)
}

tdm <- lapply(categories, generateTDM, path = pathname)
```

## Attach names
===========

``` r
bindCategoryToTDM <- function(tdm) {
  s.mat <- t(data.matrix(tdm[["tdm"]]))
  s.df <- as.data.frame(s.mat, stringsAsFactors = F)
  s.df <- cbind(s.df, rep(tdm[["name"]], nrow(s.df)))
  colnames(s.df)[ncol(s.df)] <- "targetcategory"
  return(s.df)
}

cateTDM <- lapply(tdm, bindCategoryToTDM)
```

stack
=====

``` r
tdm.stack <- do.call(rbind.fill, cateTDM)
tdm.stack[is.na(tdm.stack)] <- 0
```

hold-out
========

``` r
train.idx <- sample(nrow(tdm.stack), ceiling(nrow(tdm.stack) * 0.7))
test.idx <- (1:nrow(tdm.stack)) [- train.idx]
```

``` r
head(test.idx)
```

    ## [1]  5  6 16 18 22 24

``` r
head(train.idx)
```

    ## [1] 1598 1510  998  213 1408 1809

modelling
=========

``` r
tdm.cate <- tdm.stack[, "targetcategory"]
tdm.stack.nl <- tdm.stack[, !colnames(tdm.stack) %in% "targetcategory"]

# KNN
knn.pred <- knn(tdm.stack.nl[train.idx, ], tdm.stack.nl[test.idx, ], tdm.cate[train.idx])
knn.mat <- table("Predictions" = knn.pred, "Actual" = tdm.cate[test.idx])
knn.acc <- sum(diag(knn.mat))/sum(knn.mat)

knn.mat
```

    ##                Actual
    ## Predictions     business entertainment politics sport tech
    ##   business           153            12       12    10    6
    ##   entertainment        7           102        3     7    5
    ##   politics             2             0      101     0    9
    ##   sport                2             5        6   131    3
    ##   tech                 0             0        0     0   91

``` r
knn.acc
```

    ## [1] 0.8665667

``` r
# SVM
svm.fit <- svm(targetcategory~., data = tdm.stack[train.idx, ])
svm.pred <- predict(svm.fit, newdata = tdm.stack.nl[test.idx, ])
svm.mat <- table("Predictions" = svm.pred, "Actual" = tdm.cate[test.idx])
svm.acc <- sum(diag(svm.mat))/sum(svm.mat)

svm.mat
```

    ##                Actual
    ## Predictions     business entertainment politics sport tech
    ##   business           159             3        2     0    0
    ##   entertainment        1           116        0     2    1
    ##   politics             0             0      116     0    0
    ##   sport                2             0        2   141    1
    ##   tech                 2             0        2     5  112

``` r
svm.acc
```

    ## [1] 0.9655172
