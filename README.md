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

# 5) Protein data integration

## 5-I Introduction

Just like in cell type enrichment chapter 8, I am going to add protein class data here per gene basis.

*Basically I am adding what HPA says about protein class of each gene here*

I am using genes_with_protein_type_cols_renamed.tsv file generated in cell type enrichment chapter 8-I and 8-II

## 5-II Script

This needs merge_tsv_by_keys.py from

```url
https://github.com/TharinduTS/cell_type_enrichment_v2?tab=readme-ov-file#1-ii-merge-script
```

and 

## 5-III Run command

ran it like

```bash
python merge_tsv_by_keys.py \
  --left ranked_genes_with_cluster_categories.tsv \
  --right genes_with_protein_type_cols_renamed.tsv \
  --left-keys "Gene","Gene name" \
  --right-keys "Gene","Gene name" \
  --right-cols "Protein_Class" \
  --out  ranked_genes_with_group_cluster_and_protein_class_data.tsv
```

And then,

### Replaced empty values 

### script

This needs replace_empty_values.py from

```url
https://github.com/TharinduTS/cell_type_enrichment_v2?tab=readme-ov-file#replaced-empty-values
```

### Run command

```bash
python replace_empty_values.py \
    -i ranked_genes_with_group_cluster_and_protein_class_data.tsv \
    -o ranked_genes_cleaned.tsv \
    -c Protein_Class \
    -r No_predicted_protein_class \
    -d $'\t'
```

## 5-IV Merge Cell type data

*Then I added Cell type data for these filtered rows from the earlier filtered data file "combined_expression_data_filtered.tsv" from chapter 1*

This needs match_and_collect.py from 

```url
https://github.com/TharinduTS/cell_type_enrichment_v2?tab=readme-ov-file#8-iii-merge-tissue-type-data
```

### Run command

```bash
python match_and_collect.py \
  ranked_genes_cleaned.tsv combined_expression_data_filtered.tsv \
  --file1-keys "Gene name" "Tissue" \
  --file2-keys "Gene name" "Tissue" \
  --collect-column "Cell type" \
  --filter-col nCPM --filter-op ">" --filter-value 0 \
  --output-column "Present Cell types" \
  --sep-out " & " \
  --drop-dupes \
  --output ranked_genes_cleaned_with_cell_type_data.tsv
```

# 6) Adding Cell type expression profile

## 6-I Introduction

Here I am doing the opposite of what I did in cell type enrichment chapter 11. Adding cell type specific expression for different cell types present in different tissue types in a way that is easier to visually represent

As I have already calculated enrichment scores considering both tissue and cell type in cell type enrichment chapter 11, I am directly using 'combined_expression_data_split.tsv' from there for this integration.

And 

### Merged cluster info into Cell type data

```bash
awk -F'\t' '
BEGIN { OFS="\t" }
NR==1 { print; next }
{
    $4 = $4 " with " $5
    print
}
' combined_expression_data_split.tsv > combined_expression_data_split_with_clusters.tsv
```
## 6-II Merge data

Then I added this data to my main dataframe. To avoid further complicating the way this data is shown, I merged this data matching columns "Gene Gene name Tissue" and looking for the Cell type inside the 'Present Cell typess' column with 

add_enrichment_to_tissues.py from 

```url
https://github.com/TharinduTS/cell_type_enrichment_v2?tab=readme-ov-file#11-ii-merge-data
```

### Run command

### Duplicate Cell type data

Because the next step is going to add extra data to the Cell type column, I am keeping a copy of that column to be used in 'color by Cell type' later in plotting

```
awk -F'\t' 'BEGIN{OFS="\t"} NR==1{print $0, "Cell_types_present"; next} {print $0, $19}' ranked_genes_cleaned_with_cell_type_data.tsv > ranked_genes_cleaned_with_cell_type_data_duplicated.tsv
```

This command could be little confusing because I used cell types for the initial script writing and now I am doing it for Tissues

*Note last 3 arguments in the following command*

### Run command

```bash
python3 add_enrichment_to_tissues.py \
  --table1 combined_expression_data_split_with_clusters.tsv \
  --table2 ranked_genes_cleaned_with_cell_type_data_duplicated.tsv \
  --output Final_data_with_cell_type_expression_data.tsv \
  --missing-value "Not_high_enough" \
  --value-sep ":" \
  --tissue-sep "&" \
  --tissue1-trim-after " with " \
  --label-source table1 \
  --case-insensitive \
  --cell-type-col Tissue \
  --tissue-col-table1 "Cell type" \
  --tissue-col-table2 "Present Cell types" 
```

## 6-III Select data to plot

Just like I did in chapter Cell type enrichment chapter 11, here I am selecting how many top data rows to plot, to avoid procssing complications

first 10k data points for emailing

```
head -n 10000 Final_data_with_cell_type_expression_data.tsv> top_10k_from_final_data.tsv
```

# 7) Making interactive plots

## 7-I Introduction

I am making interactive plots for tissue type expression here similar to cell type expression chapter 11-IV here.

## 7-II Scripts

For this we need universal_plot_maker_plus.py from

```
https://github.com/TharinduTS/universal_plot_maker_plus_with_subplot/blob/main/README.md
```

## 7-III Run command

I am using all the data here for the plot as it is not as big as cell type enrichment

```bash
python universal_plot_maker_plus.py \
  --file Final_data_with_cell_type_expression_data.tsv \
  --out Tissue_type_Enrichment.html \
  --plot-type bar \
  --x-choices "Gene name | Gene" \
  --y-choices "Enrichment score|log2_enrichment| specificity_tau | Enrichment score (tau penalized)|log2_enrichment_penalized" \
  --default-x "Gene name" \
  --default-y "log2_enrichment_penalized" \
  --color-col "Tissue" \
  --color-choices "Cell_types_present|Tissue" \
  --filter-cols "Tissue|cluster_limit|Protein_Class" \
  --search-cols "Gene|Gene name|Present Cell types" \
  --details "Gene|Gene name|Tissue|clusters_used|Enrichment score|log2_enrichment| specificity_tau |log2_enrichment_penalized|top_percent_Tissue_type_count|top_percent_Tissue_types|overall_rank_by_Tissue_type|rank_within_Tissue_type|Protein_Class|Cell_types_present" \
  --title "Tissue type Enrichmnt V 2.2" \
  --dup-policy overlay \
  --sort-primary "overall_rank_by_Tissue_type" \
  --sort-primary-order asc \
  --sort-secondary "log2_enrichment_penalized" \
  --sort-secondary-order desc \
  --initial-zoom 100 \
  --self-contained \
  --lang en \
  --pt-enable \
  --pt-col "Present Cell types" \
  --pt-title "Enrichment per present Cell type" \
  --pt-x-label "Cell type" \
  --pt-y-label "log2 Enrichment Penalized" \
  --pt-color "#2a9d8f" \
  --pt-height 360 \
  --pt-width auto \
  --pt-rotate -35 \
  --pt-container-id "present-tissues-plot" \
  --pt-enable --pt-mode flow \
  --pt-anchor "#rowDetails" --pt-position after \
  --pt-offset-x -300 --pt-offset-y -10
```


