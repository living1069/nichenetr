Perform NicheNet analysis with prioritization
================
Robin Browaeys & Chananchida Sang-aram
2023-07-20

<!-- github markdown built using 
rmarkdown::render("vignettes/seurat_steps_prioritization.Rmd", output_format = "github_document")
-->

``` r
### Load Packages
library(nichenetr)
library(Seurat) # please update to Seurat V4
library(tidyverse)

### Read in Seurat object
seuratObj = readRDS(url("https://zenodo.org/record/3531889/files/seuratObj.rds"))
```

In this vignette, we will extend the basic NicheNet analysis analysis
from [Perform NicheNet analysis starting from a Seurat object:
step-by-step analysis](seurat_steps.md) by incorporating gene expression
as part of the prioritization This is a generalization of the
[Differential NicheNet](differential_nichenet.md) and
[MultiNicheNet](https://github.com/saeyslab/multinichenetr) approach.
While the original NicheNet only ranks ligands based on the ligand
activity analysis, it is now also possible to prioritize ligands based
on upregulation of the ligand/receptor, and the cell-type and condition
specificity of hte ligand and receptor.

Make sure you understand the different steps in a NicheNet analysis that
are described in that vignette before proceeding with this vignette and
performing a real NicheNet analysis on your data.

We will again make use of mouse NICHE-seq data from Medaglia et al. to
explore intercellular communication in the T cell area in the inguinal
lymph node before and 72 hours after lymphocytic choriomeningitis virus
(LCMV) infection (Medaglia et al. 2017). We will NicheNet to explore
immune cell crosstalk in response to this LCMV infection. In this
dataset, differential expression is observed between CD8 T cells in
steady-state and CD8 T cells after LCMV infection. NicheNet can be
applied to look at how several immune cell populations in the lymph node
(i.e., monocytes, dendritic cells, NK cells, B cells, CD4 T cells) can
regulate and induce these observed gene expression changes. NicheNet
will specifically prioritize ligands from these immune cells and their
target genes that change in expression upon LCMV infection.

Hence, we have to make some additional calculations, including DE of the
ligand/receptor in a sender/receiver cell type, and the average
expression of each ligand/receptor in each sender/receiver cell type.
The DE analysis boils down to computing pairwise tests between the cell
type of interest and other cell types in the dataset. We will subset the
data to only the condition of interest, “LCMV”. For this analysis we
will consider all cell types as both sender and receiver, as we want the
ligand/receptor to be specific.

The used [ligand-target matrix](https://doi.org/10.5281/zenodo.7074290)
and the [Seurat object of the processed NICHE-seq single-cell
data](https://doi.org/10.5281/zenodo.3531889) can be downloaded from
Zenodo.

## Load required packages, read in the Seurat object with processed expression data of interacting cells and NicheNet’s ligand-target prior model, ligand-receptor network and weighted integrated networks.

The NicheNet ligand-receptor network and weighted networks are necessary
to define and show possible ligand-receptor interactions between two
cell populations. The ligand-target matrix denotes the prior potential
that particular ligands might regulate the expression of particular
target genes. This matrix is necessary to prioritize possible
ligand-receptor interactions based on observed gene expression effects
(i.e. NicheNet’s ligand activity analysis) and infer affected target
genes of these prioritized ligands.

``` r
lr_network = readRDS(url("https://zenodo.org/record/7074291/files/lr_network_mouse_21122021.rds"))
ligand_target_matrix = readRDS(url("https://zenodo.org/record/7074291/files/ligand_target_matrix_nsga2r_final_mouse.rds"))
weighted_networks = readRDS(url("https://zenodo.org/record/7074291/files/weighted_networks_nsga2r_final_mouse.rds"))
lr_network = lr_network %>% distinct(from, to)
head(lr_network)
## # A tibble: 6 × 2
##   from          to   
##   <chr>         <chr>
## 1 2300002M23Rik Ddr1 
## 2 2610528A11Rik Gpr15
## 3 9530003J23Rik Itgal
## 4 a             Atrn 
## 5 a             F11r 
## 6 a             Mc1r
ligand_target_matrix[1:5,1:5] # target genes in rows, ligands in columns
##               2300002M23Rik 2610528A11Rik 9530003J23Rik            a          A2m
## 0610005C13Rik  0.000000e+00  0.000000e+00  1.311297e-05 0.000000e+00 1.390053e-05
## 0610009B22Rik  0.000000e+00  0.000000e+00  1.269301e-05 0.000000e+00 1.345536e-05
## 0610009L18Rik  8.872902e-05  4.977197e-05  2.581909e-04 7.570125e-05 9.802264e-05
## 0610010F05Rik  2.194046e-03  1.111556e-03  3.142374e-03 1.631658e-03 2.585820e-03
## 0610010K14Rik  2.271606e-03  9.360769e-04  3.546140e-03 1.697713e-03 2.632082e-03

weighted_networks_lr = weighted_networks$lr_sig %>% inner_join(lr_network, by = c("from","to"))
head(weighted_networks$lr_sig) # interactions and their weights in the ligand-receptor + signaling network
## # A tibble: 6 × 3
##   from          to     weight
##   <chr>         <chr>   <dbl>
## 1 0610010F05Rik App    0.110 
## 2 0610010F05Rik Cat    0.0673
## 3 0610010F05Rik H1f2   0.0660
## 4 0610010F05Rik Lrrc49 0.0829
## 5 0610010F05Rik Nicn1  0.0864
## 6 0610010F05Rik Srpk1  0.123
head(weighted_networks$gr) # interactions and their weights in the gene regulatory network
## # A tibble: 6 × 3
##   from          to            weight
##   <chr>         <chr>          <dbl>
## 1 0610010K14Rik 0610010K14Rik 0.121 
## 2 0610010K14Rik 2510039O18Rik 0.121 
## 3 0610010K14Rik 2610021A01Rik 0.0256
## 4 0610010K14Rik 9130401M01Rik 0.0263
## 5 0610010K14Rik Alg1          0.127 
## 6 0610010K14Rik Alox12        0.128

seuratObj = alias_to_symbol_seurat(seuratObj, "mouse")
```

# Perform the NicheNet analysis

In this case study, we want to apply NicheNet to predict which ligands
expressed by all immune cells in the T cell area of the lymph node are
most likely to have induced the differential expression in CD8 T cells
after LCMV infection.

As described in the main vignette, the pipeline of a basic NicheNet
analysis consist of the following steps:

In this case study, the receiver cell population is the ‘CD8 T’ cell
population, whereas the sender cell populations are ‘CD4 T’, ‘Treg’,
‘Mono’, ‘NK’, ‘B’ and ‘DC’. We will consider a gene to be expressed when
it is expressed in at least 10% of cells in one cluster.

``` r
# 1. Define a “sender/niche” cell population and a “receiver/target” cell population present in your expression data and determine which genes are expressed in both populations
## receiver
receiver = "CD8 T"
expressed_genes_receiver = get_expressed_genes(receiver, seuratObj, pct = 0.10)
background_expressed_genes = expressed_genes_receiver %>% .[. %in% rownames(ligand_target_matrix)]

## sender
sender_celltypes = c("CD4 T","Treg", "Mono", "NK", "B", "DC")

list_expressed_genes_sender = sender_celltypes %>% unique() %>% lapply(get_expressed_genes, seuratObj, 0.10) # lapply to get the expressed genes of every sender cell type separately here
expressed_genes_sender = list_expressed_genes_sender %>% unlist() %>% unique()

# 2. Define a gene set of interest: these are the genes in the “receiver/target” cell population that are potentially affected by ligands expressed by interacting cells (e.g. genes differentially expressed upon cell-cell interaction)

seurat_obj_receiver= subset(seuratObj, idents = receiver)
seurat_obj_receiver = SetIdent(seurat_obj_receiver, value = seurat_obj_receiver[["aggregate"]])

condition_oi = "LCMV"
condition_reference = "SS" 
  
DE_table_receiver = FindMarkers(object = seurat_obj_receiver, ident.1 = condition_oi, ident.2 = condition_reference, min.pct = 0.10) %>% rownames_to_column("gene")

geneset_oi = DE_table_receiver %>% filter(p_val_adj <= 0.05 & abs(avg_log2FC) >= 0.25) %>% pull(gene)
geneset_oi = geneset_oi %>% .[. %in% rownames(ligand_target_matrix)]

# 3. Define a set of potential ligands
ligands = lr_network %>% pull(from) %>% unique()
receptors = lr_network %>% pull(to) %>% unique()

expressed_ligands = intersect(ligands,expressed_genes_sender)
expressed_receptors = intersect(receptors,expressed_genes_receiver)

potential_ligands = lr_network %>% filter(from %in% expressed_ligands & to %in% expressed_receptors) %>% pull(from) %>% unique()

# 4. Perform NicheNet ligand activity analysis
ligand_activities = predict_ligand_activities(geneset = geneset_oi, background_expressed_genes = background_expressed_genes, ligand_target_matrix = ligand_target_matrix, potential_ligands = potential_ligands)

ligand_activities = ligand_activities %>% arrange(-aupr_corrected) %>% mutate(rank = rank(desc(aupr_corrected)))
ligand_activities
## # A tibble: 73 × 6
##    test_ligand auroc  aupr aupr_corrected pearson  rank
##    <chr>       <dbl> <dbl>          <dbl>   <dbl> <dbl>
##  1 Ebi3        0.663 0.390          0.244   0.301     1
##  2 Ptprc       0.642 0.310          0.165   0.167     2
##  3 H2-M3       0.608 0.292          0.146   0.179     3
##  4 H2-M2       0.611 0.279          0.133   0.153     5
##  5 H2-T10      0.611 0.279          0.133   0.153     5
##  6 H2-T22      0.611 0.279          0.133   0.153     5
##  7 H2-T23      0.611 0.278          0.132   0.153     7
##  8 H2-K1       0.605 0.268          0.122   0.142     8
##  9 H2-Q4       0.605 0.268          0.122   0.141    10
## 10 H2-Q6       0.605 0.268          0.122   0.141    10
## # … with 63 more rows
```

## Perform prioritization of ligand-receptor pairs

In addition to the NicheNet ligand activity (`activity_scaled`), you can
prioritize based on:

- Upregulation of the ligand in a sender cell type compared to other
  cell types: `de_ligand`
- Upregulation of the receptor in a receiver cell type: `de_receptor`
- Average expression of the ligand in the sender cell type:
  `exprs_ligand`
- Average expression of the receptor in the receiver cell type:
  `exprs_receptor`
- Condition-specificity of the ligand across all cell types:
  `ligand_condition_specificity`
- Condition-specificity of the receptor across all cell types:
  `receptor_condition_specificity`

Note that the first four criteria are calculated only in the condition
of interest.

``` r
# By default, ligand_condition_specificty and receptor_condition_specificty are 0
prioritizing_weights = c("de_ligand" = 1,
                          "de_receptor" = 1,
                          "activity_scaled" = 2,
                          "exprs_ligand" = 1,
                          "exprs_receptor" = 1,
                         "ligand_condition_specificity" = 0.5,
                         "receptor_condition_specificity" = 0.5)
```

We provide helper functions to calculate these values, including
`calculate_de` and `get_exprs_avg`. `process_table_to_ic` transforms
these different dataframes so they are compatible with the
`generate_prioritization_tables` function.

``` r
celltypes <- unique(seuratObj$celltype)
lr_network_renamed <- lr_network %>% rename(ligand=from, receptor=to)

# Only calculate DE for LCMV condition, with genes that are in the ligand-receptor network
DE_table <- calculate_de(seuratObj, celltype_colname = "celltype",
                         condition_colname = "aggregate", condition_oi = condition_oi,
                         features = union(expressed_ligands, expressed_receptors))

# Average expression information - only for LCMV condition
expression_info <- get_exprs_avg(seuratObj, "celltype", condition_colname = "aggregate", condition_oi = condition_oi)

# Calculate condition specificity - only for datasets with two conditions!
condition_markers <- FindMarkers(object = seuratObj, ident.1 = condition_oi, ident.2 = condition_reference,
                                 group.by = "aggregate", min.pct = 0, logfc.threshold = 0,
                                 features = union(expressed_ligands, expressed_receptors)) %>% rownames_to_column("gene")

# Combine DE of senders and receivers -> used for prioritization
processed_DE_table <- process_table_to_ic(DE_table, table_type = "celltype_DE", lr_network_renamed,
                                         senders_oi = sender_celltypes, receivers_oi = receiver)
  
processed_expr_table <- process_table_to_ic(expression_info, table_type = "expression", lr_network_renamed)

processed_condition_markers <- process_table_to_ic(condition_markers, table_type = "group_DE", lr_network_renamed)
```

Finally we generate the prioritization table. The `lfc_ligand` and
`lfc_receptor` columns are based on the differences between cell types
within your condition of interest. This is equivalent to subsetting your
Seurat object to only the condition of interest and running
`Seurat::FindAllMarkers`.

The columns that refer to differential expression between conditions are
those with the \_group suffix, e.g., `lfc_ligand_group` and
`lfc_receptor_group.` These are celltype agnostic: they are calculated
by using `Seurat::FindMarkers` between two conditions across all cell
types.

``` r
prior_table <- generate_prioritization_tables(processed_expr_table,
                               processed_DE_table,
                               ligand_activities,
                               processed_condition_markers,
                               prioritizing_weights = prioritizing_weights)

prior_table 
## # A tibble: 858 × 51
##    sender receiver ligand receptor lfc_ligand lfc_receptor ligand_receptor_lfc_avg p_val_ligand p_adj_ligand p_val_receptor p_adj_receptor pct_expressed_send… pct_expressed_r… avg_ligand avg_receptor ligand_receptor…
##    <chr>  <chr>    <chr>  <chr>         <dbl>        <dbl>                   <dbl>        <dbl>        <dbl>          <dbl>          <dbl>               <dbl>            <dbl>      <dbl>        <dbl>            <dbl>
##  1 NK     CD8 T    Ptprc  Dpp4          0.527       0.194                    0.361     1.14e- 7     1.54e- 3      1.84e-  4      1   e+  0               0.832            0.133     16.6           1.35           22.5  
##  2 Mono   CD8 T    Ptprc  Dpp4          0.473       0.194                    0.334     6.18e- 6     8.37e- 2      1.84e-  4      1   e+  0               0.844            0.133     14.9           1.35           20.1  
##  3 Mono   CD8 T    Ebi3   Il27ra        0.503       0.0580                   0.281     1.14e-52     1.54e-48      7.60e-  4      1   e+  0               0.122            0.139      0.546         1.13            0.619
##  4 Mono   CD8 T    Cxcl10 Dpp4          4.23        0.194                    2.21      1.34e-80     1.82e-76      1.84e-  4      1   e+  0               0.744            0.133     54.8           1.35           74.1  
##  5 DC     CD8 T    H2-M2  Cd8a          3.59        1.88                     2.73      0            0             1.97e-266      2.67e-262               0.556            0.669      9.73         10.7           104.   
##  6 B      CD8 T    H2-M3  Cd8a          0.372       1.88                     1.13      1.49e- 3     1   e+ 0      1.97e-266      2.67e-262               0.152            0.669      1.59         10.7            17.0  
##  7 Treg   CD8 T    Ptprc  Dpp4          0.241       0.194                    0.218     3.39e- 2     1   e+ 0      1.84e-  4      1   e+  0               0.643            0.133     13.2           1.35           17.9  
##  8 DC     CD8 T    H2-D1  Cd8a          1.21        1.88                     1.54      3.65e- 8     4.94e- 4      1.97e-266      2.67e-262               1                0.669     60.7          10.7           651.   
##  9 B      CD8 T    Ptprc  Dpp4          0.222       0.194                    0.208     5.93e- 3     1   e+ 0      1.84e-  4      1   e+  0               0.647            0.133     12.3           1.35           16.6  
## 10 Mono   CD8 T    H2-T22 Cd8a          0.373       1.88                     1.13      7.81e- 4     1   e+ 0      1.97e-266      2.67e-262               0.7              0.669     10.4          10.7           111.   
## # … with 848 more rows, and 35 more variables: lfc_pval_ligand <dbl>, p_val_ligand_adapted <dbl>, scaled_lfc_ligand <dbl>, scaled_p_val_ligand <dbl>, scaled_lfc_pval_ligand <dbl>, scaled_p_val_ligand_adapted <dbl>,
## #   activity <dbl>, rank <dbl>, activity_zscore <dbl>, scaled_activity <dbl>, lfc_pval_receptor <dbl>, p_val_receptor_adapted <dbl>, scaled_lfc_receptor <dbl>, scaled_p_val_receptor <dbl>,
## #   scaled_lfc_pval_receptor <dbl>, scaled_p_val_receptor_adapted <dbl>, scaled_avg_exprs_ligand <dbl>, scaled_avg_exprs_receptor <dbl>, lfc_ligand_group <dbl>, p_val_ligand_group <dbl>, lfc_pval_ligand_group <dbl>,
## #   p_val_ligand_adapted_group <dbl>, scaled_lfc_ligand_group <dbl>, scaled_p_val_ligand_group <dbl>, scaled_lfc_pval_ligand_group <dbl>, scaled_p_val_ligand_adapted_group <dbl>, lfc_receptor_group <dbl>,
## #   p_val_receptor_group <dbl>, lfc_pval_receptor_group <dbl>, p_val_receptor_adapted_group <dbl>, scaled_lfc_receptor_group <dbl>, scaled_p_val_receptor_group <dbl>, scaled_lfc_pval_receptor_group <dbl>,
## #   scaled_p_val_receptor_adapted_group <dbl>, prioritization_score <dbl>
```

As you can see, the resulting table now show the rankings for
*ligand-receptor interactions of a sender-receiver cell type pair*,
instead of just the prioritized ligands. We included all columns here,
but if you just want relevant columns that were used to calculate the
ranking:

``` r
prior_table %>% select(c('sender', 'receiver', 'ligand', 'receptor', 'scaled_lfc_ligand', 'scaled_lfc_receptor', 'scaled_p_val_ligand_adapted', 'scaled_p_val_receptor_adapted', 'scaled_avg_exprs_ligand', 'scaled_avg_exprs_receptor', 'scaled_lfc_ligand_group', 'scaled_lfc_receptor_group', 'scaled_activity'))
## # A tibble: 858 × 13
##    sender receiver ligand receptor scaled_lfc_ligand scaled_lfc_receptor scaled_p_val_ligand_adapted scaled_p_val_receptor_adapted scaled_avg_exprs_… scaled_avg_expr… scaled_lfc_liga… scaled_lfc_rece… scaled_activity
##    <chr>  <chr>    <chr>  <chr>                <dbl>               <dbl>                       <dbl>                         <dbl>              <dbl>            <dbl>            <dbl>            <dbl>           <dbl>
##  1 NK     CD8 T    Ptprc  Dpp4                 0.824               0.901                       0.843                         0.887              1.00             1.00             0.779           0.831            0.862
##  2 Mono   CD8 T    Ptprc  Dpp4                 0.808               0.901                       0.827                         0.887              0.867            1.00             0.779           0.831            0.862
##  3 Mono   CD8 T    Ebi3   Il27ra               0.817               0.831                       0.941                         0.859              1.00             0.859            0.538           0.0986           1.00 
##  4 Mono   CD8 T    Cxcl10 Dpp4                 0.997               0.901                       0.957                         0.887              1.00             1.00             0.990           0.831            0.431
##  5 DC     CD8 T    H2-M2  Cd8a                 0.989               1                           1                             1                  1.00             1.00             0.308           0.0845           0.664
##  6 B      CD8 T    H2-M3  Cd8a                 0.777               1                           0.768                         1                  1.00             1.00             0.846           0.0845           0.748
##  7 Treg   CD8 T    Ptprc  Dpp4                 0.726               0.901                       0.707                         0.887              0.741            1.00             0.779           0.831            0.862
##  8 DC     CD8 T    H2-D1  Cd8a                 0.925               1                           0.853                         1                  1.00             1.00             0.885           0.0845           0.593
##  9 B      CD8 T    Ptprc  Dpp4                 0.713               0.901                       0.745                         0.887              0.666            1.00             0.779           0.831            0.862
## 10 Mono   CD8 T    H2-T22 Cd8a                 0.779               1                           0.778                         1                  1.00             1.00             0.981           0.0845           0.664
## # … with 848 more rows
```

Cxcl10 now went up in the rankings due to both the high expression of
its potential receptor Dpp4 and its high celltype specificity
(`scaled_lfc_ligand`). You can also see this in the dotplot and heatmap
below.

``` r
best_upstream_ligands = ligand_activities %>% top_n(20, aupr_corrected) %>% arrange(desc(aupr_corrected)) %>% pull(test_ligand) %>% unique()

# DE analysis for each sender cell type
DE_table_all = Idents(seuratObj) %>% levels() %>% intersect(sender_celltypes) %>%
  lapply(get_lfc_celltype, seurat_obj = seuratObj, condition_colname = "aggregate", condition_oi = condition_oi, condition_reference = condition_reference,
         expression_pct = 0.10, celltype_col = NULL) %>% reduce(full_join) 
DE_table_all[is.na(DE_table_all)] = 0

order_ligands <- make.names(best_upstream_ligands) %>% rev()

# ligand activity heatmap
ligand_aupr_matrix <- ligand_activities %>% select(aupr_corrected) %>% as.matrix() %>% magrittr::set_rownames(ligand_activities$test_ligand) %>%
   `rownames<-`(make.names(rownames(.))) %>% `colnames<-`(make.names(colnames(.)))

vis_ligand_aupr <- as.matrix(ligand_aupr_matrix[order_ligands, ], ncol=1) %>% magrittr::set_colnames("AUPR")
p_ligand_aupr <- make_heatmap_ggplot(vis_ligand_aupr, "Prioritized ligands","Ligand activity",
                                        color = "darkorange",legend_position = "top", x_axis_position = "top",
                                        legend_title = "AUPR\ntarget gene prediction ability)") +
  theme(legend.text = element_text(size = 9))
  
  
# LFC heatmap
# First combine ligand activities with DE information and make 
ligand_activities_de <- ligand_activities %>% select(test_ligand, aupr_corrected) %>% rename(ligand = test_ligand) %>% left_join(DE_table_all %>% rename(ligand = gene))
ligand_activities_de[is.na(ligand_activities_de)] <- 0
lfc_matrix <- ligand_activities_de  %>% select(-ligand, -aupr_corrected) %>% as.matrix() %>% magrittr::set_rownames(ligand_activities_de$ligand) %>%
  `rownames<-`(make.names(rownames(.))) %>% `colnames<-`(make.names(colnames(.)))
vis_ligand_lfc <- lfc_matrix[order_ligands,]

p_ligand_lfc <- make_threecolor_heatmap_ggplot(vis_ligand_lfc, "Prioritized ligands","LFC in Sender",
                                               low_color = "midnightblue", mid_color = "white", mid = median(vis_ligand_lfc), high_color = "red",
                                               legend_position = "top", x_axis_position = "top", legend_title = "LFC") +
  theme(axis.text.y = element_text(face = "italic"))


# ligand expression Seurat dotplot
order_ligands_adapted <- str_replace_all(order_ligands, "\\.", "-")
rotated_dotplot <- DotPlot(seuratObj %>% subset(celltype %in% sender_celltypes), features = order_ligands_adapted, cols = "RdYlBu") +
  # flip of coordinates necessary because we want to show ligands in the rows when combining all plots
  coord_flip() + theme(legend.text = element_text(size = 10), legend.title = element_text(size = 12))
  
# Combine figures and legend separately
figures_without_legend <- cowplot::plot_grid(
  p_ligand_aupr + theme(legend.position = "none", axis.ticks = element_blank()) + theme(axis.title.x = element_text()),
  rotated_dotplot + theme(legend.position = "none", axis.ticks = element_blank(), axis.title.x = element_text(size = 12),
                          axis.text.y = element_text(face = "italic", size = 9), axis.text.x = element_text(size = 9,  angle = 90,hjust = 0)) +
    ylab("Expression in Sender") + xlab("") + scale_y_discrete(position = "right"),
  p_ligand_lfc + theme(legend.position = "none", axis.ticks = element_blank()) + theme(axis.title.x = element_text()) + ylab(""),
  align = "hv",
  nrow = 1,
  rel_widths = c(ncol(vis_ligand_aupr)+6, ncol(vis_ligand_lfc) + 7, ncol(vis_ligand_lfc) + 8))

legends <- cowplot::plot_grid(
    ggpubr::as_ggplot(ggpubr::get_legend(p_ligand_aupr)),
    ggpubr::as_ggplot(ggpubr::get_legend(rotated_dotplot)),
    ggpubr::as_ggplot(ggpubr::get_legend(p_ligand_lfc)),
    nrow = 1,
    align = "h", rel_widths = c(1.5, 1, 1))

combined_plot <- cowplot::plot_grid(figures_without_legend, legends, nrow = 2, align = "hv")
print(combined_plot)
```

![](seurat_steps_prioritization_files/figure-gfm/unnamed-chunk-9-1.png)<!-- -->

### Extra visualization of ligand-receptor pairs

We provide the function `make_mushroom_plot` which allows you to display
expression of ligand-receptor pairs in semicircles. By default, the fill
gradient shows the LFC between cell types, while the size of the
semicircle corresponds to the scaled mean expression.

``` r
make_mushroom_plot(prior_table, top_n = 30)
```

![](seurat_steps_prioritization_files/figure-gfm/unnamed-chunk-10-1.png)<!-- -->

We provide multiple ways to customize this plot, including changing the
“size” and “fill” values to certain columns from the prioritization
table (but without the `_ligand` or `_receptor` suffix). In addition,
you can also choose to show the rankings of each ligand-receptor-sender
pair, as well as show all data points for context.

``` r
print(paste0("Column names that you can use are: ", paste0(prior_table %>% select(ends_with(c("_ligand", "_receptor", "_sender", "_receiver"))) %>% colnames() %>%
  str_remove("_ligand|_receptor|_sender|_receiver") %>% unique, collapse = ", ")))
## [1] "Column names that you can use are: lfc, p_val, p_adj, avg, lfc_pval, scaled_lfc, scaled_p_val, scaled_lfc_pval, scaled_avg_exprs, pct_expressed"

# Change size and color columns
make_mushroom_plot(prior_table, top_n = 30, size = "pct_expressed", color = "scaled_avg_exprs")
```

![](seurat_steps_prioritization_files/figure-gfm/unnamed-chunk-11-1.png)<!-- -->

``` r

# Show rankings and other datapoints
make_mushroom_plot(prior_table, top_n = 30, show_rankings = TRUE, show_all_datapoints = TRUE)
```

![](seurat_steps_prioritization_files/figure-gfm/unnamed-chunk-11-2.png)<!-- -->

``` r

# Show true limits instead of having it from 0 to 1
make_mushroom_plot(prior_table, top_n = 30, true_color_range = TRUE)
```

![](seurat_steps_prioritization_files/figure-gfm/unnamed-chunk-11-3.png)<!-- -->

### References

<div id="refs" class="references csl-bib-body hanging-indent">

<div id="ref-medaglia_spatial_2017" class="csl-entry">

Medaglia, Chiara, Amir Giladi, Liat Stoler-Barak, Marco De Giovanni,
Tomer Meir Salame, Adi Biram, Eyal David, et al. 2017. “Spatial
Reconstruction of Immune Niches by Combining Photoactivatable Reporters
and <span class="nocase">scRNA</span>-Seq.” *Science*, December,
eaao4277. <https://doi.org/10.1126/science.aao4277>.

</div>

</div>
