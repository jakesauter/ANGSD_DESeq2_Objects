# **DESeq Objects**

A report by **Jake Sauter**

date: **3/14/2021**


## **Background: Creating DESeq2 Objects**

The code below uses the results of `featureCounts` (Unix command line tool) in order to create `DESeq2` objects consisting of the original feature count information in a `DESeqDataSet` (`DESeq.ds`)


```r
library(DESeq2)
library(magrittr)

readcounts <-
  read.table('data/featCounts_Gierlinski_genes.txt',
             header = TRUE)

orig_names <- names(readcounts)
names(readcounts) <-
  gsub(".*(WT|SNF2)(_[0-9]+).*", "\\1\\2", orig_names)

## gene IDs should be stored as row.names
row.names(readcounts) <- make.names(readcounts$Geneid)

## exclude the columns without read counts (columns 1 to 6 contain additional
## info such as genomic coordinates)
readcounts <- readcounts[ ,-c(1:6)]

sample_info <- DataFrame(
  condition = gsub("_[0-9]+", "",names(readcounts)),
  row.names =names(readcounts))

DESeq.ds <-DESeqDataSetFromMatrix(
  countData = as.matrix(readcounts),
  colData   = sample_info,
  design    = ~condition)

# Remove genes with no reads
keep_genes <- rowSums(counts(DESeq.ds)) > 0
DESeq.ds <- DESeq.ds[keep_genes, ]

# rlog normalize the data
DESeq.rlog <- rlog(DESeq.ds, blind=TRUE)
rlog.norm.counts <- assay(DESeq.rlog)
```

## **Inspecting DESeq2 Objects**

From the analyses below, we extract the following similarities and differences between the objects returned by `rlog()` and `DESeqDataSetFromMatrix()`

**Similarities**:

-   Both object types returned inherit from the `SummarizedExperiment::SummarizedExperiment` class

-   Both object types share many common slots because of this and allows easy access to metadata about the experiment

**Differences**:

-   `DESeqDataSetFromMatrix()` returns and object of type `DESeqDataSet` while `rlog()` returns an object of type `DESeqTransform`

-   `DESeqDataSet` objects are great at facilitating input processing and normalization functions like `rlog()`, while `DESeqTranform` objects are more result-oriented objects that can be used with functions like `plotPCA()`

### **Classes**

In order to view the classes of `DESeq2` objects, we will make use of the `class()` and `showClass()` built-in functions. `showClass` enables us to easily view the name and class of the **slots** of our objects.


```r
DESeq.ds %>% 
  class() %>% 
  showClass()
```

```
Class "DESeqDataSet" [package "DESeq2"]

Slots:
                                                                
Name:                        design           dispersionFunction
Class:                          ANY                     function
                                                                
Name:                     rowRanges                      colData
Class: GenomicRanges_OR_GRangesList                    DataFrame
                                                                
Name:                        assays                        NAMES
Class:                       Assays            character_OR_NULL
                                                                
Name:               elementMetadata                     metadata
Class:                    DataFrame                         list

Extends: 
Class "RangedSummarizedExperiment", directly
Class "SummarizedExperiment", by class "RangedSummarizedExperiment", distance 2
Class "Vector", by class "RangedSummarizedExperiment", distance 3
Class "Annotated", by class "RangedSummarizedExperiment", distance 4
Class "vector_OR_Vector", by class "RangedSummarizedExperiment", distance 4
```


```r
class(DESeq.rlog) %>% 
  showClass()
```

```
Class "DESeqTransform" [package "DESeq2"]

Slots:
                                                                
Name:                     rowRanges                      colData
Class: GenomicRanges_OR_GRangesList                    DataFrame
                                                                
Name:                        assays                        NAMES
Class:                       Assays            character_OR_NULL
                                                                
Name:               elementMetadata                     metadata
Class:                    DataFrame                         list

Extends: 
Class "RangedSummarizedExperiment", directly
Class "SummarizedExperiment", by class "RangedSummarizedExperiment", distance 2
Class "Vector", by class "RangedSummarizedExperiment", distance 3
Class "Annotated", by class "RangedSummarizedExperiment", distance 4
Class "vector_OR_Vector", by class "RangedSummarizedExperiment", distance 4
```

**Observed differences:**

-   `DESeqDataSetFromMatrix()` returns an object of class `DESeqDataSet` while `rlog()` returns an object of class `DESeqTransform`
-   As a part in the differences in class type, these objects have different **slots** that store differing information, such as how class `DESeqDataSet` has a `dispersionFunction` slot
-   Both of the objects have a `rowRanges` and `colData` slot, most likely due to inheritance from a shared super-class of `SummarizedExperiment`

