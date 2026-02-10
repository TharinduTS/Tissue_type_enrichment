# Tissue_type_enrichment
This is a slightly changed version of celltype_enrichment_V2_2 , that tries to find genes that show tissue specific expression patterns regardless of cell types and then look at variance in different cell types

I will be using the same filtered data file I used for Cell type Enrichment here (combined_expression_data_filtered.tsv from cell type enrichment chapter 2)

*up to this step, the pipeline should follow cell type enrichment version 2.2 pipeline chapter 1 and 2 including 'Pre-preparation'. Then tissue type enrichment chapter 1 begins with the resulting filtered data file*

# 1) Tissue type enrichment

## 1-IIntroduction

I am running the same cell type enrichment, but changing the command to use tissue instead of cell type to calculate enrichment values. Therefore everything that says cell types in the script means Tissue types

## 1-II Scripts

This needs celltype_enrichment_v1_4.py from

```url
https://github.com/TharinduTS/cell_type_enrichment_v2?tab=readme-ov-file#3-ii-celltype-enrichment-script
```

and 

run_celltype_enrichment_v1_4.sh from

```url
https://github.com/TharinduTS/cell_type_enrichment_v2?tab=readme-ov-file#3-iii-celltype-enrichment-runner
```

## 1-III Run command

```bash
./run_celltype_enrichment_v1_4.sh --input-file combined_expression_data_filtered.tsv --output-file enrichment_values_for_tissue_types.tsv --min-clusters 1 --min-count 2 --specificity-mode penalize --min-specificity 1 --cell-type-col "Tissue"  --batch-col "Tissue"
```
# 2) Visualize enrichment distribution

## 2-I Introduction

This visualizes enrichment distribution for different tissue types following cell type enrichment chapter 4

## 2-II Script

This needs plot_distribution.py from 

```url
https://github.com/TharinduTS/cell_type_enrichment_v2?tab=readme-ov-file#4-ii-script
```

## 2-III Run command

```bash
python plot_distribution.py -i enrichment_values_for_tissue_types.tsv -c log2_enrichment_penalized --near-zero 10
 -o log2_enrichment_penalized_distribution.png
```

# 3) rank genes by number of tissue types present

## 3-I Introduction

I am going to rank genes taking the number of tissue types they are expressed in following cell typ[e enrichment chapter 6

## 3-II Scripts

This needs rank_genes.py from

```url
https://github.com/TharinduTS/cell_type_enrichment_v2?tab=readme-ov-file#6-ii-rank-genes-and-estimate-celltype-counts-script
```

## 3-III Run commands

Just like in cell type enrichment, I am going to drop any values with less than 0.5 for log 2 enrichment penalized value so it has to at least have 1.5 times the background noise to be considered in the calculation

```awk
awk -F'\t' 'NR==1 {print; next} { if ($11+0 >= 0.5) print }' enrichment_values_for_tissue_types.tsv > enrichment_values_for_tissue_types_filtered.tsv
```

Then I ranked genes by the number of tissue types they are expressed in

```bash
python rank_genes.py \
  --input enrichment_values_for_tissue_types_filtered.tsv \
  --output ranked_genes_unique_tissuetypes.tsv \
  --presence-col "Tissue" \
  --unique-col "Tissue" \
  --unique \
  --drop-na \
  --drop-negatives \
  --top-percent 100 \
  --top-col log2_enrichment_penalized \
  --sorting-col log2_enrichment_penalized \
  --gene-col Gene \
  --output-count-col top_percent_Tissue_type_count \
  --output-list-col top_percent_Tissue_types \
  --output-rank-within-col rank_within_Tissue_type \
  --output-overall-rank-col overall_rank_by_Tissue_type \
  --verbose
```

# 4) Cluster data integration

## 4-I Introduction

I am adding a filter for 1 or less and 2 or higher clusters just like in cell type enrichment chapter 7 here

## 4-II Script

This needs categorize_column.py from

```url
https://github.com/TharinduTS/cell_type_enrichment_v2?tab=readme-ov-file#7_ii-cluster-categories-script
```

## 4-III Run command

```
ython categorize_column.py ranked_genes_unique_tissuetypes.tsv ranked_genes_with_cluster_categories.tsv clusters_used 2 cluster_limit
```




