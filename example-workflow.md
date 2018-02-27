# Example FATSQ Processing Workflow

This is an example of the basic steps I go through to process a pair of FASTQ files for downstream analysis. In this example, I am assuming that we have received a pair of human `.fastq.gz` files from the local Illumina sequencer and that they have been processed in the standard manner.

## Setup

We will be using two test files (`test-1.fastq.gz` and `test-2.fastq.gz`). Hopefully, your files will be much bigger in practice. However, the logic of the code below is the same.

To start, we need to open a terminal and go to the example folder:

~~~bash
cd example
~~~

## Basic Validation & Statistics

The first step is to identify the files we have. As a minimum we need to:

1. Copy the files to a secure location that we know is backed up;
2. Record which files come from which biological samples;
3. Rename the files so that we can easily identify them later (by biological ID);
4. Record the metadata (run information, index, read count, etc) for later use; and
5. Make sure that the sample pairs fit together and are both valid.

After securing, renaming and backing up the files, we can check them for validity and extract the metadata. For this we can use several tools, but I will use:

* [fqtools](https://github.com/alastair-droop/fqtools) to validate the file pair and to get the read counts; and
* [fastq-metadata](https://github.com/alastair-droop/fastq-metadata) to extract run & index data from the file headers.

~~~bash
fqtools count fastq/test-1.fastq.gz fastq/test-2.fastq.gz
fastq-metadata fastq/test-1.fastq.gz fastq/test-2.fastq.gz
~~~

The first command does several things: it checks that both files are valid, it makes sure that they are correctly paired, and it tells us how many reads there are in each file. The second command simply reads out the metadata from the first header of each file. We should record these data for later reference.

## Initial QC

Before we perform any cleaning on the data, it is useful to take a look at the files. We can then compare these QC data to those collected after cleaning to assess how useful the cleaning has been. We use [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) do perform the QC checks:

~~~bash
fastqc --noextract --nogroup --quiet --format=fastq --outDir=./qc fastq/test-1.fastq.gz
fastqc --noextract --nogroup --quiet --format=fastq --outDir=./qc fastq/test-2.fastq.gz
~~~

Once we've generated the FastQC data, we should look through it for any mission-critical errors. Remember that the cleaning we're about to perform will remove many of these errors.

## Data Cleaning

Before we can use these data, we must remove low quality data and high-quality data that does not come from our biological sample. We will use [cutadapt](http://cutadapt.readthedocs.io/en/stable/) for this. Cutadapt will do many things, but in this case we will ask it to:

* Remove low-quality (PHRED < 20) tails from reads;
* Remove adapter sequences from the ends of both reads; and
* Remove read pairs where either pair has a length (after processing) less than 20.

Cutadapt will write a log file that tells us what it has done. It is very useful to save this file.

To run Cutadapt, we need to specify the sequences of the adapters that were used. In this case, we know that the samples have been run using the standard adapters. If you don't know you can get a hint from the FastQC report above, but it is best to ask the sequencing facility that generated your data.

~~~bash
ADAPTER_5=AGATCGGAAGAGCACACGTCTGAACTCCAGTCAC
ADAPTER_3=AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGTAGATCTCGGTGGTCGCCGTATCATT

cutadapt -q20 -m20 -n10 -a$ADAPTER_5 -A$ADAPTER_3 -ofastq/test-trimmed-1.fastq.gz -pfastq/test-trimmed-2.fastq.gz fastq/test-1.fastq.gz fastq/test-2.fastq.gz > qc/test-cutadapt.log
~~~

We can now use [cutadapt-stats](https://github.com/alastair-droop/cutadapt-stats) to pull out the overview statistics from the log file:

~~~bash
cutadapt-stats -c qc/test-cutadapt.log > qc/test-cutadapt.csv
~~~

## Post-Cleaning QC

After running cutadapt, it is good practice to re-run the FastQC step so that we can check if we've done enough cleaning:

~~~bash
fastqc --noextract --nogroup --quiet --format=fastq --outDir=./qc fastq/test-trimmed-1.fastq.gz
fastqc --noextract --nogroup --quiet --format=fastq --outDir=./qc fastq/test-trimmed-2.fastq.gz
~~~
