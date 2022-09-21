TSS profile plots
=================

Import the data and plot POLR2A metaplot
----------------------------------------

-   consensus peaks
-   Gencode gene annotations

``` r
gencode_gr <- rtracklayer::import("/Shares/rinn_class/data/genomes/human/gencode/v32/gencode.v32.annotation.gtf")

data_path <- "/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/"
consensus_peaks <- import_peaks(file.path(data_path, "00_consensus_peaks/results/filtered_consensus"))
```

Step 1: Create promoter windows
-------------------------------

We're going to create a GRanges object that contains 6kb promoter windows for each gene in the Gencode annotation. First we'll need to filter the Gencode GRanges object to just the genes and then we can use the promoter function from GRanges that will allow us to specify how big of a window we want upstream and downstream of the TSS (you can have asymmetrical windows).

``` r
# The gencode annotation contains an entry for each exon, transcript, etc.
# Use table(gencode_gr$type) to see how many of each entry there are. 
# We don't want to create a "promoter" window around each exon for example
# which is why we need to filter to just the genes.
genes <- gencode_gr[gencode_gr$type == "gene"]
# This function is a convenience function built into GenomicRanges
all_promoters_gr <- promoters(genes, upstream = 3e3, downstream = 3e3)
#table(width(all_promoters_gr))
```

Step 2: Transform peaks into a coverage object
----------------------------------------------

In order to calculate what the peak coverage across each promoter is we'll convert the peaks GRanges object which currently holds a range for each peak into a run-length encoded list where 0 represents the genomic coordinates where there is no peak present and 1 represents the locations where a peak is present. The reason for run length encoding is that storing this vector without this compression means that we would be storing a numeric value for each of the 3.2 billion base pairs. This would mean allocating a vector in memory that's ~180 GB -- instead with run-length encoding we're in the ~100 KB range.

``` r
# We can use any binding protein here (use names(consensus_peaks) to see the available DBPs)
peak_coverage <- coverage(consensus_peaks[["POLR2A"]])
```

### Step 2.1: Some housekeeping to keep our chromosomes straight

This step will accomplish two things: filter out promoters that fall outside the bounds of our coverage vectors and filter out chromosomes that are not in common between the promoters object and the peak coverage object. The reason we need to do this is because the peaks may not extend to the end of each chromosome and therefore there is likely to be promoters that fall outside of the peak coverage vectors -- since we know that there are no peaks on those promoters and therefore they don't give us any extra information about where peaks are relative to promoters we'll filter them out. Also, it creates problems for the Views command that we'll use to subset the coverage vectors to just the promoter windows.

``` r
##### Filter promoters to bounds of peak coverage vectors

# This is the length of each run-length encoded vector in the peak_coverage object
# If the last peak on each chromosome falls near the end of that chromosome then
# these lengths will be approximately the length of the chromosomes.
coverage_length <- elementNROWS(peak_coverage)

# This will create a GRanges object where there is one range per chromosome
# and it is the width of the coverage vector -- we can use these ranges to 
# filter the promoters falling outside of these boundaries in the next step.
# Each DBP will be different here.
coverage_gr <- GRanges(seqnames = names(coverage_length),
                       IRanges(start = rep(1, length(coverage_length)), 
                               end = coverage_length))

# Okay, now we're all ready to filter out those promoters that fall beyond the bounds of the 
# coverage vector. 
all_promoters_gr <- subsetByOverlaps(all_promoters_gr, 
                                  coverage_gr, 
                                  type="within", 
                                  ignore.strand=TRUE)

##### Keep track of the chromosomes that are in common and filter/reorder to match

# This creates a vector of the chromosomes that are in common between the peak_coverage and promoters.
# Keeping track of this will fix a similar problem to the step above since this DBP may not
# bind on all chromosomes. 
chromosomes <- intersect(names(peak_coverage), unique(as.character(seqnames(all_promoters_gr))))
# We can also ensure they're in the same order and contain the same chromosomes
# by indexing with this vector
peak_coverage <- peak_coverage[chromosomes]
# In order to match the list format of the peak_coverage object
# we'll also coerce the GRanges object into an IntegerRangesList.
# If you recall, one of the main features of GRanges object is capturing
# the chromosome information -- when converting to an IRanges list, 
# each chromosome will be represented by a named element in the list.
all_promoters_ir <- as(all_promoters_gr, "IntegerRangesList")[chromosomes]
```

