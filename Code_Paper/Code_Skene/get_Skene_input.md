---
title: "Single Cell Data Set - Skene 2018"
output: 
  html_document:
    toc: true
    toc_float: true
    keep_md: true
---



# Load Data

### Load single cell dataset


```r
library(tidyverse)
f <- "../../Data/KI_lvl1/celltype_data_allKImouse_wtHypo_MergedStriatal_1to1only_level1_thresh0_trim0.rda"
load(f)
exp <- as.data.frame(celltype_data[[1]]$all_sct) %>% rownames_to_column("Gene")
```

Only keep genes with a unique name


```r
exp_lvl5 <- exp %>% add_count(Gene) %>% 
  filter(n==1) %>%
  select(-n) %>%
  gather(key = Lvl5,value=Expr_sum_mean,-Gene) %>% as.tibble()
  as.tibble()
```

```
## # A tibble: 0 x 0
```

### Load gene coordinates

Load gene coordinates and extend upstream and downstream coordinates by 100kb.

File downloaded from MAGMA website (https://ctg.cncr.nl/software/magma).

Filtered to remove extended MHC (chr6, 25Mb to 34Mb).


```r
gene_coordinates <- 
  read_tsv("../../Data/NCBI/NCBI37.3.gene.loc.extendedMHCexcluded",
           col_names = FALSE,col_types = 'cciicc') %>%
  mutate(start=ifelse(X3-100000<0,0,X3-100000),end=X4+100000) %>%
  select(X2,start,end,1) %>% 
  rename(chr="X2", ENTREZ="X1") %>% 
  mutate(chr=paste0("chr",chr))
```

### Load mouse to human 1to1 orthologs

File downloaded from: http://www.informatics.jax.org/homology.shtml and parsed.


```r
m2h <- read_tsv("../../Data/m2h.txt",col_types = "iccccc") %>% 
  select(musName,entrez) %>%
  rename(Gene=musName) %>% rename(ENTREZ=entrez)
```

# Transform Data

### Write dictonary for cell type names


```r
dic_lvl5 <- select(exp_lvl5,Lvl5) %>% unique() %>% mutate(makenames=make.names(Lvl5))
write_tsv(dic_lvl5,"../../Data/KI_lvl1/dictionary_cell_type_names.txt")
```

# QC

### Scale to 10k molecules

Each cell type is scaled to the same total number of molecules. 


```r
exp_lvl5 <- exp_lvl5 %>% group_by(Lvl5) %>% mutate(Expr_sum_mean=Expr_sum_mean*1e6/sum(Expr_sum_mean))
```

# Specificity Calculation

The specifitiy is defined as the proportion of total expression performed by the cell type of interest (x/sum(x)).

### Lvl5


```r
exp_lvl5 <- exp_lvl5 %>% group_by(Gene) %>% 
  mutate(specificity=Expr_sum_mean/sum(Expr_sum_mean),
         max=max(Expr_sum_mean))
```

### Get 1to1 orthologs and Normalise

Get 1to1 orthologs, standard normalized specificity within cell type.


```r
exp_lvl5 <- inner_join(exp_lvl5,m2h,by="Gene") %>% group_by(Lvl5) %>%
    mutate(spe_norm=GenABEL::rntransform(specificity))
```

### Functions

#### Get MAGMA input continuous


```r
magma_input <- function(d,Cell_type,spec, outputfile_name){
 d_spe <- d %>% select(ENTREZ,Cell_type,spec) %>% spread(key=Cell_type,value=spec)
 colnames(d_spe) <- make.names(colnames(d_spe))
 dir.create("MAGMA_Skene", showWarnings = FALSE)
 write_tsv(d_spe,paste0("MAGMA_Skene/",Cell_type,"_",spec,"_",outputfile_name,".txt"))
}
```

#### Get LDSC input top 10%


```r
write_group  = function(df,Cell_type,outputfile_name) {
  df <- select(df,Lvl5,chr,start,end,ENTREZ)
  dir.create(paste0("LDSC_Skene/Bed_4LDSC2_",outputfile_name), showWarnings = FALSE,recursive = TRUE)
  write_tsv(df[-1],paste0("LDSC_Skene/Bed_4LDSC2_",outputfile_name,"/",make.names(unique(df[1])),".bed"),col_names = F)
return(df)
}
```


```r
ldsc_bedfile <- function(d,Cell_type, outputfile_name){
  d_spe <- d %>% inner_join(gene_coordinates,by="ENTREZ") %>% group_by_(Cell_type) %>% filter(specificity>=quantile(specificity,0.9)) 
  d_spe %>% do(write_group(.,Cell_type,outputfile_name))
}
```

### Write MAGMA/LDSC input files 

Keep all genes for continuous analysis.


```r
magma_input(exp_lvl5,"Lvl5","spe_norm","no_filter")
```

Filter genes with expression below 1 in the top cell type (scaled to 1M molecules) and take top 10% most specific genes.


```r
exp_lvl5 %>% filter(max>1) %>% ldsc_bedfile("Lvl5","exp_gt_0.1")
```