## **Inspecting Slots Directly**

We noticed a few differences based on the **slots** of our objects. In order to better observe these differences, we can directly access the names of the slots with `slotNames()` as well as make use of set based operations like `intersect()` and `setdiff()`


```r
DESeq.ds %>% 
    slotNames()
```

```
[1] "design"             "dispersionFunction" "rowRanges"         
[4] "colData"            "assays"             "NAMES"             
[7] "elementMetadata"    "metadata"          
```


```r
DESeq.rlog %>% 
  slotNames()
```

```
[1] "rowRanges"       "colData"         "assays"          "NAMES"          
[5] "elementMetadata" "metadata"       
```

**Shared Slots**


```r
ds_slots <- slotNames(DESeq.ds)
rlog_slots <- slotNames(DESeq.rlog)
intersect(ds_slots, rlog_slots)
```

```
[1] "rowRanges"       "colData"         "assays"          "NAMES"          
[5] "elementMetadata" "metadata"       
```

Above we see that `DESeqDataSet` and `DESeqTransform` share more slots than previously pointed out.

**Unique slots to `DESeqDataSet`**:


```r
setdiff(ds_slots, rlog_slots)
```

```
[1] "design"             "dispersionFunction"
```

Here we see specifically that

**Unique slots to `DESeqTransform`**


```r
setdiff(rlog_slots, ds_slots)
```

```
character(0)
```

Interestingly, we see here that `DESeqTransform` holds no unique slots in relation to `DESeqDataSet`.

### **Downstream Functions**

`DESeqDataSet`:

-   Great for accessing `counts()`, `assays()`, `colData()` , `rowData()`


```r
help(DESeqDataSet)
```

> </font size=2.5> DESeqDataSet is a subclass of RangedSummarizedExperiment, **used to store the input values, intermediate calculations and results of an analysis of differential expression**. The DESeqDataSet class enforces non-negative integer values in the "counts" matrix stored as the first element in the assay list. In addition, a formula which specifies the design of the experiment must be provided. The constructor functions create a DESeqDataSet object from various types of input: a RangedSummarizedExperiment, a matrix, count files generated by the python package HTSeq, or a list from the tximport function in the tximport package. See the vignette for examples of construction from different types. </font>

`DESeqTransform` :

-   We can use our `DESeqTransform` object for the `plotPCA()` function as well as other downstream normalized-data expecting functions.


```r
help(DESeqTransform)
```

> </font size=2.5> This constructor function would not typically be used by "end users". This simple class extends the RangedSummarizedExperiment class of the SummarizedExperiment package. It is used by rlog and varianceStabilizingTransformation **to wrap up the results into a class for downstream methods, such as plotPCA.** </font>

## **Extracting Expression Values**

`DESeqDataSet`:


```r
library(knitr)

DESeq.ds %>% 
  counts() %>% 
  head() %>% 
  kable()
```



|          | SNF2_1| SNF2_2| SNF2_3| SNF2_4| SNF2_5| WT_1| WT_2| WT_3| WT_4| WT_5|
|:---------|------:|------:|------:|------:|------:|----:|----:|----:|----:|----:|
|YAL012W   |   7351|   7180|   7648|   8119|   5944| 4312| 3767| 3040| 5604| 4167|
|YAL068C   |      2|      2|      2|      1|      0|    0|    0|    0|    2|    2|
|YAL067C   |    103|     51|     44|     90|     53|   12|   23|   21|   30|   29|
|YAL066W   |      2|      0|      0|      0|      0|    0|    0|    0|    0|    0|
|YAL065C   |      5|      9|      6|      3|      1|   10|    5|    2|    4|    3|
|YAL064W.B |     13|      8|     10|      9|      6|    9|   12|    4|    4|    8|

`DESeqTranfrom`:

By observing the `help(DESeqTransform)` page, we can see that

> <font size=2.5> [If] a DESeqTransform if a DESeqDataSet was provided, or a matrix if a count matrix was provided as input. Note that for DESeqTransform output, the matrix of transformed values is stored in assay(rld). To avoid returning matrices with NA values, in the case of a row of all zeros, the rlog transformation returns zeros (essentially adding a pseudocount of 1 only to these rows). </font>


```r
DESeq.rlog %>% 
  assay() %>% 
  head() %>% 
  kable()
```



