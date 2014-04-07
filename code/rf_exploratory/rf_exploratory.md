Random forest exploratory analysis
========================================================

Load the data and metadata. In this analysis, I will be using the cleaned RPKM data.

Load libraries:

```r
library(kernlab)
library(cvTools)
```

```
## Loading required package: lattice
## Loading required package: robustbase
```

```r
library(randomForest)
```

```
## randomForest 4.6-7
## Type rfNews() to see new features/changes/bug fixes.
```

```r
library(edgeR)
```

```
## Loading required package: limma
```

```r
library(caret)
```

```
## Warning: package 'caret' was built under R version 3.0.3
```

```
## Loading required package: ggplot2
```

```r
library(plyr)
library(limma)
library(VennDiagram)
```

```
## Loading required package: grid
```



```r
rDes <- read.delim("../../data/experimental_design_cleaned.txt")
rownames(rDes) <- rDes$TCGA_patient_id
rDat <- read.delim("../../data/aml.rnaseq.gaf2.0_rpkm_cleaned.txt", row.names = 1, 
    check.names = FALSE)
```



## 1) Functions for cross-validation

```r
# Function to select features using linear models input.dat: training data
# set input.labels: true outcomes for training data
fs.lm <- function(input.dat, input.labels) {
    norm.factor <- calcNormFactors(input.dat)
    design <- model.matrix(~input.labels)
    colnames(design) <- c("Intercept", "Label")
    dat.voomed <- voom(input.dat, design, lib.size = colSums(input.dat) * norm.factor)
    fit <- lmFit(dat.voomed, design)
    ebFit <- eBayes(fit)
    hits <- topTable(ebFit, n = Inf, coef = "Label")
    train.features <- rownames(hits)[1:50]
    return(train.features)
}
```



```r
# Function to choose features by selecting the top loadings in PCA
# input.dat: training data set input.labels: true outcomes for training data
fs.pca <- function(input.dat, input.labels) {
    pca.res <- prcomp(t(input.dat))
    top.features <- sort(abs(pca.res$rotation[, 1]), decreasing = TRUE)[1:10]
    return(names(top.features))
}
```



```r
# Function to choose features using correlations to outcomes input.dat:
# training data set input.labels: true outcomes for training data
fs.corr <- function(input.dat, input.labels) {
    gene.corrs <- apply(input.dat, 1, function(x) return(suppressWarnings(cor(x, 
        as.numeric(input.labels)))))
    gene.corrs <- gene.corrs[order(abs(gene.corrs), na.last = NA, decreasing = TRUE)]
    return(names(gene.corrs[1:10]))
}
```



```r
# Function to run k-fold cross validation with a random forest all.dat: all
# data used in the analysis all.labels: true outcomes for the data K: number
# of folds to use in CV
rf.cv <- function(all.dat, all.labels, K = 5, fs.method = "lm") {
    set.seed(540)
    folds <- cvFolds(ncol(all.dat), K = K)
    
    conf_matrix <- matrix(0, nrow = 2, ncol = 2, dimnames = list(c("trueA", 
        "trueB"), c("predA", "predB")))
    feature.list <- list()
    
    for (f in 1:K) {
        train.samples <- folds$subsets[folds$which != f, ]
        test.samples <- folds$subsets[folds$which == f, ]
        
        if (fs.method == "lm") {
            train.features <- fs.lm(all.dat[, train.samples], all.labels[train.samples])
        } else if (fs.method == "pca") {
            train.features <- fs.pca(all.dat[, train.samples], all.labels[train.samples])
        } else if (fs.method == "corr") {
            train.features <- fs.corr(all.dat[, train.samples], all.labels[train.samples])
        }
        feature.list[[f]] <- train.features
        
        train.dat <- data.frame(class = all.labels[train.samples], t(all.dat[train.features, 
            train.samples]))
        test.dat <- data.frame(t(all.dat[train.features, test.samples]))
        test.labels <- all.labels[test.samples]
        
        fit.rf <- randomForest(class ~ ., train.dat)
        pred.rf <- predict(fit.rf, newdata = test.dat, type = "response")
        
        results <- table(test.labels, pred.rf)
        
        conf_matrix[1, 1] <- conf_matrix[1, 1] + results[1, 1]
        conf_matrix[1, 2] <- conf_matrix[1, 2] + results[1, 2]
        conf_matrix[2, 1] <- conf_matrix[2, 1] + results[2, 1]
        conf_matrix[2, 2] <- conf_matrix[2, 2] + results[2, 2]
    }
    
    rf.sens <- conf_matrix[1, 1]/sum(conf_matrix[1, ])
    rf.spec <- conf_matrix[2, 2]/sum(conf_matrix[2, ])
    rf.acc <- (conf_matrix[1, 1] + conf_matrix[2, 2])/sum(conf_matrix)
    names(feature.list) <- c(1:length(feature.list))
    
    return(list(acc = rf.acc, sens = rf.sens, spec = rf.spec, feature.list = feature.list))
}
```



