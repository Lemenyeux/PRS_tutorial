Next, we'd like to perform basic quality controls (QC) on the target genotype data. 

In this tutorial, we've simulated some samples using the 1000 genome european genotypes. 
You can download the data [here](https://github.com/choishingwan/PRS-Tutorial/raw/master/resources/EUR.zip). 

Unzip the data as follow:

```bash
unzip EUR.zip
```

??? note "What's the md5sum of the genotype files?"

    |File|md5sum|
    |:-:|:-:|
    |**EUR.bed**           |96ce8f494a57114eaee6ef9741676f58|
    |**EUR.bim**           |852d54c9b6d1159f89d4aa758869e72a|
    |**EUR.covariate**     |afff13f8f9e15815f2237a62b8bec00b|
    |**EUR.fam**           |8c6463c0d8f32f975cdc423b1b80a951|
    |**EUR.height**        |052beb4cae32ac7673f1d6b9e854c85b|

!!! note
    We assume PLINK is installed in your PATH directory, which allow us to use `plink` instead of `./plink`.
    If PLINK is not in your PATH directory, replace all instance of `plink` in the tutorial to `./plink` assuming
    the PLINK executable is located within your working directory

# Genotype file format

# Basic filterings
The power and validity of PRS analyses are highly dependent on the quality of the base and target data, therefore 
both data sets must be quality controlled to the high standards implemented in GWAS studies, e.g. removing SNPs according to low genotyping rate, minor allele frequency and individuals with low genotyping rate (see [Marees et al](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6001694/)).

The following `plink` command perform some basic filterings

```bash
plink --bfile EUR \
    --maf 0.05 \
    --hwe 1e-6 \
    --geno 0.01 \
    --mind 0.01 \
    --make-bed \
    --out EUR.QC
```
Each of the parameters corresponds to the following

| Paramter | Value | Description|
|:-:|:-:|:-|
| bfile | EUR | Inform `plink` that the input genotype files should have a prefix of `EUR` |
| maf | 0.05 | Try to filter out any SNPs with minor allele frequency less than 0.05. Genotype error are more likely to influence SNPs with low MAF. Large sample size can adapt a lower MAF threshold|
| hwe | 1e-6 | Filtering SNPs with low p-value from the Hardy-Weinberg exact test. SNPs with significant p-value from the HWE test are more likely to harbor genotyping error or are under selection. Filtering should be performed on the control samples to avoid filtering SNPs that are causal (under selection in cases)|
| geno | 0.01 | Exclude SNPs that are missing in large proportion of subjects. A two pass filtering is usually performed (see [Marees et al](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6001694/)).|
| mind | 0.01 | Exclude individual who have a high rate of genotype missingness. This might indicate problems in the DNA sample. (see [Marees et al](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6001694/) for more details).|
| make-bed | - | Inform `plink` to generate a binary genotype file |
| out | EUR | Inform `plink` that all output should have a prefix of `EUR` |

??? note "How many SNPs were filtered?"
    A total of `266` SNPs were removed due to Hardy-Weinberg exact test results

# Filter related samples
Related samples in the target data might lead to overfitted results, hampering the generalizability of the results. 

To remove related samples, we first need to perform prunning to remove highly correlated SNPs:
```bash
plink \
    --bfile EUR.QC \
    --indep-pairwise 200 50 0.25 \
    --out EUR.QC
```

This will generate two files 1) **EUR.QC.prune.in** and 2) **EUR.QC.prune.out**
The **EUR.QC.prune.in** file contains SNPs that has r2 less than 0.25 between them. 

Samples with more than third-degree relatedness (pi-hat > 0.125) can then be removed with 

```bash
plink \
    --bfile EUR.QC \
    --extract EUR.QC.prune.in \
    --rel-cutoff 0.125 \
    --out EUR.QC
```
The IDs of the un-related samples are located in **EUR.QC.rel.id**, which can then be extracted using
```bash
plink \
    --bfile EUR.QC \
    --keep EUR.QC.rel.id \
    --make-bed \
    --out EUR.QC.unrel
```