|          |     SNF2_1|     SNF2_2|     SNF2_3|     SNF2_4|     SNF2_5|       WT_1|       WT_2|       WT_3|       WT_4|       WT_5|
|:---------|----------:|----------:|----------:|----------:|----------:|----------:|----------:|----------:|----------:|----------:|
|YAL012W   | 12.3627818| 12.5646933| 12.6276243| 12.4168422| 12.5900380| 12.6575541| 12.1462787| 12.1393157| 12.3228233| 12.2700097|
|YAL068C   |  0.0051747|  0.0160872|  0.0163863| -0.0180524| -0.0357003| -0.0316208| -0.0362454| -0.0341909|  0.0148639|  0.0258891|
|YAL067C   |  5.6762464|  5.3899421|  5.3134854|  5.5626038|  5.5358528|  5.0498885|  5.0927354|  5.1450406|  5.1172067|  5.2098866|
|YAL066W   | -1.8504800| -1.8715771| -1.8715401| -1.8732068| -1.8703548| -1.8681275| -1.8706686| -1.8695099| -1.8717291| -1.8704087|
|YAL065C   |  2.2285958|  2.3574180|  2.2861376|  2.1806517|  2.1671793|  2.5125413|  2.2814640|  2.2121076|  2.2310898|  2.2287585|
|YAL064W.B |  3.0509871|  3.0029311|  3.0488029|  2.9702903|  2.9925436|  3.1580736|  3.1289667|  2.9590118|  2.9048027|  3.0430814|

### **Adding Custom Matrix**

In the following exercise, we will add a custom matrix named `my_personal_normalization` to our `DESeq.ds` object using three different methods.

**Bioconductor Method**

This is possible through the use of the `assay()` accessor and setter function.


```r
assay(DESeq.ds, "my_personal_normalization") <- 
  assay(DESeq.ds, 'counts')

DESeq.ds %>% 
  assays()
```

```
List of length 2
names(2): counts my_personal_normalization
```

**S4 Inheritance Method**

We may also choose to extend the S4 class with our own sub-class in order to facilitate the use of a **slot**.


```r
setClass(
  "myDESeqDataSet",
  contains="DESeqDataSet",
  slots=c(my_personal_normalization="matrix")
) -> myDESeqDataSet

DESeq.ds.mine <- as(DESeq.ds,"myDESeqDataSet")
DESeq.ds.mine@my_personal_normalization <- 
  counts(DESeq.ds)

DESeq.ds.mine %>% 
  class() %>% 
  showClass()
```

```
Class "myDESeqDataSet" [in ".GlobalEnv"]

Slots:
                                                                
Name:     my_personal_normalization                       design
Class:                       matrix                          ANY
                                                                
Name:            dispersionFunction                    rowRanges
Class:                     function GenomicRanges_OR_GRangesList
                                                                
Name:                       colData                       assays
Class:                    DataFrame                       Assays
                                                                
Name:                         NAMES              elementMetadata
Class:            character_OR_NULL                    DataFrame
                                   
Name:                      metadata
Class:                         list

Extends: 
Class "DESeqDataSet", directly
Class "RangedSummarizedExperiment", by class "DESeqDataSet", distance 2
Class "SummarizedExperiment", by class "DESeqDataSet", distance 3
Class "Vector", by class "DESeqDataSet", distance 4
Class "Annotated", by class "DESeqDataSet", distance 5
Class "vector_OR_Vector", by class "DESeqDataSet", distance 5
```

**Quick and Dirty Method**

Finally, we can manually set our matrix to be an attribute of the original object.


```r
attr(DESeq.ds, "my_personal_normalization") <- 
  counts(DESeq.ds)

DESeq.ds %>% 
  attributes() %>% 
  names()
```

```
 [1] "design"                    "dispersionFunction"       
 [3] "rowRanges"                 "colData"                  
 [5] "assays"                    "NAMES"                    
 [7] "elementMetadata"           "metadata"                 
 [9] "class"                     "my_personal_normalization"
```

## **Inspecting DESeq2 Source Code**

In the following code blocks we will be using the `::` operator to access functions and objects that **are exported** from the `DESeq2` pacakge, as well as using the `:::` operator to access functions and objects that **are not exported** from the package.

### **rlog()**


```r
DESeq2::rlog
```

