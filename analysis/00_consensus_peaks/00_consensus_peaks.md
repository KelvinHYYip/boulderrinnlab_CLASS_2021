Create consensus peaks
======================

Creating and exporting consensus peaks from individual peak files Note you will want to place these files in your folder and make a "results/chipseq/consensus\_peaks" directory path

``` r
consensus_peaks <- create_consensus_peaks("/scratch/Shares/rinnclass/data/peaks")

for(i in 1:length(consensus_peaks)) {
  rtracklayer::export(consensus_peaks[[i]], 
                      paste0("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/00_consensus_peaks/results/consensus_peaks/", 
                             names(consensus_peaks)[i], 
                             "_consensus_peaks.bed"))
}
```

Filtering for number of peaks
=============================

Filtering the consensus peaks to files with at least 250 peaks (per last years class cut-off)

``` r
num_peaks_threshold <- 250

filtered_consensus_peaks <- consensus_peaks[sapply(consensus_peaks, length) > num_peaks_threshold]

#Export consensus peaks
for(i in 1:length(filtered_consensus_peaks)) {
  rtracklayer::export(filtered_consensus_peaks[[i]], 
                      paste0("/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/00_consensus_peaks/results/filtered_consensus/", 
                             names(filtered_consensus_peaks)[i], 
                             "_filtered_consensus_peaks.bed"))
}
```

1) Promoter Regions
===================

Genomic features you will want to save directly to your results folder since they will be used for all analyses and not just chipseq (where we exported consensus and filtered consensus peaks)

``` r
gencode_gr <- rtracklayer::import("/Shares/rinn_class/data/genomes/human/gencode/v32/gencode.v32.annotation.gtf")

lncrna_mrna_promoters <- get_promoter_regions(gencode_gr, biotype = c("lncRNA", "protein_coding"))
rtracklayer::export(lncrna_mrna_promoters, "/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/00_consensus_peaks/results/promoters/lncrna_mrna_promoters.gtf")


lncrna_promoters <- get_promoter_regions(gencode_gr, biotype = "lncRNA")

rtracklayer::export(lncrna_promoters, "/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/00_consensus_peaks/results/promoters/lncrna_promoters.gtf")


mrna_promoters <- get_promoter_regions(gencode_gr, biotype = "protein_coding")
rtracklayer::export(mrna_promoters, "/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/00_consensus_peaks/results/promoters/mrna_promoters.gtf")
```

2) Gene Bodies
==============

``` r
lncrna_mrna_genebody <- gencode_gr[gencode_gr$type == "gene" & 
                                     gencode_gr$gene_type %in% c("lncRNA", "protein_coding")]
rtracklayer::export(lncrna_mrna_genebody, "/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/00_consensus_peaks/results/genebody/lncrna_mrna_genebody.gtf")


lncrna_genebody <- gencode_gr[gencode_gr$type == "gene" & 
                                gencode_gr$gene_type %in% c("lncRNA")]
rtracklayer::export(lncrna_genebody, "/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/00_consensus_peaks/results/genebody/lncrna_genebody.gtf")


mrna_genebody <- gencode_gr[gencode_gr$type == "gene" & 
                              gencode_gr$gene_type %in% c("protein_coding")]
rtracklayer::export(mrna_genebody, "/scratch/Shares/rinnclass/tardigrades/CLASS_2021/analysis/00_consensus_peaks/results/genebody/mrna_genebody.gtf")
```