Step 3: Subset the peak coverage vector to just the promoter windows
--------------------------------------------------------------------

Here we'll use the Views function to mask the peak coverage object everywhere but in the windows of the promoters.

``` r
promoter_peak_view <- Views(peak_coverage, all_promoters_ir)
# Note that these are still in run-length encoding format.
```

Step 4: Contstruct a matrix of the coverage values of each promoter region
--------------------------------------------------------------------------

We'll not just convert the run-length encoding vectors to actual vectors -- note how much larger the object becomes when represented as vectors (use object.size function). Then we'll row bind the vectors into one matrix.

``` r
# Converts the Rle objects to vectors -- this is now a small enough size to do so
# The t() function will transpose the vectors so that the each position in the promoter
# 1 through 6000 is a column. 
promoter_peak_view <- lapply(promoter_peak_view, function(x) t(viewApply(x, as.vector)))

# Convert this to a matrix. 
promoter_peak_matrix <- do.call("rbind", promoter_peak_view)
```

Step 5: Align the positive and negative strand promoters
--------------------------------------------------------

Since the genes that are transcribed from the minus strand will have their upstream and downstream values flipped relative to the plus strand promoters, we need to reverse those vectors so that upstream and downstream values are consistent.

``` r
# We're just going to flip one strand because we want to get them in the same orientation
minus_idx <- which(as.character(strand(all_promoters_gr)) == "-")
promoter_peak_matrix[minus_idx,] <- promoter_peak_matrix[minus_idx, ncol(promoter_peak_matrix):1]

# We can get rid of the rows that have no peaks -- notice
# that this reduces the size of the data quite a bit
# even Pol II is not bound to every promoter

promoter_peak_matrix <- promoter_peak_matrix[rowSums(promoter_peak_matrix) > 0,]
```

Step 6: Sum the columns, normalize, and plot
--------------------------------------------

To summarize this matrix, we'll sum up the number of binding events at each position in this 6kb window. This vector represents the overall peak coverage of each posistion, for the purpose of visualizing this, we'll normalize by the total coverage so that the area under the curve in the plot sums to one.

``` r
peak_sums <- colSums(promoter_peak_matrix)
# Normalization -- this makes it into a sort of density plot -- it sums to 1.
peak_dens <- peak_sums/sum(peak_sums)

# Create a data frame in order to plot this. 
metaplot_df <- data.frame(x = -3e3:(3e3-1), dens = peak_dens)
```

plot POLR2A\_promoter metaplot
==============================

``` r
# Plot it with ggplot geom_line
ggplot(metaplot_df, aes(x = x, y = dens)) + 
  geom_vline(xintercept = 0, lty = 2) + 
  geom_line(size = 1.5) + 
  ggtitle("POLR2A Promoter Metaplot") + 
  scale_x_continuous(breaks = c(-3000, 0, 3000),
                     labels = c("-3kb", "TSS", "+3kb"),
                     name = "") + 
  ylab("Peak frequency")
```

![](04_metaplots_files/figure-markdown_github/unnamed-chunk-9-1.png)

``` r
ggsave("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/04_metaplot/figures/POLR2A_promoter_metaplot.pdf")
```

    ## Saving 7 x 5 in image

make a function with all teh steps from above : "profile\_tss"
==============================================================