```
function (object, blind = TRUE, intercept, betaPriorVar, fitType = "parametric") 
{
    n <- ncol(object)
    if (n >= 30 & n < 50) {
        message("rlog() may take a few minutes with 30 or more samples,\nvst() is a much faster transformation")
    }
    else if (n >= 50) {
        message("rlog() may take a long time with 50 or more samples,\nvst() is a much faster transformation")
    }
    if (is.null(colnames(object))) {
        colnames(object) <- seq_len(ncol(object))
    }
    if (is.matrix(object)) {
        matrixIn <- TRUE
        object <- DESeqDataSetFromMatrix(object, DataFrame(row.names = colnames(object)), 
            ~1)
    }
    else {
        matrixIn <- FALSE
    }
    if (is.null(sizeFactors(object)) & is.null(normalizationFactors(object))) {
        object <- estimateSizeFactors(object)
    }
    if (blind) {
        design(object) <- ~1
    }
    if (missing(intercept)) {
        sparseTest(counts(object, normalized = TRUE), 0.9, 100, 
            0.1)
    }
    if (blind | is.null(mcols(object)$dispFit)) {
        if (is.null(mcols(object)$baseMean)) {
            object <- getBaseMeansAndVariances(object)
        }
        object <- estimateDispersionsGeneEst(object, quiet = TRUE)
        object <- estimateDispersionsFit(object, fitType, quiet = TRUE)
    }
    if (!missing(intercept)) {
        if (length(intercept) != nrow(object)) {
            stop("intercept should be as long as the number of rows of object")
        }
    }
    rld <- rlogData(object, intercept, betaPriorVar)
    if (matrixIn) {
        return(rld)
    }
    se <- SummarizedExperiment(assays = rld, colData = colData(object), 
        rowRanges = rowRanges(object), metadata = metadata(object))
    dt <- DESeqTransform(se)
    attr(dt, "betaPriorVar") <- attr(rld, "betaPriorVar")
    if (!is.null(attr(rld, "intercept"))) {
        mcols(dt)$rlogIntercept <- attr(rld, "intercept")
    }
    dt
}
<bytecode: 0x558945059d80>
<environment: namespace:DESeq2>
```

### **estimateDispersions**

Upon inspection of `estimateDispersions`, we can see that is is actually a **standardGeneric** function, meaning that the heavy-lifting code for each possible input data type is located within a data-type specific function.


```r
DESeq2::estimateDispersions
```

```
standardGeneric for "estimateDispersions" defined from package "BiocGenerics"

function (object, ...) 
standardGeneric("estimateDispersions")
<bytecode: 0x55892968a320>
<environment: 0x558929694208>
Methods may be defined for arguments: object
Use  showMethods("estimateDispersions")  for currently available ones.
```

Specifically for this task, we are interested in what happens when we call `estimateDispersions` with a `DESeqDataSet` type


```r
DESeq2:::estimateDispersions.DESeqDataSet
```

```
function (object, fitType = c("parametric", "local", "mean"), 
    maxit = 100, quiet = FALSE, modelMatrix = NULL, minmu = 0.5) 
{
    if (!.hasSlot(object, "rowRanges")) 
        object <- updateObject(object)
    if (is.null(sizeFactors(object)) & is.null(normalizationFactors(object))) {
        stop("first call estimateSizeFactors or provide a normalizationFactor matrix before estimateDispersions")
    }
    if (!is.null(sizeFactors(object))) {
        if (!is.numeric(sizeFactors(object))) {
            stop("the sizeFactor column in colData is not numeric.\nthis column could have come in during colData import and should be removed.")
        }
        if (any(is.na(sizeFactors(object)))) {
            stop("the sizeFactor column in colData contains NA.\nthis column could have come in during colData import and should be removed.")
        }
    }
    if (all(rowSums(counts(object) == counts(object)[, 1]) == 
        ncol(object))) {
        stop("all genes have equal values for all samples. will not be able to perform differential analysis")
    }
    if (!is.null(dispersions(object))) {
        if (!quiet) 
            message("found already estimated dispersions, replacing these")
        mcols(object) <- mcols(object)[, !(mcols(mcols(object))$type %in% 
            c("intermediate", "results")), drop = FALSE]
    }
    stopifnot(length(maxit) == 1)
    fitType <- match.arg(fitType, choices = c("parametric", "local", 
        "mean"))
    checkForExperimentalReplicates(object, modelMatrix)
    if (!quiet) 
        message("gene-wise dispersion estimates")
    object <- estimateDispersionsGeneEst(object, maxit = maxit, 
        quiet = quiet, modelMatrix = modelMatrix, minmu = minmu)
    if (!quiet) 
        message("mean-dispersion relationship")
    object <- estimateDispersionsFit(object, fitType = fitType, 
        quiet = quiet)
    if (!quiet) 
        message("final dispersion estimates")
    object <- estimateDispersionsMAP(object, maxit = maxit, quiet = quiet, 
        modelMatrix = modelMatrix)
    return(object)
}
<bytecode: 0x558945866e90>
<environment: namespace:DESeq2>
```

