Load data frames: promoter, genebody, and genebodymMinusPromoter peak occurance
===============================================================================

``` r
#1)
promoter_peak_occurence_df <- read.csv(file = '/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/01_global_peak_properties/results/peak_occurence_dataframe.csv')

#2)
genebody_peak_occurence_df <- read.csv(file =
"/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/01_global_peak_properties/results/genebody_peak_occurence_dataframe.csv")

#3)
genebodyMinusPromoter_df <- read.csv(file =
"/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/01_global_peak_properties/results/GgenebodyMinusPromoter_df.csv")
```

compare peak\_occurence\_df
===========================

``` r
# promoter
g <- ggplot(promoter_peak_occurence_df, aes(x = number_of_dbp))
g + geom_density(alpha = 0.2, color = "#424242", fill = "#424242") +
  xlab(expression("Number of DBPs")) +
  ylab(expression("Density")) +
  ggtitle("DBPs distribution mRNA and lncRNA genes",
          subtitle = "Promoters") 
```

![](08_01_global_peak_properties_genebody_NR_files/figure-markdown_github/unnamed-chunk-2-1.png)

``` r
ggsave("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/08_genebody_vs_promoter/figures/DBPs_distribution_promoter.png", height = 5, width = 8)


# genebody
g <- ggplot(genebody_peak_occurence_df, aes(x = number_of_dbp))
g + geom_density(alpha = 0.2, color = "#424242", fill = "#424242") +
  xlab(expression("Number of DBPs")) +
  ylab(expression("Density")) +
  ggtitle("DBPs distribution mRNA and lncRNA genes",
          subtitle = "Genebody") 
```

![](08_01_global_peak_properties_genebody_NR_files/figure-markdown_github/unnamed-chunk-2-2.png)

``` r
ggsave("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/08_genebody_vs_promoter/figures/DBPs_distribution_genebody.png", height = 5, width = 8)


# genebody without promoter
g <- ggplot(genebodyMinusPromoter_df, aes(x = number_of_dbp))
g + geom_density(alpha = 0.2, color = "#424242", fill = "#424242") +
  xlab(expression("Number of DBPs")) +
  ylab(expression("Density")) +
  ggtitle("DBPs distribution mRNA and lncRNA genes",
          subtitle = "Genebody without Promoter") 
```

![](08_01_global_peak_properties_genebody_NR_files/figure-markdown_github/unnamed-chunk-2-3.png)

``` r
ggsave("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/08_genebody_vs_promoter/figures/DBPs_distribution_genebodyMinusPromoter.png", height = 5, width = 8)
```

make a new df to merge all number\_of\_dbp
==========================================

``` r
# make a new df to merge all number_of_dbp
GB <- genebody_peak_occurence_df %>%
  dplyr::select(gene_length, gene_id, number_of_dbp) %>%
  dplyr::rename(number_of_dbp_genebody = number_of_dbp)

promoter <- promoter_peak_occurence_df %>%
  dplyr::select(gene_id, gene_name, gene_type,chr, X3kb_up_tss_start, strand, number_of_dbp) %>%
  dplyr::rename(number_of_dbp_promoter = number_of_dbp)

GB_no_promoter <- genebodyMinusPromoter_df %>%
  dplyr::select(gene_id, number_of_dbp, new_gene_length) %>%
  dplyr::rename(number_of_dbp_genebodyNoPromoter = number_of_dbp)

merge_promoter_GB <- merge(promoter, GB, by = "gene_id")
peak_occurance_all_df <- merge(merge_promoter_GB, GB_no_promoter, by = "gene_id")

write_csv(peak_occurance_all_df, "/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/08_genebody_vs_promoter/results/peak_occurance_all_df.csv")
```

make densiry plot with all types
================================

``` r
# select only number_of_dbp columns and rearange
sele <- peak_occurance_all_df %>% dplyr::select(number_of_dbp_promoter, number_of_dbp_genebody,number_of_dbp_genebodyNoPromoter )
rev_df = stack(sele)
rev_df_final <- rev_df %>% dplyr::rename(number_of_dbp = values, type = ind)

g <- ggplot(rev_df_final, aes(x = number_of_dbp, group=type, color = type, fill = type))
g + geom_density(alpha = 0.2) +
  xlab(expression("Number of DBPs")) +
  ylab(expression("Density")) +
  ggtitle("DBPs distribution mRNA and lncRNA genes") 
```

![](08_01_global_peak_properties_genebody_NR_files/figure-markdown_github/unnamed-chunk-4-1.png)