``` r
profile_tss <- function(peaks, 
                        promoters_gr,
                        upstream = 3e3,
                        downstream = 3e3) {
  

  peak_coverage <- coverage(peaks)
  
  coverage_length <- elementNROWS(peak_coverage)
  coverage_gr <- GRanges(seqnames = names(coverage_length),
                         IRanges(start = rep(1, length(coverage_length)), 
                                 end = coverage_length))
  
  promoters_gr <- subsetByOverlaps(promoters_gr, 
                                       coverage_gr, 
                                       type="within", 
                                       ignore.strand=TRUE)
  chromosomes <- intersect(names(peak_coverage), 
                           unique(as.character(seqnames(promoters_gr))))
  peak_coverage <- peak_coverage[chromosomes]
  
  promoters_ir <- as(promoters_gr, "IntegerRangesList")[chromosomes]
  
  promoter_peak_view <- Views(peak_coverage, promoters_ir)
  
  promoter_peak_view <- lapply(promoter_peak_view, function(x) t(viewApply(x, as.vector)))
  promoter_peak_matrix <- do.call("rbind", promoter_peak_view)
  
  minus_idx <- which(as.character(strand(promoters_gr)) == "-")
  promoter_peak_matrix[minus_idx,] <- promoter_peak_matrix[minus_idx,
                                                           ncol(promoter_peak_matrix):1]
  
  promoter_peak_matrix <- promoter_peak_matrix[rowSums(promoter_peak_matrix) > 1,]
  
  peak_sums <- colSums(promoter_peak_matrix)
  peak_dens <- peak_sums/sum(peak_sums)
  
  metaplot_df <- data.frame(x = -upstream:(downstream-1),
                            dens = peak_dens)
  
  return(metaplot_df)
}
```

Use this function to make separate plots for lncRNA and mRNA
============================================================

First we'll create separate objects for lncRNA promoters and mRNA promoters, then we'll supply each of these to the new function we just made.

``` r
# lncRNA promoter profiles
lncrna_genes <- genes[genes$gene_type == "lncRNA"]
lncrna_promoters <- promoters(lncrna_genes, upstream = 3e3, downstream = 3e3)

lncrna_metaplot_profile <- profile_tss(consensus_peaks[["POLR2A"]], lncrna_promoters)

# mRNA promoter profiles
mrna_genes <- genes[genes$gene_type == "protein_coding"]
mrna_promoters <- promoters(mrna_genes, upstream = 3e3, downstream = 3e3)

mrna_metaplot_profile <- profile_tss(consensus_peaks[["POLR2A"]], mrna_promoters)

# We can row bind these dataframes so that we can plot them on the same plot
mrna_metaplot_profile$gene_type <- "mRNA"
lncrna_metaplot_profile$gene_type <- "lncRNA"
combined_metaplot_profile <- bind_rows(mrna_metaplot_profile, lncrna_metaplot_profile)

ggplot(combined_metaplot_profile, 
       aes(x = x, y = dens, color = gene_type)) +
  geom_vline(xintercept = 0, lty = 2) + 
  geom_line(size = 1.5) + 
  ggtitle("POLR2A Promoter Metaplot") + 
  scale_x_continuous(breaks = c(-3000, 0, 3000),
                     labels = c("-3kb", "TSS", "+3kb"),
                     name = "") + 
  ylab("Peak frequency") + 
  scale_color_manual(values = c("#424242","#a8404c"))
```

![](04_metaplots_files/figure-markdown_github/unnamed-chunk-11-1.png)

``` r
ggsave("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/04_metaplot/figures/POLR2A_promoter_metaplot_mRNAvsncRNA.pdf")
```

    ## Saving 7 x 5 in image

make metaplot df for all DBPs
=============================

``` r
# Let's now run this for all off the DBPs and compile it into one data frame.
# Let's first define an empty data.frame to which we can row_bind each new one created.

##metaplot_df <- data.frame(x = integer(), dens = numeric(), dbp = character())

# Started run at 9:58 PM -- finished at 1am
# SKIP ZNF382 -- only 19 peaks -- probably none overlap promoters
# Number 391

#part commented out,takes to long to create md file
##for(i in c(1:390, 392:length(consensus_peaks))) {
##  print(names(consensus_peaks)[[i]])
##  tmp_df <- profile_tss(consensus_peaks[[i]], promoters_gr = all_promoters_gr)
##  tmp_df$dbp <- names(consensus_peaks)[[i]]
##  metaplot_df <- bind_rows(metaplot_df, tmp_df)
##}
##write_csv(metaplot_df, "/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/04_metaplot/results/metaplot_df.csv")
#table(names(consensus_peaks) %in% unique(metaplot_df$dbp))
#which(names(consensus_peaks) == "ZNF382")
#Sys.time()
#Sys.Date()
```

