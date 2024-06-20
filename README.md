# LAI_pipeline
Pipeline for local ancestry inference on WES data

# Reference Panel
Performing local ancestry inference on WES data posed two challenges:
1. **Inadequate intersection between reference and query.** Since local ancestry inference is traditionally done with SNP array data, there will be a very small overlap between the WES force genotype calls and the old reference panel.
2. **Lack of indigenous American references.** The INMEGEN dataset has many usable AMR samples, but has poor intersection with the query. That intersection becomes even smaller when the INMEGEN datset is combined with the 1000genomes and Genome Asia datasets.

To address these issues, we created a new reference panel with the goal of having good coverage of the exome. We selected 19,611,021 markers from 1000genomes (MAF > 0.01). Since most of the AMR samples in 1000genomes are admixed, we supplemented them with 142 Colombian exomes (coverage 100x, SureSelect v7) that were sequenced as part of another project in the lab. We used Illumina's DRAGEN v4.2 to perform force genotyping (FGT) at all 19,611,021 sites, which yielded **438,375 markers** covered by both 1000genomes and WES FGT calls.

**Reference Panel QC.** Using unsupervised ADMIXTURE v1.3 with K=5 on the combined dataset (1000genomes + 142 CLM exomes), selected 50 of AFR, AMR, and EUR samples to comprise the new reference panel (so **150 samples** total). I used plink to check for markers with a low genotyping rate (geno --0.5). The reference panel was purposefully left unfiltered to maximize the intersection with the query samples.

# Method Overview
RFMix v2 is used to generate a local ancestry "landscape" of each query sample. Using the outputs, we calculate the probable local ancestry at a specific loci.

# Step 1: Force genotyping
Example script:
```
dragen -f -r {reference_hash_table} \
--output-directory {sample} \
--output-file-prefix {sample} \
--bam-input {path_to_bam_file} \
--enable-map-align-output false \
--enable-variant-caller true \
--vc-target-bed {target_bed} \
--vc-target-bed-padding 100 \
--vc-enable-phasing true \
--vc-remove-all-soft-clips true \
--vc-emit-ref-confidence gVCF \
--vc-forcegt-vcf {sites_to_genotype_at.vcf}
```
DRAGEN documentation: https://support-docs.illumina.com/SW/dragen_v42/Content/SW/DRAGEN/ForceGenotyping.htm?Highlight=force%20genotyping

After the gVCF has been generated, you can extract the FGT sites using bcftools:
```
bcftools view -o sample.force_genotyped.vcf.gz -i 'FGT==1' sample.input.gvcf.gz
```
Then combine the VCFs for all query samples. [add python script]

# Step 2: Phasing
shapeit4.2 documentation: https://odelaneau.github.io/shapeit4/#documentation

Download these files if they are not already downloaded:
* hg38 genetic maps: https://github.com/odelaneau/shapeit4/blob/master/maps/genetic_maps.b38.tar.gz
* 1000genomes hg38:
```
wget
http://hgdownload.soe.ucsc.edu/gbdb/hg38/1000Genomes/ALL.chr${i}.shapeit2_integrated_sn
vindels_v2a_27022019.GRCh38.phased.vcf.gz
```

Separate the query vcf by chromosome:
```
bcftools view -o chr${i}.force_genotyped.vcf -r 'chr${i}' all_samples.force_genotyped.vcf
```
Remove leading "chr" from all vcf entries
```
sed -i 's/chr//g' chr${i}.force_genotyped.vcf
```
Index with tabix
```
bgzip chr{i}.force_genotyped.vcf
tabix -p vcf chr${i}.force_genotyped.vcf.gz
```
Run shapeit4.2
```
shapeit4.2 \
--input chr${i}.force_genotyped.vcf.gz \
--map chr${i}.b38.gmap.gz \
--region ${i}
--reference ALL.chr${i}.shapeit2_integrated_sn
vindels_v2a_27022019.GRCh38.phased.vcf.gz \
--thread 4 \
--sequencing \
--output chr${i}.phased.vcf.gz \
--log chr${i}.phasing.log
```

# Step 3: Marker Selection
Check for markers with low genotyping rate
```
plink --vcf chr${i}.phased.vcf.gz --geno 0.5 --make-just-bim --out chr${i}.geno_check
```
Use vcftools for maf and hwe filtering
```
vcftools --gzvcf chr${i}.phased.vcf.gz \
--maf 0.01 --hwe 0.0001 \
--recode --out chr${i}.query_filtered
```

# Step 4: Run RFMix
Documentation: https://github.com/slowkoni/rfmix/blob/master/MANUAL.md