``` r
ggsave("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/08_genebody_vs_promoter/figures/DBPs_distribution_promotervsgenebody.png", height = 5, width = 8)
```

correlations
============

``` r
# Correlation between promoter and genebody minus promoter

g <- ggplot(peak_occurance_all_df, aes(x = number_of_dbp_promoter, y =number_of_dbp_genebodyNoPromoter)) 
g + geom_point(alpha=0.3) +
  #geom_abline(slope = 1000, intercept = 0) +
  xlab(expression("Number of DBPs on promoters")) +
  ylab(expression("Number of DBPs on genebody- no promoter")) +
  ggtitle("Correlation between Number of DBPs on promoter vs genebody without promoter")
```

![](08_01_global_peak_properties_genebody_NR_files/figure-markdown_github/unnamed-chunk-5-1.png)

``` r
ggsave("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/08_genebody_vs_promoter/figures/Correlation_DBPs_promotervsgenebody.png", height = 5, width = 8)

# Correlation with length - promoter
g <- ggplot(peak_occurance_all_df, aes(x = number_of_dbp_promoter, y =gene_length)) 
g + geom_point(alpha=0.3) +
  #geom_abline(slope = 1000, intercept = 0) +
  xlab(expression("Number of DBPs on promoters")) +
  ylab(expression("gene length")) +
  ggtitle("Correlation with DBPs on promoter and gene length")
```

![](08_01_global_peak_properties_genebody_NR_files/figure-markdown_github/unnamed-chunk-5-2.png)

``` r
ggsave("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/08_genebody_vs_promoter/figures/Correlation_DBPs_promotervslength.png", height = 5, width = 8)

# Correlation with length - gene body 
g <- ggplot(peak_occurance_all_df, aes(x = number_of_dbp_genebody, y =gene_length)) 
g + geom_point(alpha=0.3) +
  #geom_abline(slope = 1000, intercept = 0) +
  xlab(expression("Number of DBPs on gene")) +
  ylab(expression("gene length")) +
  ggtitle("Correlation with DBPs on promoter and gene length")
```

![](08_01_global_peak_properties_genebody_NR_files/figure-markdown_github/unnamed-chunk-5-3.png)

``` r
ggsave("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/08_genebody_vs_promoter/figures/Correlation_DBPs_on_genebody_vs_length.png", height = 5, width = 8)

# Correlation with length - gene body minus promoter
g <- ggplot(peak_occurance_all_df, aes(x = number_of_dbp_genebodyNoPromoter, y =new_gene_length)) 
g + geom_point(alpha=0.3) +
  #geom_abline(slope = 1000, intercept = 0) +
  xlab(expression("Number of DBPs on gene without promoter")) +
  ylab(expression("'new' gene length")) +
  ggtitle("Correlation with DBPs on promoter and gene length")
```

![](08_01_global_peak_properties_genebody_NR_files/figure-markdown_github/unnamed-chunk-5-4.png)

``` r
ggsave("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/08_genebody_vs_promoter/figures/Correlation_DBPs_on_genebodyNoPromoter_vs_new_length.png", height = 5, width = 8)


#### look only at high binders 
promoter_high_binders_df <- peak_occurance_all_df %>% filter(peak_occurance_all_df$number_of_dbp_promoter >= 350)

# plot scatter plot for only promoter high binders 
g <- ggplot(promoter_high_binders_df, aes(x = number_of_dbp_promoter, y =number_of_dbp_genebodyNoPromoter)) 
g + geom_point(alpha=0.3) +
  #geom_abline(slope = 1000, intercept = 0) +
  xlab(expression("Number of DBPs on promoters")) +
  ylab(expression("Number of DBPs on genebody- no promoter")) +
  ggtitle("Correlation between Number of DBPs on promoter vs genebody without promoter",
          subtitle = "promoter high binders")
```

![](08_01_global_peak_properties_genebody_NR_files/figure-markdown_github/unnamed-chunk-5-5.png)

``` r
ggsave("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/08_genebody_vs_promoter/figures/Correlation_DBPs_promotervsgenebody_high_binders.png", height = 5, width = 8)
```

define high binders for both vs only for promoters if needed later
==================================================================

``` r
#### look only at high binders 
high_binders_both_df <- peak_occurance_all_df %>% filter((peak_occurance_all_df$number_of_dbp_promoter >= 350) & (peak_occurance_all_df$number_of_dbp_genebodyNoPromoter >= 350))

high_binders_only_promoter_df <- peak_occurance_all_df %>% filter((peak_occurance_all_df$number_of_dbp_promoter >= 350) & (peak_occurance_all_df$number_of_dbp_genebodyNoPromoter < 350))
```