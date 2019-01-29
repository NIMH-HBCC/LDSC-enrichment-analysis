---
title: "Single Cell Data Set - Zeisel 2018"
output: 
  html_document:
    toc: true
    toc_float: true
    keep_md: true
---

# Load Data

### Load necessary libraries


```r
library(tidyverse)
library("rhdf5")
library("snow")
```

### Load single cell dataset


```r
file="../../Data/Sten Whole Brain 10xGenomics 2018/Aggregate_20_july_2018/l5_all.agg.loom"
h5f <- H5Fopen(file)
exp <- as.data.frame(t(h5f$matrix))
exp$Gene <- h5f$row_attrs$Gene
```

Only keep genes with a unique name and tidy data


```r
exp <- exp %>% add_count(Gene) %>% 
  filter(n==1) %>%
  gather(key = column,value=Expr,-Gene) %>%
  as.tibble()
```


### Load gene coordinates

Load gene coordinates and extend upstream and downstream coordinates by 100kb.

File downloaded from MAGMA website (https://ctg.cncr.nl/software/magma).

Filtered to remove extended MHC (chr6, 25Mb to 34Mb).


```r
gene_coordinates <- 
  read_tsv("../../Data/NCBI/NCBI37.3.gene.loc.extendedMHCexcluded",col_names = FALSE,col_types = 'cciicc') %>%
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

### Add Cell type names


```r
cell_types <- cbind(column=as.character(paste0("V",1:265)),
                    Lvl1=h5f$col_attrs$TaxonomyRank1,
                    Lvl2=h5f$col_attrs$TaxonomyRank2,
                    Lvl3=h5f$col_attrs$TaxonomyRank3,
                    Lvl4=h5f$col_attrs$TaxonomyRank4,
                    Lvl5=h5f$col_attrs$ClusterName,
                    Description=h5f$col_attrs$Description,
                    NCells=h5f$col_attrs$NCells) %>%  
                    as.tibble() %>%
                    mutate(NCells=as.numeric(NCells))

exp_lvl5 <- inner_join(exp,cell_types,by="column") %>% as.tibble() %>% ungroup() %>% rename(Expr_sum_mean=Expr)
```

### Write dictonary for cell type names

Used to have nice labels in plotting


```r
dic_lvl5 <- select(cell_types,-column,-NCells)
write_tsv(dic_lvl5,"../../Data/Sten Whole Brain 10xGenomics 2018/dictionary_cell_type_names.txt")
```

### Get summary stats on the dataset


```r
sumstats_lvl5 <- exp_lvl5 %>% 
  group_by(Lvl5) %>%
  summarise(mean_UMI=sum(Expr_sum_mean),
            Ngenes=sum(Expr_sum_mean>0))

cell_types_arranged <- cell_types %>% 
  group_by(Lvl5) %>% 
  summarise(NCells=sum(NCells)) 

sumstats_lvl5 <- inner_join(sumstats_lvl5,cell_types_arranged) %>% mutate(total_UMI=mean_UMI*NCells)
```

# QC

### Remove low quality cell types

Cell types should have at least 200k reads.


```r
filter_lvl5 <- sumstats_lvl5 %>% filter(total_UMI>200000)
exp_lvl5 <- filter(exp_lvl5,Lvl5%in%filter_lvl5$Lvl5)
```

### Remove not expressed genes


```r
not_expressed <- exp_lvl5 %>% 
  group_by(Gene) %>% 
  summarise(total_sum=sum(Expr_sum_mean)) %>% 
  filter(total_sum==0) %>% 
  select(Gene) %>% unique() 

exp_lvl5 <- filter(exp_lvl5,!Gene%in%not_expressed$Gene)
```

### Scale data

Each cell type is scaled to the same total number of molecules. 


```r
exp_lvl5 <- exp_lvl5 %>% 
  group_by(Lvl5) %>% 
  mutate(Expr_sum_mean=Expr_sum_mean*1e6/sum(Expr_sum_mean))
```

# Specificity Calculation

The specifitiy is defined as the proportion of total expression performed by the cell type of interest (x/sum(x)).

We also record the maximum expression of the gene in all cell types. 

### Lvl5


```r
exp_lvl5 <- exp_lvl5 %>% 
  group_by(Gene) %>% 
  mutate(specificity=Expr_sum_mean/sum(Expr_sum_mean),
         max=max(Expr_sum_mean))
```

### Get 1to1 orthologs and Normalise

Join with 1to1 orthologs, standard normalized specificity within cell type.


```r
exp_lvl5 <- inner_join(exp_lvl5,m2h,by="Gene") %>% 
  group_by(Lvl5) %>%
    mutate(spe_norm=GenABEL::rntransform(specificity))
```

### Functions

#### Get MAGMA input continuous


```r
magma_input <- function(d,Cell_type,spec, outputfile_name){
 d_spe <- d %>% select(ENTREZ,Cell_type,spec) %>% spread(key=Cell_type,value=spec)
 colnames(d_spe) <- make.names(colnames(d_spe))
 dir.create("MAGMA_Zeisel", showWarnings = FALSE)
 write_tsv(d_spe,paste0("MAGMA_Zeisel/",Cell_type,"_",spec,"_",outputfile_name,".txt"))
}
```

#### Get LDSC input top 10%


```r
write_group  = function(df,Cell_type,outputfile_name) {
  df <- select(df,Lvl5,chr,start,end,ENTREZ)
  dir.create(paste0("LDSC_Zeisel/Bed_4LDSC2_",outputfile_name), showWarnings = FALSE,recursive = TRUE)
  write_tsv(df[-1],paste0("LDSC_Zeisel/Bed_4LDSC2_",outputfile_name,"/",make.names(unique(df[1])),".bed"),col_names = F)
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

Filter genes with expression below 1 TPM in the top cell type and take top 10% most specific genes for LDSC.


```r
exp_lvl5 %>% filter(max>1) %>% ldsc_bedfile("Lvl5","exp_gt_0.1")
```