plot dendrogram and heatmap of all DBPs
=======================================

select a cluster of the dendrogram
----------------------------------

``` r
# Let's filter out those peaks that don't pass the threshold, since
# in particular here it will make a difference, since the 
# average profile will be very lumpy for those DBPs with few peaks.
passing_peaks <- names(consensus_peaks)[sapply(consensus_peaks, length) > 250]

# Let's cluster the metaplots df. First we need to turn it into a matrix.
metaplot_df <- read_csv("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/04_metaplot/metaplot_df.csv") %>%
  filter(dbp %in% passing_peaks)
```

    ## 
    ## ── Column specification ────────────────────────────────────────────────────────────────────────────────
    ## cols(
    ##   x = col_double(),
    ##   dens = col_double(),
    ##   dbp = col_character()
    ## )

``` r
# Pivot wider into a matrix
metaplot_matrix <- metaplot_df %>% 
  pivot_wider(names_from = x, values_from = dens) %>%
  column_to_rownames("dbp") %>%
  as.matrix()

# Z-Scale the rows
mm_scaled <- t(scale(t(metaplot_matrix)))

# make dendrogram
metaplot_hclust <- hclust(dist(mm_scaled), method = "complete")

# Plot the dendrogram
pdf("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/04_metaplot/figures/tss_profile_dendrogram.pdf", height = 4, width = 9) # 5 and 27 originally
par(cex=0.3)
plot(metaplot_hclust)
dev.off()
```

    ## RStudioGD 
    ##         2

``` r
# Cut the tree to make some clusters
clusters <- cutree(metaplot_hclust, h = 75)
```

![](04_metaplots_files/figure-markdown_github/unnamed-chunk-13-1.png)

``` r
#table(mm_scaled)
?cutree
clus2 <- mm_scaled[clusters == 2,]

#dim before cut
dim(mm_scaled)
```

    ## [1]  459 6000

``` r
#dim after cut
dim(clus2)
```

    ## [1]  132 6000

``` r
metaplot_cut_hclust <- hclust(dist(clus2), method = "complete")
# Plot the dendrogram

pdf("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/04_metaplot/figures/tss_profile_dendrogram_cluster2.pdf", height = 4, width = 9) # 5 and 27 originally
par(cex=0.3)
plot(metaplot_cut_hclust)
dev.off()
```

    ## RStudioGD 
    ##         2

``` r
# make a heatmap:
col_fun <- colorRamp2(c(-3, 0, 3), c("#5980B3", "#ffffff", "#B9605D"))
split <- data.frame(split = c(rep("-3kb",3000), rep("+3kb", 3000)))
pdf("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/04_metaplot/results/tss_profile_heatmap.pdf", height = 30, width = 9)
Heatmap(mm_scaled, cluster_columns = F, col = col_fun, border = TRUE, 
        show_column_names = FALSE,
        use_raster = TRUE,
        row_split = 5,
        column_split = split,
        column_gap = unit(0, "mm"),row_names_gp = gpar(fontsize = 7))
dev.off()
```

    ## RStudioGD 
    ##         2

plot individual tss profiles
============================

``` r
par(cex = 1)
# plot some individual TSS profiles
plot_tss_profile(metaplot_matrix, "H3K4me1", save_pdf = TRUE, save_dir ="/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/04_metaplot/figures/")
```

    ## NULL

    ## NULL

``` r
plot_tss_profile(metaplot_matrix, "NFIC", save_pdf = TRUE, save_dir ="/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/04_metaplot/figures/")
```

![](04_metaplots_files/figure-markdown_github/unnamed-chunk-14-1.png)

    ## NULL

![](04_metaplots_files/figure-markdown_github/unnamed-chunk-14-2.png)

    ## NULL