# Remove samples with abnormal heterozygosity rate
Individual with high or low heterozygosity rate can be contaminated or are inbreed.
It is therefore a good idea to remove these samples from our dataset before continuing the analyse.
Heterozygosity rate can be calculated using `plink` after performing prunning. 
```bash
plink \
    --bfile EUR.QC.unrel \
    --extract EUR.QC.prune.in \
    --het \
    --out EUR
```
This will generate the **EUR.het** file which contains the F coefficient estimates.
It will be easier to filter the samples using `R` instead of `awk`:
Open a `R` section by tying `R` in your terminal
```R
dat <- read.table("EUR.het", header=T) # Read in the EUR.het file, specify it has header
m <- mean(dat$F) # Calculate the mean  
s <- sd(dat$F) # Calculate the SD
problem <- subset(dat, F> m+3*s | F< m-3*s) # Get any samples with F coefficient 3 sd away from population mean
write.table(problem[,c(1,2)], "EUR.prob.sample", quote=F, row.names=F) # print FID and IID for problematic samples
```
With **EUR.prob.sample**, we can then remove the problematic samples from our genotype file using `plink`
```bash
plink \
    --bfile EUR.QC.unrel \
    --remove EUR.prob.sample \
    --make-bed \
    --out EUR.QC.unrel.het
```

# Check for mis-matched Sex information

!!! note
    As sex chromosome information are missing from our simulated samples. 
    It is not possible to perform the mis-match sex check. (And we used simulated sex)
    Thus, this section is only served as a reference in case your samples contain the 
    sex chromosome and sex information which permits checking if there are mis-matched sex information

Sometimes, sample mislabeling can occur, which can lead to invalid results. 
A good indication of mislabeled sample is a mismatch between the biological sex and the reported sex. 
If the biological sex does not match up with the reported sex, it is likely that the sample has been mislabeled.

Before performing sex check, prunning should be performed (see [here](target.md#filter-related-samples)).
Sex check can then easily be carried out using `plink`
```bash
plink \
    --bfile EUR.QC.unrel.het \
    --extract EUR.QC.prune.in \
    --check-sex \
    --out EUR
```

This will generate a file called **EUR.sexcheck** containing the F-statistics for each individual.
For male, the F-statistic should be > 0.8 and Female should have a value < 0.2.

```bash
awk 'NR==FNR{a[$1]=$5} \
    NR!=FNR && a[$1]==1 && $6 > 0.8 {print $1,$2} \
    NR!=FNR && a[$1]==2 && $6 < 0.2 {print $1,$2} ' \
    EUR.QC.unrel.het.fam EUR.sexcheck > EUR.valid.sex 
```
Here is a breakdown of the above script
1. Read in the first file (`NR==FNR`) and store the sex (`$5`) into a dictionary using the FID (`$1`) as the key
2. Read in the second file (`NR!=FNR`), if the individual is a male (`a[$1]==1`) and the F-statistic (`$6`) is larger than 0.8, print its FID and IID
3. Read in the second file (`NR!=FNR`), if the individual is a female (`a[$1]==2`) and the F-statistic (`$6`) is less than 0.2, print its FID and IID

The samples can then be extracted using the `--keep` command.

# Remove Ambiguous SNPs
If the base and target data were generated using different genotyping chips and the chromosome strand (+/-) for either is unknown, then it is not possible to match ambiguous SNPs (i.e. those with complementary alleles, either C/G or A/T) across the data sets, because it will be unknown whether the base and target data are referring to the same allele or not. 

Ambiguous SNPs can be obtained by examining the bim file:
```bash
awk '!( ($5=="A" && $6=="T") || \
        ($5=="T" && $6=="A") || \
        ($5=="G" && $6=="C") || \
        ($5=="C" && $6=="G")) {print}' \
        EUR.QC.unrel.het.bim > EUR.unambig.snp 
```

The ambiguous SNPs can then be removed by

```bash
plink \
    --bfile EUR.QC.unrel.het \
    --extract EUR.unambig.snp \
    --make-bed \
    --out EUR.QC.unrel.het.NoAmbig
```

??? note "How many ambiguous SNPs were there?"
    There are `17,260` ambiguous SNPs