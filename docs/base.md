# Obtaining the base data file
The first step in Polygenic Risk Score (PRS) analyses is to generate or obtain the base data (GWAS summary statistics). Ideally these will correspond to the most powerful GWAS results available on the phenotype under study. In this example, we will use a modified version of the Height GWAS summary statistics generated by the [GIANT consortium](https://portals.broadinstitute.org/collaboration/giant/index.php/GIANT_consortium_data_files#GWAS_Anthropometric_2014_Height). You can download the summary statistic file [here](https://github.com/choishingwan/PRS-Tutorial/raw/master/resources/GIANT.height.gz) or you can use the following bash command:
``` bash

curl https://github.com/choishingwan/PRS-Tutorial/raw/master/resources/GIANT.height.gz -L -O

```

which will create a file called **GIANT.height.gz** in your working directory. 

!!! warning
    If you download the summary statistics without using the bash command, and are using a MAC machine, the gz file will be decompressed automatically, resulting in a **GIANT.height** file instead. 
    
    To maintain consistency, we suggest compressing the **GIANT.height** file with 
    ```
    gzip GIANT.height
    ```
    before starting the tutorial
    


# Reading the base data file
**GIANT.height.gz** is compressed. To read its content, you can type:

```bash
gunzip -c GIANT.height.gz | head
```

which will display the first 10 lines of the file

!!! note
    Working with compressed files reduces storage space requirements

The **GIANT.height.gz** file contains the following columns:

|SNP|CHR|BP|A1|A2|MAF|SE|P|N|INFO|OR|
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|rs2073813	|1	|753541	|A	|G	|0.125	|0.0083	|0.68	|69852	|0.866425782879888	|0.996605773454898|
rs12562034	|1	|768448	|A	|G	|0.092	|0.0088	|0.55	|88015	|0.917520990188678	|0.994714020220009|
rs2980319	|1	|777122	|A	|T	|0.125	|0.006	|0.65	|148975	|0.847126999058955	|0.997303641721713|

The column headers correspond to the following: 

- **SNP**: SNP ID, usually in the form of rs-ID
- **CHR**: The chromosome in which the SNP resides
- **BP**: Chromosomal co-ordinate of the SNP
- **A1**: The effect allele of the SNP
- **A2**: The non-effect allele of the SNP
- **MAF**: The minor allele frequency (MAF) of the SNP
- **SE**: The standard error (SE) of the effect size esimate
- **P**: The P-value of association between the SNP genotypes and the base phenotype
- **N**: Number of samples used to obtain the effect size estimate
- **INFO**: The imputation information score
- **OR**: The effect size estimate of the SNP, if the outcome is binary/case-control. If the outcome is continuous or treated as continuous then this will be the BETA

# QC checklist: Base data

Below we perform QC on these base data according to the 'QC checklist' in our [guide paper](https://doi.org/10.1101/416545), which we recommend that users follow while going through this tutorial and when performing PRS analyses:

# \# Heritability check
We recommend that PRS analyses are performed on base data with a chip-heritability estimate $h_{snp}^{2} > 0.05$. 
The chip-heritability of a GWAS can be estimated using e.g. LD Score Regression (LDSC). 
Our GIANT height GWAS data are known to have a chip-heritability much greater than 0.05 and so we can move on to the next QC step. 

# \# Effect allele
The GIANT consortium report which is the effect allele and which is the non-effect allele in their results, critical for PRS association results to be in the correction direction.

!!! Important
    Some GWAS results files do not make clear which allele is the effect allele and which is the non-effect allele.
    If the incorrect assumption is made in computing the PRS, then the effect of the PRS in the target data will be in the wrong direction.

    To avoid misleading conclusions the effect allele from the base (GWAS) data must be known.


# \# File transfer

A common problem is that the downloaded base data file can be
corrupted during download, which can cause PRS software to crash 
or to produce errors in results. 
However, a `md5sum` hash is 
generally included in files so that file integrity can be checked. 
The following command performs this `md5sum` check: 

```bash tab="Linux"
md5sum GIANT.height.gz
```

```bash tab="OS X"
md5 GIANT.height.gz
```


if the file is intact, then `md5sum` generates a string of characters, which in this case should be: `80e48168416a2fdbe88d68cdfebd4ca2`. 
If a different string is generated, then the file is corrupted.

# \# Genome build
The GIANT summary statistic are on the same genome build as the target data that we will be using. 
You must check that your base and target data are on the same genome build, and if they are not then use a tool such as [LiftOver](https://genome.ucsc.edu/cgi-bin/hgLiftOver) to make the builds consistent across the data sets.

# \# Standard GWAS QC
As described in the paper, both the base and target data should be subjected to the standard stringent QC steps performed in GWAS. 
If the base data have been obtained as summary statistics from a public source, then the typical QC steps that you will be able to perform on them are to filter the SNPs according to INFO score and MAF. 
SNPs with low minor allele frequency (MAF) or imputation information score (INFO) are more likely to generate false positive results due to their lower statistical power (and higher probability of genotyping errors in the case of low MAF). 
Therefore, SNPs with low MAF and INFO are typically removed before performing downstream analyses.
We recommend removing SNPs with MAF < 1% and INFO < 0.8 (with very large base sample sizes these thresholds could be reduced if sensitivity checks indicate reliable results).
These SNP filters can be achieved using the following code:

```bash
gunzip -c GIANT.height.gz |\
awk 'NR==1 || ($6 > 0.01) && ($10 > 0.8) {print}' |\
gzip  > Height.gz
```

The bash code above does the following:

1. Decompresses and reads the **GIANT.height.gz** file
2. Prints the header line (`NR==1`)
3. Prints any line with MAF above 0.01 (`$6` because the sixth column of the file contains the MAF information)
4. Prints any line with INFO above 0.8 (`$10` because the tenth column of the file contains the INFO information)
5. Compresses and writes the results to **Height.gz**

# \# Ambiguous SNPs
If the base and target data were generated using different genotyping chips and the chromosome strand (+/-) that was used for either is unknown, then it is not possible to pair-up the alleles of ambiguous SNPs (i.e. those with complementary alleles, either C/G or A/T SNPs) across the data sets, because it will be unknown whether the base and target data are referring to the same allele or not. While allele frequencies could be used to infer which alleles are on the same strand, the accuracy of this could be low for SNPs with MAF close to 50% or when the base and target data are from different populations. Therefore, we recommend removing all ambiguous SNPs to avoid introducing this potential source of systematic error.

Ambiguous SNPs can be removed in the base data and then there will be no such SNPs in the subsequent analyses, since analyses are performed only on SNPs that overlap between the base and target data.

Nonambiguous SNPs can be retained using the following:
```bash
gunzip -c Height.gz |\
awk '!( ($4=="A" && $5=="T") || \
        ($4=="T" && $5=="A") || \
        ($4=="G" && $5=="C") || \
        ($4=="C" && $5=="G")) {print}' |\
    gzip > Height.noambig.gz
```

??? note "How many non-ambiguous SNPs were there?"
    There are `609,041` ambiguous SNPs


# \# Mismatching SNPs
SNPs that have mismatching alleles reported in the base and target data are either resolvable by "strand-flipping" the alleles to their complementary alleles in e.g. the target data, such as for a SNP with A/C in the base data and G/T in the target, or non-resolvable, such as for a SNP with C/G in the base and C/T in the target. 
Most polygenic score software perform strand-flipping automatically for SNPs that are resolvable, and remove non-resolvable mismatching SNPs.

Since we need the target data to know which SNPs have mismatching alleles, we will perform this strand-flipping in the target data.

# \# Duplicate SNPs
If an error has occurred in the generation of the base data then there may be duplicated SNPs in the base data file.
Most PRS software do not allow duplicated SNPs in the base data input and thus they should be removed, using a command such as the one below: 

```bash
gunzip -c Height.noambig.gz |\
awk '{ print $1}' |\
sort |\
uniq -d > duplicated.snp
```

The above command does the following:

1. Decompresses and reads the **Height.noambig.gz** file
2. Prints out the first column of the file (which contains the SNP ID; change `$1` to another number if the SNP ID is located in another column, e.g. `$3` if the SNP ID is located on the third column)
3. Sort the SNP IDs. This will put duplicated SNP IDs next to each other
4. Print out any duplicated SNP IDs using the uniq command and print them to the *duplicated.snp* file


??? note "How many duplicated SNPs are there?"
    There are a total of `10` duplicated SNPs

Duplicated SNPs can then be removed using the `grep` command:
```bash
gunzip -c Height.noambig.gz  |\
grep -vf duplicated.snp |\
gzip - > Height.QC.gz
```

The above command does the following:

1. Decompresses and reads the **Height.noambig.gz** file
2. From the file, remove (`-v`) any lines contains string within the **duplicated.snp** file (`-f`)
3. Compresses and writes the results to **Height.QC.gz**

# \# Sex chromosomes 
Sometimes sample mislabelling can occur, which may lead to invalid results. One indication of a mislabelled sample is a difference between reported sex and that indicated by the sex chromosomes. While this may be due to a difference in sex and gender identity, it could also reflect mislabeling of samples or misreporting and, thus, individuals in which there is a mismatch between biological and reported sex are typically removed. See the Target Data section in which a sex-check is performed.

# \# Sample overlap
Since the target data were simulated there are no overlapping samples between the base and target data here (see the relevant section of [the paper](https://doi.org/10.1101/416545) for discussion of the importance of avoiding sample overlap). 

# \# Relatedness
Closely related individuals within and between the base and the target data may lead to overfitted results, limiting the generalizability of the results (see the relevant sections of [the paper](https://doi.org/10.1101/416545)). Relatedness within the target data is tested in the Target Data section.

The **Height.QC.gz** base data are now ready for using in downstream analyses.

    
