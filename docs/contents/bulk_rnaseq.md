# Bulk RNA-seq

Bulk RNA-seq pipeline in bcbio:
- aligns reads with STAR (2pass), or hisat2 vs genome and transcriptome references (human, mouse, custom references);
- quantifies expression counts with salmon, kallisto;
- runs quality control;
- calculates TPM with tximport;
- detects rusions with arriba, pizzly;
- creates a SummarizedExperiment object for downstream analysis in R;
- calls variants with gatk or with vardict;
- supports [spike-in](https://en.wikipedia.org/wiki/RNA_spike-in) calibration

## Workflow

This example processes 6 RNA-seq samples from the [FDA's Sequencing Quality Control project](http://www.fdaseqc.org/).

### 1. Install STAR index (if there is no STAR in the current bcbio installation)
```bash
bcbio_nextgen.py upgrade -u skip --genomes hg38 --aligners star --cores 10
```

### 2. Setup bcbio project
Download input data, create project structure and config files. This will download six samples from the SEQC project, three from the HBRR panel and three from the UHRR panel (100G download). :
```shell
wget https://raw.githubusercontent.com/bcbio/bcbio-nextgen/master/config/examples/rnaseq-seqc-getdata.sh
bash rnaseq-seqc-getdata.sh
```

Step 2 in detail:

#### 2.1 Create input directory and download fastq files
```bash
mkdir -p seqc/input
```

#### 2.2 Download a template yaml file describing RNA-seq analysis
```bash
wget --no-check-certificate https://raw.githubusercontent.com/bcbio/bcbio-nextgen/master/config/examples/rnaseq-seqc.yaml
```
rnaseq-seqc.yaml:
```yaml
# Template for human RNA-seq using Illumina prepared samples
---
details:
  - analysis: RNA-seq
    genome_build: hg38
    algorithm:
      aligner: star
      expression_caller:
      - salmon
      - kallisto
      fusion_caller:
      - arriba
      - pizzly
      quality_format: standard      
      strandedness: auto
      trim_reads: false
      quantify_genome_alignments: true
upload:
  dir: ../final
resources:
  star:
    cores: 10
    memory: 10G
```

#### 2.3 Download fastq files into input dir
```bash
cd seqc/input
for SAMPLE in SRR950078 SRR950079 SRR950080 SRR950081 SRR950082 SRR950083
do 
   wget -c -O ${SAMPLE}_1.fastq.gz ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR950/${SAMPLE}/${SAMPLE}_1.fastq.gz
   wget -c -O ${SAMPLE}_2.fastq.gz ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR950/${SAMPLE}/${SAMPLE}_2.fastq.gz
done
cd ../../
```

#### 2.4 Prepare a sample sheet
```
wget -c --no-check-certificate https://raw.githubusercontent.com/bcbio/bcbio-nextgen/master/config/examples/seqc.csv
```

seqc.csv:
```
samplename,description,panel
SRR950078,UHRR_rep1,UHRR
SRR950079,HBRR_rep1,HBRR
SRR950080,UHRR_rep2,UHRR
SRR950081,HBRR_rep2,HBRR
SRR950082,UHRR_rep3,UHRR
SRR950083,HBRR_rep3,HBRR
```

#### 2.5 Generate yaml config file for analysis
project.yaml = template.yaml x sample_sheet.csv
```
bcbio_nextgen.py -w template rnaseq-seqc.yaml seqc.csv seqc/input/*.gz
```

In the result you should see a folder structure:
```
seqc
|---config
|---input
|---final
|---work
```

`seqc/config/seqc.yaml` is the main config file to run the bcbio project.

### 3. Run the analysis:
Single node:
```bash
cd seqc/work
bcbio_nextgen.py ../config/seqc.yaml -n 8
```

ipython: `sbatch bcbio.sh`:
```bash
#!/bin/bash

# https://slurm.schedmd.com/sbatch.html

#SBATCH --partition=priority        # Partition (queue)
#SBATCH --time=5-00:00:00           # Runtime in D-HH:MM format
#SBATCH --job-name=bulkrnatest      # Job name
#SBATCH -c 1
#SBATCH --mem-per-cpu=10G           # Memory needed per CPU
#SBATCH --output=project_%j.out     # File to which STDOUT will be written, including job ID
#SBATCH --error=project_%j.err      # File to which STDERR will be written, including job ID
#SBATCH --mail-type=NONE            # Type of email notification (BEGIN, END, FAIL, ALL)

bcbio_nextgen.py ../config/seqc.yaml -n 72 -t ipython -s slurm -q medium -r t=0-72:00 -r conmem=20 --timeout 3000 --tag "rna"
```

## Parameters

* `transcript_assembler` If set, will assemble novel genes and transcripts and merge the results into the known annotation. Can have multiple values set in a list. Supports ['cufflinks', 'stringtie'].
* `transcriptome_align` If set to True, will also align reads to just the transcriptome, for use with EBSeq and others.
* `expression_caller` A list of optional expression callers to turn on. Supports ['cufflinks', 'express', 'stringtie', 'dexseq', 'kallisto']. Salmon and count based expression estimation are run by default. Sailfish is deprecated.
* `fusion_caller` A list of optional fusion callers to turn on. Supports [oncofuse, pizzly, ericscript, arriba].
* `variantcaller` Variant calling algorithm to call variants on RNA-seq data. Supports [gatk-haplotype] or [vardict].
* `spikein_fasta` A FASTA file of spike in sequences to quantitate. There are quantitated separately, so should be things you are not expecting to match anywhere to the genome or trancriptome of the species you are working with.
* `quantify_genome_alignments` If set to True, run Salmon quantification using the genome alignments from STAR, when available. If STAR alignments are not available, use Salmon's SA mode with decoys.
* `transcriptome_gtf` A GTF file to use to specify transcripts which will override the bcbio-installed versions. This is used if you have an alternate transcriptome you want to quantitate. You can also use this option to add your own transcripts to a set to quantitate, just append them to your GTF file and sort it.
* `transcriptome_fasta` A fasta file - transcriptome fasta reference, use along with the transcriptome_gtf to override the installed references. How to generate the fasta with [gffread](http://ccb.jhu.edu/software/stringtie/gff.shtml#gffread_ex).
* `strandedness`: `[unstranded (default), firststrand, secondstrand, auto]`. Set if your library is stranded. For dUTP marked libraries, firststrand is correct; for Scriptseq prepared libraries, secondstrand is correct. The wrongly set strandedness could cause up to 90% count loss in salmon. If you are unsure run one sample with `unstranded` and infer strandedness from the data, see more info in the [HBC knowledgebase](https://github.com/hbc/knowledgebase/blob/master/rnaseq/strandedness.md). For example dUTP + [xGEN IDT UMI](https://www.idtdna.com/pages/products/next-generation-sequencing/adapters/xgen-udi-umi-adapters) gives secondstrand, not firststrand. We don't set strandedness in STAR, because it is irreversible, see [this discussion](https://github.com/COMBINE-lab/salmon/issues/590#issuecomment-733417813). Currently we don't set strandedness in kallisto as well. `auto` forces strand auto-detection in salmon with `-l A` option. See why auto strandedness and the properly set known strandedness give slightly different counts [here](https://github.com/COMBINE-lab/salmon/issues/669).
* `disambiguation: mm10`- use for PDX samples. Aligns reads to human and mouse genome. `quantify_genome_alignments: true` with disambiguation does not produce mouse counts in the final counts file becase the refined STAR alignment is used as salmon input, also this combination might produce a lot of messages in the stderr (20M) and fail for some AWS instances. If you need human and mouse counts, use pseudoalignment with `quantify_genome_alignments: false`.

## QC and Basic DE analysis
By default, bcbio runs simple [R scripts](https://github.com/bcbio/bcbio-nextgen/tree/master/bcbio/scripts/R).
To make them work, you need >1 sample, and `category` in the metadata for each sample:
```yaml
algorithm:
  aligner: star
  strandedness: unstranded
metadata:
  category: normal
```
If there is no category in the yaml, the script assigns the same `fake_category` to all samples.

A more comprehensive QC and DE analysis is possible with [bcbiornaseq](https://bioinformatics.sph.harvard.edu/bcbioRNASeq/)
* `bcbiornaseq` A dictionary of key-value pairs to be passed as options to bcbioRNAseq. Currently supports _organism_ as a key and takes the latin name of the genome used (_mus musculus_, _homo sapiens_, etc) and _interesting_groups_ which will be used to color quality control plots:
```yaml
algorithm:
  tools_on:
  - bcbiornaseq
  bcbiornaseq:
    organism: homo sapiens
    interesting_groups: [treatment, genotype, etc, etc]
```

## Output

### RNA-seq

Project directory:
```
├── counts
    ├── tximport-counts.csv -- gene-level counts for DE analysis generated from salmon counts by tximport
    ├── bcbio-se.rds -- SummarizedExperiment object with all counts
bcbio-se.html -- a simple Rmd QC report
├── annotated_combined.counts -- gene counts with symbols from featureCounts (don't use this)
├── bcbio-nextgen-commands.log -- commands run by bcbio
├── bcbio-nextgen.log -- logging information from bcbio run
├── combined.counts -- gene counts with gene IDs from featureCounts (don't use this)
├── metadata.csv -- provided metadata about each sample
├── multiqc
    ├── multiqc_report.html -- multiQC report
├── programs.txt -- program versions of tools run
├── project-summary.yaml -- YAML description of project, with derived
  metadata
└── tx2gene.csv -- transcript to gene mappings for use with tximport
```
Sample directories:
```
S1
├── S1-ready.bam -- coordinate-sorted whole genome alignments
├── S1-ready.counts -- featureCounts counts (don't use this)
├── S1-transcriptome.bam -- alignments to the transcriptome
├── salmon
│   ├── abundance.h5 -- h5 object, usable with sleuth
│   └── quant.sf -- salmon quantifications, usable with tximport
└── STAR
    ├── S1-SJ.bed -- STAR junction file in BED format
    └── S1-SJ.tab -- STAR junction file in tabular format
```
bcbioRNASeq directory:
```
bcbioRNASeq/
├── data
│ ├── bcb.rds -- bcbioRNASeq object with gene-level data
├── data-transcript
│ ├── bcb.rds -- bcbioRNASeq object with transcript-level data
├── quality_control.html -- quality control report
├── quality_control.Rmd -- RMarkdown that generated quality control report
├── results/2021-12-18
│ ├── gene -- gene level information
│ │ └── counts
│ │ ├── counts.csv.gz -- count matrix from tximport, suitable for count-based analyses
│ │ └── metadata.csv.gz -- metadata and quality control data for samples
│ ├── quality_control/bcb/assays
│ │ └── tpm.csv -- TPM from tximport, use for visualization
│ ├── transcript/counts -- transcript level information
│ │ └── counts.csv.gz -- transcript level count matrix, suitable for count-based analyses needed transcript-level data
│ │ └── metadata.csv.gz -- metadata and quality control for samples
```
Workflow for analysis:

For gene-level analyses, we recommend loading the gene-level counts.csv.gz and the metadata.csv.gz and using [DESeq2](https://bioconductor.org/packages/release/bioc/html/DESeq2.html) to do the analysis. For a more in-depth walkthrough of how to use DESeq2, refer to our [DGE_workshop](https://hbctraining.github.io/DGE_workshop_salmon/schedule/).

For transcript-level analyses, we recommend using [sleuth](https://seqcluster.readthedocs.io/mirna_annotation.html) with the bootstrap samples. You can load the abundance.h5 files from Salmon, or if you set `kallisto` as an expression caller, use the abundance.h5 files from that.

Another great alternative is to use the Salmon quantification to look at differential transcript usage (DTU) instead of differential transcript expression (DTE). The idea behind DTU is you are looking for transcripts of genes that have been flipped from one isoform to another. The [Swimming downstream](https://www.bioconductor.org/packages/devel/workflows/vignettes/rnaseqDTU/inst/doc/rnaseqDTU.html#salmon-quantification) tutorial has a nice walkthrough of how to do that.

## Steps

Step are outlined [here](https://www.michaelchimenti.com/2019/03/bcbio-rna-seq-under-the-hood/)

## Description

RNA-seq pipeline includes steps for quality control, adapter trimming, alignment, variant calling, transcriptome reconstruction and post-alignment quantitation at the level of the gene and isoform.

We recommend using the STAR aligner for all genomes. Use Tophat2 only if you do not have enough RAM available to run STAR (about 30 GB).

Our current recommendation is to run adapter trimming only if using the Tophat2 aligner. Adapter trimming is very slow, and aligners that soft clip the ends of reads such as STAR and hisat2, or algorithms using pseudoalignments like Salmon handle contaminant sequences at the ends properly. This makes trimming unnecessary. Tophat2 does not perform soft clipping so if using Tophat2, trimming must still be done.

Salmon, which is an extremely fast alignment-free method of quantitation, is run for all experiments. Salmon can accurately quantitate the expression of genes, even ones which are hard to quantitate with other methods (see [this paper](https://doi.org/10.1186/s13059-015-0734-x) for example for Sailfish, which performs similarly to Salmon). Salmon can also quantitate at the transcript level which can help gene-level analyses
(see [this paper](https://doi.org/10.12688/f1000research.7563.1) for example). We recommend using the Salmon quantitation rather than the counts from featureCounts to perform downstream quantification.

Still, we had at least two projects with initally low % mapped (STAR) at 50-70% which greatly benefited from trimming both in terms of % mapped and N mapped reads.
We trimmed with 
bcbio.yaml:
```yaml
algorithm:
  trim_reads: read_through 
  adapters: illumina
```
or with [fastp](https://github.com/OpenGene/fastp)
```bash
#!/bin/bash
fastp -I {input.R1]} -I {input.R2} -o {output.R1_trim_fastq} -O {output.R2_trim_fastq} -h {output.fastp_html} -g -x -5 -3

# Additional flag definitions:
# -g trims polyG, common artifact in Illumina 2-dye sequencing chemistry (MiniSeq, NextSeq, Novaseq)
# -x trims polyX, any other homopolymeric stretch
# -5 sliding window 5’ quality trimming
# -3 sliding window 3’ quality trimming
```

Although we do not recommend using the featureCounts based counts, the alignments are still useful because they give you many more quality metrics than the quasi-alignments from Salmon.

After a bcbio RNA-seq run there will be in the `upload` directory a directory for each sample which contains a BAM file of the aligned and unaligned reads, a `salmon` directory with the output of Salmon, including TPM values, and a `qc` directory with plots from FastQC and qualimap.

In addition to directories for each sample, in the `upload` directory there is a project directory which contains a YAML file describing some summary statistics for each sample and some provenance data about the bcbio run. In that directory is also a `combined.counts` file with the featureCounts derived counts per cell.
