Load Filtered Consensus Peaks, Promoter Regions, & Gene Body
============================================================

``` r
filtered_consensus_peaks_files <- list.files("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/00_consensus_peaks/results/filtered_consensus", 
                                             pattern = "*.bed",
                                             full.names = TRUE)

filtered_consensus_peaks <- lapply(filtered_consensus_peaks_files, rtracklayer::import)

names(filtered_consensus_peaks) <- gsub("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/00_consensus_peaks/results/filtered_consensus/|_filtered_consensus_peaks.bed", "", filtered_consensus_peaks_files)

# Import annotations
lncrna_mrna_promoters <- rtracklayer::import("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/00_consensus_peaks/results/promoters/lncrna_mrna_promoters.gtf")
lncrna_mrna_genebody <- rtracklayer::import("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/00_consensus_peaks/results/genebody/lncrna_mrna_genebody.gtf")

lncrna_gene_ids <- lncrna_mrna_genebody$gene_id[lncrna_mrna_genebody$gene_type == "lncRNA"]
mrna_gene_ids <- lncrna_mrna_genebody$gene_id[lncrna_mrna_genebody$gene_type == "protein_coding"]
```

Make Data Frame: DBP, Number of Genes Bound, and Peak Length
============================================================

``` r
num_peaks_df <- data.frame("dbp" = names(filtered_consensus_peaks),
                           "num_peaks" = sapply(filtered_consensus_peaks, length))


#get peak lengths for each dbp
num_peaks_df$total_peak_length <- sapply(filtered_consensus_peaks, function(x) sum(width(x)))
```

Generate Promoter Peak Counts & Overlapping Promoters
=====================================================

``` r
promoter_peak_counts <- count_peaks_per_feature(lncrna_mrna_promoters, filtered_consensus_peaks, type = "counts")

num_peaks_df$peaks_overlapping_promoters <- rowSums(promoter_peak_counts)

num_peaks_df$peaks_overlapping_lncrna_promoters <- rowSums(promoter_peak_counts[,lncrna_gene_ids])

num_peaks_df$peaks_overlapping_mrna_promoters <- rowSums(promoter_peak_counts[,mrna_gene_ids])
```

Generate Gene Body Peak Counts
==============================

``` r
genebody_peak_counts <- count_peaks_per_feature(lncrna_mrna_genebody, 
                                                filtered_consensus_peaks, 
                                                type = "counts")

num_peaks_df$peaks_overlapping_genebody <- rowSums(genebody_peak_counts)
num_peaks_df$peaks_overlapping_lncrna_genebody <- rowSums(genebody_peak_counts[,lncrna_gene_ids])
num_peaks_df$peaks_overlapping_mrna_genebody <- rowSums(genebody_peak_counts[,mrna_gene_ids])

#The counts produced here are not directly used in the pipeline, however, this data is useful for other analysis.
```

Merge with Human TF Database
============================

``` r
# The human TFs
# https://www.cell.com/cms/10.1016/j.cell.2018.01.029/attachment/ede37821-fd6f-41b7-9a0e-9d5410855ae6/mmc2.xlsx

suppressWarnings(human_tfs <- readxl::read_excel("/scratch/Shares/rinnclass/data/mmc2.xlsx",
                                sheet = 2, skip = 1))
```

    ## New names:
    ## * `` -> ...4

``` r
names(human_tfs)[4] <- "is_tf"

length(which(tolower(num_peaks_df$dbp) %in% tolower(human_tfs$Name)))
```

    ## [1] 438

``` r
human_tfs <- human_tfs[tolower(human_tfs$Name) %in% tolower(num_peaks_df$dbp), 1:4]
names(human_tfs) <- c("ensembl_id",
                      "dbp",
                      "dbd",
                      "tf")

num_peaks_df <- merge(num_peaks_df, human_tfs, all.x = T)

# Let's check how many NAs -- we should have some missing values.

write_csv(num_peaks_df, "/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/01_global_peak_properties/results/num_peaks_df.csv")
```

Generate Promoter Peak Occurence Data Frame
===========================================