### **rlogData()**

Below, we must use `:::` in order to access `rlogData()` as it is not an exported function.


```r
DESeq2:::rlogData
```

```
function (object, intercept, betaPriorVar) 
{
    if (is.null(mcols(object)$dispFit)) {
        stop("first estimate dispersion")
    }
    samplesVector <- as.character(seq_len(ncol(object)))
    if (!missing(intercept)) {
        if (length(intercept) != nrow(object)) {
            stop("intercept should be as long as the number of rows of object")
        }
    }
    if (is.null(mcols(object)$allZero) | is.null(mcols(object)$baseMean)) {
        object <- getBaseMeansAndVariances(object)
    }
    samplesVector <- factor(samplesVector, levels = unique(samplesVector))
    if (missing(intercept)) {
        samples <- factor(c("null_level", as.character(samplesVector)), 
            levels = c("null_level", levels(samplesVector)))
        modelMatrix <- stats::model.matrix.default(~samples)[-1, 
            ]
        modelMatrixNames <- colnames(modelMatrix)
        modelMatrixNames[modelMatrixNames == "(Intercept)"] <- "Intercept"
    }
    else {
        samples <- factor(samplesVector)
        if (length(samples) > 1) {
            modelMatrix <- stats::model.matrix.default(~0 + samples)
        }
        else {
            modelMatrix <- matrix(1, ncol = 1)
            modelMatrixNames <- "samples1"
        }
        modelMatrixNames <- colnames(modelMatrix)
        if (!is.null(normalizationFactors(object))) {
            nf <- normalizationFactors(object)
        }
        else {
            sf <- sizeFactors(object)
            nf <- matrix(rep(sf, each = nrow(object)), ncol = ncol(object))
        }
        intercept <- as.numeric(intercept)
        infiniteIntercept <- !is.finite(intercept)
        intercept[infiniteIntercept] <- -10
        normalizationFactors(object) <- nf * 2^intercept
        mcols(object)$allZero <- infiniteIntercept
    }
    objectNZ <- object[!mcols(object)$allZero, ]
    stopifnot(all(!is.na(mcols(objectNZ)$dispFit)))
    if (missing(betaPriorVar)) {
        logCounts <- log2(counts(objectNZ, normalized = TRUE) + 
            0.5)
        logFoldChangeMatrix <- logCounts - log2(mcols(objectNZ)$baseMean + 
            0.5)
        logFoldChangeVector <- as.numeric(logFoldChangeMatrix)
        varlogk <- 1/mcols(objectNZ)$baseMean + mcols(objectNZ)$dispFit
        weights <- 1/varlogk
        betaPriorVar <- matchWeightedUpperQuantileForVariance(logFoldChangeVector, 
            rep(weights, ncol(objectNZ)))
    }
    stopifnot(length(betaPriorVar) == 1)
    lambda <- 1/rep(betaPriorVar, ncol(modelMatrix))
    if ("Intercept" %in% modelMatrixNames) {
        lambda[which(modelMatrixNames == "Intercept")] <- 1e-06
    }
    fit <- fitNbinomGLMs(object = objectNZ, modelMatrix = modelMatrix, 
        lambda = lambda, renameCols = FALSE, alpha_hat = mcols(objectNZ)$dispFit, 
        betaTol = 1e-04, useOptim = FALSE, useQR = TRUE)
    normalizedDataNZ <- t(modelMatrix %*% t(fit$betaMatrix))
    normalizedData <- buildMatrixWithZeroRows(normalizedDataNZ, 
        mcols(object)$allZero)
    if (!missing(intercept)) {
        normalizedData <- normalizedData + ifelse(infiniteIntercept, 
            0, intercept)
    }
    colnames(normalizedData) <- colnames(object)
    attr(normalizedData, "betaPriorVar") <- betaPriorVar
    if ("Intercept" %in% modelMatrixNames) {
        fittedInterceptNZ <- fit$betaMatrix[, which(modelMatrixNames == 
            "Intercept"), drop = FALSE]
        fittedIntercept <- buildMatrixWithNARows(fittedInterceptNZ, 
            mcols(object)$allZero)
        fittedIntercept[is.na(fittedIntercept)] <- -Inf
        attr(normalizedData, "intercept") <- fittedIntercept
    }
    normalizedData
}
<bytecode: 0x5589326df218>
<environment: namespace:DESeq2>
```
