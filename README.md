# QuASAR: Quantitative allele specific analysis of reads
QuASAR is an R package, that implements a statistical method for: i) joint genotyping across sequencing datasets of the same individual, ii) identifying heterozygous loci, and iii) conducting inference on allelic imbalance. 
The sequencing data can be RNA-seq, DNase-seq, ATAC-seq or any other type of high-throroughput sequencing data. 
The input data to QuASAR is a processed pileup files (as will be detailed later). 
Here, we do not cover in depth important pre-processing steps such as choice of the aligner, read filtering and duplicate removal. 

We also want to emphisize that the current software is still in development, we would kindly appreciate any comments and bug reports. 

<!---
Prior to analsyis, RNA-Seq data must undergo alignment with a modern aligner, quality filtering, duplicate removal, and the creation of pileups. There are many tools and tutorials available for preprocessing Next Generation Sequencing data, but we will only describe the tools we used and expect the user to have basic familiarity with standard bioinformatics command-line tools. Our goal with this tutorial is to cover the following:

1. Installing QuASAR
2. Preprocessing 
   * Alignment, filtering, and removing duplicates. (Description of, not a tutorial how)
   * Pileups and clean pileups
3. QuASAR analyis pipeline
   * Genotyping single or multiple samples
   * Inference on ASE
   * Sample workflow

**Quick-start**: Users comfortable processing RNA-Seq data to the level of pileups should skip to the second step of preprocessing. 
-->

## 1. Installation

```R
require(devtools)
install_github('QuASAR', 'piquelab')
library('QuASAR')
```

Installing R packages from GitHub within an R session has problems sometimes. Alternatively, you can clone/fork this repository then and then build the package:

```C
git clone git@github.com:piquelab/QuASAR.git
R CMD build QuASAR
```

## 2. Preprocessing
### Alignment & filtering
Raw reads can be aligned to the reference genome using your favorite aligner. Because allele-specific analysis is extremely sensitive to read biases and mapping errors, we strongly recommend adding steps to remove PCR duplicates and to remove reads aligning to areas with known mappability issues (e.g., [Degner et al, 2009]).


### Pileups & cleaned pileups
Using the samtools mpileup command, create a pileup file from aligned reads. Provide a fasta-formatted reference genome (hg19.fa) and a bed file of positions you wish to pileup on (e.g., 1KG SNP positions):

```C
samtools mpileup -f hg19.fa -l snps.af.bed input.bam | gzip > input.pileup.gz
```

Next, convert the pileup file into bed format and use intersectBed to include the allele frequencies from a bed file. The bed file with allele frequences should be five colums: 1-3) coordinate, 4) SNP ID, 5) allele frequency. The awk filter step (below) removes positions not covered by a read, positions covered by indels, and reference skips:

```C
less input.pileup.gz | awk -v OFS='\t' '{ if ($4>0 && $5 !~ /[^\^][<>]/ && $5 !~ /\+[0-9]+[ACGTNacgtn]+/ && $5 !~ /-[0-9]+[ACGTNacgtn]+/ && $5 !~ /[^\^]\*/) print $1,$2-1,$2,$3,$4,$5,$6}' | sortBed -i stdin | intersectBed -a stdin -b snps.af.bed -wo | cut -f 1-7,11-14 | gzip > input.pileup.bed.gz
```

Finally, get the read counts at each position, and, if desired, perform any additional filtering. The result will be the input file for QuASAR. An example processing script is provided here: scripts/convertPileupToQuasar.R.

```C
R --vanilla --args input.pileup.bed.gz < convertPileupToQuasar.R
```

The final file should look something like this:

```C
zless input.quasar.in.gz | head -5
chr1	879910	879911	G	A	rs143853699	0.02	21	0	0
chr1	892379	892380	G	A	rs150615968	0.0041	22	0	0
chr1	893384	893385	G	A	rs140972868	0.01	6	0	0
chr1	894101	894102	A	T	rs188691615	0.01	6	0	0
chr1	894430	894431	G	A	rs201791495	9e-04	9	0	0
```

The final fields are as follows:
1. Chromosome
2. Start position
3. End position
4. Reference allele
5. Alternate allele
6. SNP ID
7. SNP allele frequency
8. Number of reads mapping to the reference allele
9. Number of read mapping to the alternate allele
10. Number of reads not mapping to either allele

## 3. Running QuASAR

### Prepare the input samples 
To run the provided sample data, or any data, we provide a few helper functions to merge samples across the union of all annotated loci `UnioinExtractFields`, .  To `fdgf`
```R
folderName="~/QuASAR/sampleinput/"
fileNames=paste0(folderName,dir(folderName,"Et.*gz"))
ase.dat <- UnionExtractFields(fileNames, combine=TRUE)
ase.dat.gt <- PrepForGenotyping(ase.dat, min.coverage=5)
sample.names <- colnames(ase.dat.gt$ref)
```
### Genotype multiple samples

```R
ase.joint <- fitAseNullMulti(ase.dat.gt$ref, ase.dat.gt$alt, log.gmat=log(ase.dat.gt$gmat))
```
    
### Inference on ASE
```R
ourInferenceData <- aseInference(gts=ase.joint$gt, eps.vect=ase.joint$eps, priors=ase.dat.gt$gmat, ref.mat=ase.dat.gt$ref, alt.mat=ase.dat.gt$alt, min.cov=10, sample.names=sample.names, annos=ase.dat.gt$annotations)
```
### Sample workflow


<!-- links -->
[Degner et al, 2009]:http://www.ncbi.nlm.nih.gov/pubmed/19808877
