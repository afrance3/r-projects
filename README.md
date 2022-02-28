---
title: "Differential Expression analysis Example"
author: "Adam France"
date: "`r Sys.Date()`"
output: html_document
---

## What is Differential Expression analysis?
Lets say you have two phenotypes: a healthy, or wildtype, and non-healthy, or disease type. You want to know 
the difference in these two groups based on what genes are being expressed. Since we can't directly measure this in vivo, we take samples and measure the amount of messenger RNA that corresponds to that organisms known genes. By knowing which genes are being over, or under expressed in a disease state, we can better understand a disease and 
design therapies for specific target(S). 
Specifically for this example, I am using a kidney-fibrosis dataset, where samples where taken from the kidneys in mice from a healthy and disease(fibrosis) group.

The code was adapted from a course on [Data Camp](https://www.datacamp.com) called [RNA-Seq with Bioconductor in r]( https://app.datacamp.com/learn/courses/rna-seq-with-bioconductor-in-r).

The raw data file can be found [here](https://assets.datacamp.com/production/repositories/1766/datasets/bf1d0eff910f1b2cad36e5acdc2a182e95c63965/fibrosis_smoc2_rawcounts_unordered.csv)

Documentation for [DESEq2](http://bioconductor.org/packages/devel/bioc/vignettes/DESeq2/inst/doc/DESeq2.html), the main package used in this exercise.

r version used in this exercise: 
```{r}
R.version.string
```

## The libraries needed for this exercise:
```{r message=FALSE}
library(dplyr)
library(DESeq2)
library(RColorBrewer)
library(pheatmap)
library(tidyverse)
library(forcats)

```
Bioconductor is also needed to install some of these packages. once installed,
you can call `Biocmanager::install()` with the package name in parentheses.


## Load in the raw counts matrix
```{r}
smoc2_rawcounts <- read.csv("fibrosis-raw.txt")
```

Lets take a look at the data:
```{r}
head(smoc2_rawcounts, 5)
```
Okay, everything looks right. We have column (X) which list the names of genes being measured. the other columns are the names of the sample groups, with the raw counts for the corresponding gene in column x.

## Since no metadata file was included, we need to create the meta data table (a 7x3 Dataframe)
DESeq needs to know the exact structure of the sample groups to match with the raw counts matrix.
```{r}
genotype <- c(replicate(7,"smoc2_oe"))
condition <- c(replicate(4, "fibrosis")) 
condition <- append(condition, (replicate(3, "normal"))) 

samples <- c("smoc2_fibrosis1","smoc2_fibrosis2","smoc2_fibrosis3",
             "smoc2_fibrosis4","smoc2_normal1","smoc2_normal3",
             "smoc2_normal4")

smoc2_metadata <- data.frame( genotype, condition ) 
rownames(smoc2_metadata) <- samples
```


Lets check to make sure eveything is lined up:
```{r}
smoc2_metadata
```


## Use the match() function to reorder the columns of the raw counts
```{r}
reorder_idx <- match(rownames(smoc2_metadata), colnames(smoc2_rawcounts))
```

Reorder the columns of the count data
```{r}
reordered_smoc2_rawcounts <- smoc2_rawcounts[,reorder_idx]

head(reordered_smoc2_rawcounts, 5)
```
We can now see the order of columns has changed to match the meta data file we created.



## Create the DESEq Object:
```{r message=FALSE, warning=FALSE, error=FALSE}
dds_smoc2 <- DESeqDataSetFromMatrix(countData = reordered_smoc2_rawcounts,
                                    colData = smoc2_metadata,
                                    design = ~ condition)
```


Determine the size factors to use for normalization
```{r}
dds_smoc2 <- estimateSizeFactors(dds_smoc2)
```

Extract the normalized counts
```{r}
smoc2_normalized_counts <- counts(dds_smoc2, normalized = TRUE) %>% data.frame() 
#there are problems down the line if you dont format this as a data frame
```

## Heirarchical HeatMap
Transform the normalized counts 
```{r}
vsd_smoc2 <- vst(dds_smoc2, blind = TRUE)
```
Extract the matrix of transformed counts
```{r}
vsd_mat_smoc2 <- assay(vsd_smoc2)
```
Compute the correlation values between samples
```{r}
vsd_cor_smoc2 <- cor(vsd_mat_smoc2) 
```
Plot the heatmap
```{r}
pheatmap(vsd_cor_smoc2, annotation = select(smoc2_metadata, condition))
```

## PCA PLot
Plot the PCA of PC1 and PC2
```{r}
plotPCA(vsd_smoc2, intgroup='condition')
```

## Run DESeq22
Run the DESeq2 analysis
```{r message=FALSE}
analysis_object <- DESeq(dds_smoc2)
```

## Plot dispersions
```{r}
plotDispEsts(analysis_object)
```

Extract the results of the differential expression analysis
```{r}
smoc2_res <- results(analysis_object, 
                     contrast = c("condition", "fibrosis", "normal"), 
                     alpha = 0.05)
```


Shrink the log2 fold change estimates to be more accurate
```{r message=FALSE}
smoc2_res <- lfcShrink(analysis_object, 
                       contrast =  c("condition","fibrosis" ,"normal" ),
                       res = smoc2_res, type="ashr") # 'ashr' is a package that may need to be installed

#Extract results
smoc2_res <- results(analysis_object, 
                     contrast = c("condition","fibrosis","normal"), 
                     alpha = 0.05, 
                     lfcThreshold = 0.32)

#Shrink the log2 fold changes
smoc2_res <- lfcShrink(analysis_object, 
                       contrast = c("condition","fibrosis","normal"), 
                       res = smoc2_res, type='ashr') # 'ashr' is a package that may need to be installed
```


### Reviewing the results
```{r}
summary(smoc2_res)
```
if we look at the 'LFC' (log fold change) rows we have about 12% change for both up and down of our non-zero genes. These are sensible numbers given that we are comparing two groups of the same tissue type from the same organism. we started with about 47,000 genes, so roughly 7,000 genes is a good indicator that something is going on. if we were getting much higher percentages we would know something was wrong either in the analysis or in library preparation.  

Save results as a data frame
```{r}
smoc2_res_all <- data.frame(smoc2_res)
```

Subset the results to only return the significant genes with p-adjusted values less than 0.05
```{r}
smoc2_res_sig <- subset(smoc2_res_all, padj < 0.05)
```

# Visualizing results:
## Create MA plot
```{r}
plotMA(smoc2_res)
```

Generate logical column 
```{r}
smoc2_res_all <- data.frame(smoc2_res) %>% mutate(threshold = padj < 0.05)
```


## Volcano plot
```{r message=FALSE, warning=FALSE}
ggplot(smoc2_res_all) + 
  geom_point(aes(x = log2FoldChange, y = -log10(padj), color = threshold)) + 
  xlab("log2 fold change") + 
  ylab("-log10 adjusted p-value") + 
  theme(legend.position = "none", 
        plot.title = element_text(size = rel(1.5), hjust = 0.5), 
        axis.title = element_text(size = rel(1.25)))
```
In the volcano plot we are looking at the amount of change in expression vs the p-value for the gene. the x-axis is the fold-change, so points to the right of zero are 1x, 2x more, and points to left -1x, -2x etc. the y-axis represents statistical significance, where up means higher significance (less likely due to chance) and down means less significance. 


## Heat Map
Subset normalized counts to significant genes
```{r}
sig_norm_counts_smoc2 <- smoc2_normalized_counts[rownames(smoc2_res_sig),]
```

Choose heatmap color palette
```{r}
heat_colors <- brewer.pal(n = 6, name = "YlOrRd")
```

Plot the Heatmap
```{r}
pheatmap(sig_norm_counts_smoc2, 
         color = heat_colors, 
         cluster_rows = T, 
         show_rownames = F,
         annotation = select(smoc2_metadata, condition), 
         scale = "row")
```
Here we see some good separation in terms of color and the pheotypes, indicating that the difference we see in gene expression is due to fibrosis, and not chance. if the situation were reversed, where there was no significant difference in gene expression, the plot would look more uniform in color, with no clear separation between the two groups.


### for generating a list of genes of interest
you can run this list against a database to get an idea as to what these genes do.
```{r}
smoc2_genes_sig <- smoc2_rawcounts[,1][(as.integer(rownames(smoc2_res_sig)))]

head(smoc2_genes_sig)
```