``` r
promoter_peak_occurence <- count_peaks_per_feature(lncrna_mrna_promoters, filtered_consensus_peaks, 
                                               type = "occurrence")
# Output to promoter_peak_occurence_matrix
write.table(promoter_peak_occurence, "/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/01_global_peak_properties/results/lncrna_mrna_promoter_peak_occurence_matrix.tsv")


# First make sure promoter_peak_occurence and lncrna_mrna_promoters are in the same order
stopifnot(all(colnames(promoter_peak_occurence) == lncrna_mrna_promoters$gene_id))



# We are going to use the promoter peak occurence matrix above to keep adding onto in future files.

peak_occurence_df <- data.frame("gene_id" = colnames(promoter_peak_occurence),
                                "gene_name" = lncrna_mrna_promoters$gene_name,
                                "gene_type" = lncrna_mrna_promoters$gene_type,
                                #"gene_length" = 
                                "chr" = lncrna_mrna_promoters@seqnames,   
                                "3kb_up_tss_start" = lncrna_mrna_promoters@ranges@start,
                                "strand" = lncrna_mrna_promoters@strand,
                                "number_of_dbp" = colSums(promoter_peak_occurence))

write_csv(peak_occurence_df, "/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/01_global_peak_properties/results/peak_occurence_dataframe.csv")
```

Find unbound promoters
======================

``` r
unbound_promoters <- peak_occurence_df %>% filter(peak_occurence_df$number_of_dbp < 1)

# here is just a simple index and filter of the index to have at least 1 dbp bound.
#There are no unbound promoters.
```

Find DBPs with no overlaps with promoter
========================================

``` r
dbp_promoter_ovl <- get_overlapping_peaks(lncrna_mrna_promoters, filtered_consensus_peaks)

num_ovl <- sapply(dbp_promoter_ovl, length)
```

Plot peak\_occurence\_df
========================

``` r
g <- ggplot(peak_occurence_df, aes(x = number_of_dbp))

#AES layer will know to make a histogram since there is only one set of values for x.
g + geom_density(alpha = 0.2, color = "#424242", fill = "#424242") +
  xlab(expression("Number of DBPs")) +
  ylab(expression("Density")) +
  ggtitle("DBPs distribution",
          subtitle = "mRNA and lncRNA genes") 
```

![](01_global_peak_properties_files/figure-markdown_github/unnamed-chunk-9-1.png)

``` r
ggsave("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/01_global_peak_properties/figures/DBPs_distribution.png", height = 5, width = 8)
```

Find super promoter
===================

Here we are defining super promoters as promoters that bind more than 350 different DBPs.
-----------------------------------------------------------------------------------------

``` r
super_promoter <- promoter_peak_occurence[, colSums(promoter_peak_occurence) > 350]

super_promoter_dbps <- data.frame(dbp = rownames(super_promoter), num_super_promoter_overlaps = rowSums(super_promoter))

super_promoter_features <- merge(super_promoter_dbps, num_peaks_df)

super_promoter_features <- dplyr::select(super_promoter_features, dbp, num_super_promoter_overlaps, everything() )

super_promoter_features <- super_promoter_features %>%
  mutate(percent_super_prom_ov = num_super_promoter_overlaps/peaks_overlapping_promoters * 100, 
         percent_of_total_peaks_at_super_prom = num_super_promoter_overlaps/num_peaks * 100,
         zincfinger = ifelse(grepl(" ZF",dbd), "ZF", "Other"))

ggplot(super_promoter_features, aes(x = percent_super_prom_ov, color = zincfinger)) +
  geom_density()
```

![](01_global_peak_properties_files/figure-markdown_github/unnamed-chunk-10-1.png)

``` r
ggplot(super_promoter_features, aes(x = percent_of_total_peaks_at_super_prom, y = percent_super_prom_ov)) + 
  geom_point()
```

![](01_global_peak_properties_files/figure-markdown_github/unnamed-chunk-10-2.png)

``` r
write.csv(super_promoter_features, "/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/01_global_peak_properties/results/super_promoter_features.csv")
```