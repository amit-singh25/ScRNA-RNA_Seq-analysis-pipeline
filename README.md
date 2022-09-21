
## Introduction 

In the analysis of high-throughput big genomic sequencing data, it is required to write custom scripts to form the glue between tools or to perform specific analysis tasks. All Squeegeeing data are big size in nature, handling and pre/processing these data required high computing processing power. Generally, most of the tasks are carried out in the computer cluster. Various processing steps are required in order to get the final matrix form of the data for further downstream analysis. The easiest way to install this software is via [Bioconda](https://bioconda.github.io/).

The sequence analysis pipeline involves the below steps.

1. Quality assessment of sequencing data
2. Mapping reads onto a reference genome
3. Quantification and normalization
4. Downstream analysis (Statistic and Machine learning approach)
5. Report and Visualization

Multiple scripting languages are required for complete the genomics sequence analysis, for bioinformatic/sequence analysis Bash and R/biconductor language is reported here and Matlab for mathematical modeling.


## Quality assement of sequencing data 

Most sequencer machines generate a QC report as a part of their analysis pipeline, however, the focal point of the problems generated by the sequencer itself. Different bioinformatics tool aims to provide the information or spot the problem which originates either in the sequencer or in the starting experimental library material. A widely used tool is FastQC. It is performed by a series module and generates an HTML output evaluating pass/ fail results. The different analysis modules namely Basic Statistics, Per Base Sequence Quality, Per Sequence Quality Scores, Per Base Sequence Content, Per Base GC Content, Per Sequence GC Content, Per Base N Content, Sequence Length Distribution, Duplicate Sequences, Overrepresented Sequences, Overrepresented Kmers. Command-line can be used as (fastqc .fastq.gz*) for generating reports. More details of the module can be found [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/).


#### Adapter Remove 

Illumina adapter and other technical sequences are performed by TruSeq2 and TruSeq3 (as used by HiSeq and MiSeq machines)
for both single-end and paired-end modes. Depending on specific issues which may occur in library preparation, other sequences may work better for a given data set. Therefore, generating some quality metrics for raw sequence data is necessary. Different tool used widely such as  [Cutadapt](https://cutadapt.readthedocs.io/), [Trimmomatic](https://github.com/usadellab/Trimmomatic), [Flexbar](https://github.com/seqan/flexbar). Below bash script for Flexbar for trimming raw data. 

````
#!/bin/bash
#PBS -N trim_data
##PBS -j oe
#PBS -m e
#PBS-l mem=100G
#PBS -l walltime=50:13:59
#PBS -l nodes=1:ppn=8   
#PBS -e ./trim_data_$PBS_JOBID.err       
#PBS -o ./trim_data_$PBS_JOBID.out
#PBS -V
#echo $PBS_JOBNAME
#echo $PBS_JOBID
name=$1
data=~/raw_data
trim_data=~/raw_data
flexbar -r ${data}/${name}_1.fq -p ${data}/${name}_2.fq -t ${trim_data}/$(name) - 10 -z BZ2 -m 30 -u 0 -q 28 -a ${data}/Adapter.fa -f sanger

````

## Mapping reads onto a reference genome 

The initial process of sequencing analysis is (Alignment) which involved mapping the reads to the reference genome. This gives the precise location in the genome of each base pair in each sequencing read comes from. Further, mapped reads can be used to identify genetic variation and the genotypes of individuals at different locations in the genome (Variant calling ->Variant Annotation ->Visualization) using [bcftools](https://samtools.github.io/bcftools/bcftools.html), [VEP](https://www.ensembl.org/info/docs/tools/vep/index.htm),[IGV] (https://software.broadinstitute.org/software/igv/). This pipeline is typically used in whole-genome sequence analysis. Additionally, if the data are chip-sequencing the mapped read is used to identify the read distribution profile (Peak calling -> Peak annotation ->Peak visualization).
Different tools are used for the analysis such as [MACS2](https://samtools.github.io/bcftools/), [HOMER](http://homer.ucsd.edu/homer/ngs/peaks.html), [Bedtools](https://bedtools.readthedocs.io/en/latest/), [samtools](http://samtools.sourceforge.net/), [bamtools](https://github.com/pezmaster31/bamtools).
For RNA sequencing, mapped reads are further quantified as counted per gene. Gene-level count datasets for downstream analysis such as exploratory data analysis (EDA) for quality assessment and to explore the relationship between samples, differential gene expression analysis, and visualization of the results. Various software tools are used for alignment to reference the genome. Few names are outlined here, [hisat2](http://ccb.jhu.edu/software/hisat2/), [BWA](http://bio-bwa.sourceforge.net/bwa.shtml), [Bowtie](http://bowtie-bio.sourceforge.net/index.shtml), [subread](http://subread.sourceforge.net/), [STAR](https://github.com/alexdobin/STAR). Below script used for STAR alignment, the upper part of the script is used for indexing the reference genome, later part of the script is used for mapping the raw data. 

#### Align to Reference Genome

```
#!/bin/bash
#PBS -N Star_alignment
##PBS -j oe
#PBS -m e
#PBS-l mem=100G
#PBS -l walltime=50:13:59
#PBS -l nodes=1:ppn=8   
#PBS -e ./Star_alignment_$PBS_JOBID.err       
#PBS -o ./Star_alignment_$PBS_JOBID.out
#PBS -V
#echo $PBS_JOBNAME
#echo $PBS_JOBID
name=$1
ref_genome=~/genome_fold
data=~/raw_data
out=~/alignment
# Script to generate the genome index
STAR --runThreadN 8 \
--runMode genomeGenerate \
--genomeDir ${ref_genome} \
--genomeFastaFiles ${ref_genome} reference.fa \
--sjdbGTFfile ${ref_genome} reference.gtf \
--genomeSAindexNbases 6
--sjdbOverhang 99
# After genome indices generated,read alignment can be perfomed
STAR --runThreadN 8 --genomeDir ${ref_genome} --sjdbGTFfile ${ref_genome}/reference.gtf \
--readFilesCommand zcat \
--readFilesIn ${data}/${name}_1.fastq.gz ${data}/${name}_2.fastq.gz \
--outSAMtype BAM SortedByCoordinate \
--outSAMunmapped Within \
--outSAMattributes Standard 
--outFileNamePrefix ${out}/${name} \
--quantMode ${name}_GeneCounts

```

## Quantification and normalization

Alignments mapped read is counted using [Cufflinks](http://cole-trapnell-lab.github.io/cufflinks/), [eXpress](https://pachterlab.github.io/eXpress/overview.html), [HTSeq](https://htseq.readthedocs.io/en) using both the Intersection-Strict and the Union approaches, [RSEM](https://github.com/deweylab/RSEM), [featureCounts](http://subread.sourceforge.net/featureCounts.html) and [Stringtie](https://ccb.jhu.edu/software/stringtie/). There are some methods do not consider the classical alignment process and carry out alignment, counting and normalization in one single step such as [Kallisto](https://pachterlab.github.io/kallisto/), [Sailfish](https://www.cs.cmu.edu/~ckingsf/software/sailfish/), [Salmon](https://combine-lab.github.io/salmon/getting_started/)
Further, gene expression values are represented using the normalization techniques provided by each algorithm in R/Bioconductor: Transcripts per Million (TPM), Fragments per Kilobase of Mapped reads (FPKM), Trimmed Mean of M values (TMM from edgeR), Relative Log Expression (RLE from DESeq2). 

```
#!/bin/bash
#PBS -N htseq
##PBS -j oe
#PBS -m e
##PBS -l file=100GB
#PBS-l mem=100G
#PBS -l walltime=50:13:59
#PBS -l nodes=1:ppn=8   
#PBS -e ./htseq_$PBS_JOBID.err           # stderr file
#PBS -o ./htseq_$PBS_JOBID.out
#PBS -V
#echo $PBS_JOBNAME
#echo $PBS_JOBID
name=$1
gtf=~/genome
out=~/alignment
data=~/count
htseq-count -f bam \
-r pos \
--type=exon --idattr=gene_id \
--stranded=reverse \
--mode=intersection-nonempty \
${input}/${name}_accepted_hits.bam \
${gtf}/Reference.gtf >${output}/${name}.htseq_count.txt
```

## Down stream analysis (Statistic and Machine learning approach)

A primary objective of many gene expression experiments is to detect transcripts showing differential expression across various conditions.
Downstream analyses with RNA-Seq data include testing for differential expression between samples condition, cluster analysis of selected genes and samples, detecting condition gene-specific expression, GO, signaling pathway analysis (functional enrichment analysis and Network Construction), Dimensional reduction approach for data visualization, identifying novel genes and exons and novel splice junctions, ability to detect gene fusion events.
The below code demonstrates some of the key aspects of RNA sequence downstream analysis.

## Single cell sequencing analysis pipeline

Single-cell transcriptomics determines the gene expression level of individual cells by simultaneously measuring the messenger RNA (mRNA) concentration of hundreds to thousands of genes. It allows the studying of new biological questions in which cell-specific changes in transcriptome are important, e.g. heterogeneity of cell responses, cell type identification, inference of gene regulatory networks across the cells. There are also commercial platforms available for single-cell sequencing, 10X Genomics chromium platform widely used in the present day.
Most computational analysis methods from bulk RNA-seq can be used for the analysis of single-cell sequencing. In most cases, computational analysis requires adaptation of the existing methods or the development of new ones.
Here, analysis code is demonstrated for the 10x genomics chromium platform. The analysis pipeline typically starts as bulk RNA.  
[Cell Ranger](https://support.10xgenomics.com/single-cell-gene-expression/software/pipelines/latest/using/tutorial_in) software used for the single-cell pre-processing analysis. It integrates several tools and reported the count matrix as an output. 

##### Alignemnt and gene count

```
#!/bin/bash
#PBS -N cell_ranger
#PBS -j oe 
##PBS -l file=32GB
#PBS-l mem=100G
#PBS -l walltime=50:13:59
#PBS -l nodes=1:ppn=8	
#PBS -o ./cell_ranger_$PBS_JOBID.out
#PBS -e ./cell_ranger_$PBS_JOBID.err
echo $PBS_JOBID
echo $PBS_JOBNAME
cd $PBS_O_WORKDIR

#echo "=========================================================="
#echo "Starting on : $(date)"
#echo "Running on node : $(hostname)"
#echo "Current directory : $(pwd)"
#echo "Current job ID : $JOB_ID"
#echo "Current job name : $JOB_NAME"
#echo "Task index number : $SGE_TASK_ID"
#echo "=========================================================="

genome=~/genome/Ref_genome
data=~/single_cell/fastqs
name=$1

##### Downlaod the Refernce genome from ensembl(example mouse) and fileter all gene only keep protein biotype before mapping the sequnce data
wget ftp://ftp.ensembl.org/pub/release-93/fasta/mus_musculus/dna/Mus_musculus.GRCm38.dna.primary_assembly.fa.gz
gunzip Mus_musculus.GRCm38.dna.primary_assembly.fa.gz

wget ftp://ftp.ensembl.org/pub/release-93/gtf/mus_musculus/Mus_musculus.GRCm38.93.gtf.gz
gunzip Mus_musculus.GRCm38.93.gtf.gz

gn=~/genome/10x_genome
cellranger mkgtf ${gn}/Refernce.gtf Reference.filtered.gtf \
--attribute=gene_biotype:protein_coding \


#########Genome indexing 
cellranger mkref --genome=mm10 \
--fasta=${gn}/Refernce.fa \
--genes=${gn}/Refernce.gtf \
--ref-version=3.0.0

################# Gene count  
cellranger count --chemistry=SC3Pv3 \
--id=${name}-REX --project=P180721 \
--transcriptome=${gn} \
--fastqs=${data} \
--sample=${name} \
--localcores=7 \
--expect-cells=500
--localmem=128
Resources

https://lmweber.org/OSTA-book/space-ranger-visium.html
