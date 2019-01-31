---
title: "Run MAGMA"
output: 
  html_document:
    keep_md: true
---

We can now run MAGMA on the processed intelligence GWAS using the following commands:


```r
cd MAGMA_Zeisel/
int="../../../Data/GWAS/Intelligence/sumstats/int.annotated_35kbup_10_down.genes.raw"
RANDOM_GENE_SET="../../../Data/random_gene_sets/synaptic_ARC_Seth_Grant.t.txt"
cell_type="Lvl5_spe_norm_no_filter.txt"
magma --gene-results  $int --set-annot  $RANDOM_GENE_SET --gene-covar $cell_type filter onesided --out int
cd ..
```