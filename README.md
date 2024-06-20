# LAI_pipeline
Pipeline for local ancestry inference on WES data

## Reference Panel
Performing local ancestry inference on WES data posed two challenges:
1. **Inadequate intersection between reference and query** Since local \
ancestry inference is traditionally done with SNP array data, there will be a \
very small overlap between the WES force genotype calls and the reference \
panel.
2. **Lack of indigenous American references** The INMEGEN dataset has many \
usable AMR samples, but has poor intersection with the query. That \
intersection becomes even smaller when the INMEGEN datset is combined with the \
1000genomes and Genome Asia datasets.

To address these issues, we created a new reference panel with the goal of \
having good coverage of the exome. We selected 19,611,021 markers from \
1000genomes (MAF > 0.01). Since most of the AMR samples in 1000genomes are \
admixed, we supplemented them with 142 Colombian exomes (coverage 100x, \
SureSelect v7) that were sequenced as part of another project in the lab. We \
used Illumina's DRAGEN v4.2 to perform force genotyping (FGT) at all \
19,611,021 sites, which yielded 438,375 markers covered by both 1000genomes \
and WES FGT calls.