## 2) Train a random forest to predict cytogenetic risk

Set up the data. Remove samples where the cytogenetic risk category is not determined, and set categories to poor and not poor:

```r
rfDes <- rDes[rDes$Cytogenetic_risk != "N.D.", ]
rf.labels <- mapvalues(rfDes$Cytogenetic_risk, c("Good", "Intermediate", "Poor"), 
    c(0, 0, 1), warn_missing = TRUE)
rf.labels <- factor(rf.labels)
rfDat <- rDat[, rownames(rfDes)]
```


Run a 5-fold cross-validation for the data:

```r
cv.risk.res <- rf.cv(rfDat, rf.labels, K = 5, fs.method = "corr")
cv.risk.res[1:3]
```

```
## $acc
## [1] 0.8125
## 
## $sens
## [1] 0.8955
## 
## $spec
## [1] 0.5476
```

```r
fts <- cv.risk.res[[4]]
common.fts.risk <- intersect(fts[[1]], intersect(fts[[2]], intersect(fts[[3]], 
    intersect(fts[[4]], fts[[5]]))))
```


## 3) Predict the different cytogenetic mutations

First look at trisomy 8:

```r
cv.trisomy8.res <- rf.cv(rDat, factor(as.numeric(rDes$trisomy_8)), K = 5, fs.method = "lm")
cv.trisomy8.res[1:3]
```

```
## $acc
## [1] 0.9497
## 
## $sens
## [1] 0.9875
## 
## $spec
## [1] 0.6316
```

```r
fts <- cv.trisomy8.res[[4]]
common.fts.trisomy8 <- intersect(fts[[1]], intersect(fts[[2]], intersect(fts[[3]], 
    intersect(fts[[4]], fts[[5]]))))
```


Next look at deletions of chromosome 5:

```r
cv.del5.res <- rf.cv(rDat, factor(as.numeric(rDes$del_5)), K = 5, fs.method = "lm")
cv.del5.res[1:3]
```

```
## $acc
## [1] 0.9665
## 
## $sens
## [1] 0.9877
## 
## $spec
## [1] 0.75
```

```r
fts <- cv.del5.res[[4]]
common.fts.del5 <- intersect(fts[[1]], intersect(fts[[2]], intersect(fts[[3]], 
    intersect(fts[[4]], fts[[5]]))))
```


Next look at deletions of chromosome 7:

```r
cv.del7.res <- rf.cv(rDat, factor(as.numeric(rDes$del_7)), K = 5, fs.method = "lm")
cv.del7.res[1:3]
```

```
## $acc
## [1] 0.9553
## 
## $sens
## [1] 0.9873
## 
## $spec
## [1] 0.7143
```

```r
fts <- cv.del7.res[[4]]
common.fts.del7 <- intersect(fts[[1]], intersect(fts[[2]], intersect(fts[[3]], 
    intersect(fts[[4]], fts[[5]]))))
```


Now get a sense of how many genes are shared between these 4 different sets:

```r
fts.all <- list(risk = common.fts.risk, trisomy8 = common.fts.trisomy8, del5 = common.fts.del5, 
    del7 = common.fts.del7)
plot.new()
venn.plot <- venn.diagram(fts.all, filename = NULL, fill = c("red", "yellow", 
    "blue", "green"))
grid.draw(venn.plot)
```

![plot of chunk unnamed-chunk-12](figure/unnamed-chunk-12.png) 

