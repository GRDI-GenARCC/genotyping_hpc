# Preprocessing of genomic data
## Background
Raw sequence data requires pre-processing to be used in genotyping pipelines. Here is an overview of an older version of the GATK "best practices", which we will be using parts of. 
![image](https://github.com/TonyKess/genotyping_hpc/assets/33424749/45bbf013-1eaa-4bea-8d8e-1681c8f4db4a)

This section provides links for running tools we have used for genotyping fisheries species, with an emphasis on handling low and medium coverage data through working with genotype likelihoods and imputation. We run these tools in [SLURM](https://slurm.schedmd.com/documentation.html).

## Directory setup

First, we need to set up the directory structure:
  
```
mkdir sets
mkdir trim
mkdir align
mkdir angsd_out
mkdir angsd_in

```
All scripts launch from the project directory, which is one level above these other directories.

## Passing project information to tools

Next we put together a parameter file, that has information on project names, software locations, and other pieces of info to pass to scripts. It looks like this:

```
#locations
#main directory for project
projdir=<project location>
#sample info
projname=<project name>

#genome file location
genome=<path to target species genome>
#species
species=$(<species name>)

#software
#gatk - for deduplication mostly
gatk=<somepath>/software/<gatk4.something.jar>

#gatk37 - old version that still allows indel realignment, requires openjdk 8, built in gatk37 conda env
#to run this one make sure you source your conda environment
#and then run conda activate gatk37 to get openjdk8 
gatk37=<some path>/software/GenomeAnalysisTK.jar

```

We assume our reads are already hanging out in the aptly named. For this tutorial, we are working with reads in [fastq format](https://knowledge.illumina.com/software/general/software-general-reference_material-list/000002211)
  
```
reads/
```

Go to the raw reads and use split to make set files for parallel SLURM jobs. For low depth sequencing or small genomes, we can make large set files: 

 ``` 
cd reads
  
ls *R1.fastq.gz | \
  sed 's/\_R1.fastq.gz//' > ../sets/<project name>_inds.tsv 
  
 split -l 100 \
  -d \
 ../sets/<project name>\_inds.tsv \
 ../sets/<project name>.set

```

For large per-individual datasets (high genomic coverage, large genome, both), we need to make smaller sets, e.g. split by ~10-15 rather than 100. 
  
## Read trimming
   
Launch the first script in the analysis pipeline, using default trimming parameters in [fastp](https://github.com/OpenGene/fastp) to remove adapter content, and add a sliding window function to remove polyG tails, as suggested by [Lou et al. 2022](https://doi.org/10.1111/1755-0998.13559). This script will be launched to run in parallel on all individuals, 100 at a time for low depth samples.

```
for i in {00..<n set files>}  ;  do sbatch --export=ALL,set=<project name>.set$i,paramfile=WGSparams_<project name>.tsv 01_fastp_parallel.sh ;  done
```
There is a tradeoff for adapter trimming and parallelizing with fastp - it does not become much more efficient at anything beyond [4 threads](https://hpc.nih.gov/training/gatk_tutorial/preproc.html#preproc-single-tools). So we run our each on a single node and allocate 4 threads per task, with 16 separate jobs running. 

Different tasks will have different "optimal settings" (i.e. good enough) for parallelization and multithreading, which I will try to indicate at each step.

## Alignment
     
Next, we need to index the genome for alignment and alignment cleanup. We only need to do this once per genome, like so:

```
sbatch --export=ALL,paramfile=WGSparams_<project name>.tsv 02_genome_index_faidx.sh
```
Now we can get to aligning our reads against the reference genome. This process gains from additional threads (and does weird stuff when you run it with GNU parallel on large read files...) so we take up a whole node per run, with 64 cpus. 
 
```
for i in {00..<n set files>} ;    do sbatch --export=ALL,set=<project name>.set$i,paramfile=WGSparams_<project name>.tsv 03_bwamem2align.sh ;  done
  
```

This process generates [.bam](https://support.illumina.com/help/BS_App_RNASeq_Alignment_OLH_1000000006112/Content/Source/Informatics/BAM-Format.htm) files that are sorted by position. These represent a compressed and space-efficient recording of alingments to a reference genome for each read.
## Refining alignments

Next, we remove sequence duplicates using the genome analysis toolkit. Two things to make sure of in advance. First, we need to make a directory for the temporary files produced by gatk:

```
mkdir gatktemp
```
I make separate ones for each project but you can also just point to a single location across scripts.
The other thing worth doing is optimizing the number of threads utilized per run based on file size. The GATK tools don't usually make use of multithreading, so it's often more efficient to just allocate a single cpu/task and then run those tasks [simultaneously.](https://en.wikipedia.org/wiki/Embarrassingly_parallel) For the high depth samples this means fewer total CPUs/run of the script.We adjust these parameters in the sbatch commands:

```
#SBATCH --cpus-per-task=64
```

and passed to GNU parallel:

```
parallel --jobs 64
```
  
Then, just like other tasks, we can launch the jobs for all samples simulatenously.

```
for i in {00..<n set files>} ;    do sbatch --export=ALL,set=<project name>.set$i,paramfile=WGSparams_<project name>.tsv 04_gatkmarkdup.sh ;  done


At the same time, we are going to need a sequence dictionary for indel realignemt. So we can also produce that now, just once per reference genome.
 
```
sbatch --export=ALL,paramfile=WGSparams_<project name> 05_genome_sequencedicitonary.sh
```

For realignment around insertsions/deletions, we will first need to index the deduplicated alignment files we are working from. Again, we will adjust the number of CPUs to the batch size and run with 1 cpu/sample. 

```
for i in {00..<n set files>} ;    do sbatch --export=ALL,set=<project name>.set$i,paramfile=WGSparams_<project name>.tsv 06_samtools_indexdedup_parallel.sh ;  done
```
Now we can get to actually realigning! Worth nothing this step is from a deprecated version of GATK, but because we are going to do our SNP calling and analysis of genotype likelihoods in ANGSD, we still need to realign. This can be done in a couple of steps, first identifying realignment targets:

```
for i in {00..<n set files>} ;    do sbatch --export=ALL,set=<project name>.set$i,paramfile==WGSparams_<project name> 07_indel_target_parallel.sh ;  done
```
And then updating those alignments:
 
```
for i in for i in {00..<n set files>} ;    do sbatch --export=ALL,set=<project name>.set$i,paramfile=WGSparams_<project name> 08_indel_realign_parallel.sh ;  done
```

This should cover all of the pre-processing steps, and we are ready to move to producing genotype and likelihoods from ANGSD.
