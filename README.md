# CutRunTools

This package contains the pipeline for conducting a CUT&RUN analysis.
The pipeline comprises of read trimming, alignment steps, motif finding steps, and finally the motif footprinting step. 

To get started, please see [INSTALL.md](INSTALL.md) about how to set up the pipeline.

Once the package is installed, please see [USAGE.md](USAGE.md) about usage. Basic usage is provided below.


## Basic Usage

CutRunTools requires a JSON configuration file which specifies all that is needed to run an analysis. 
A sample configuration file is below. 

```json
{
	"Rscriptbin": "/n/app/R/3.3.3/bin",
	"pythonbin": "/n/app/python/2.7.12/bin",
	"trimmomaticbin": "/n/app/trimmomatic/0.36/bin",
	"trimmomaticjarfile": "trimmomatic-0.36.jar",
	"bowtie2bin": "/n/app/bowtie2/2.2.9/bin",
	"samtoolsbin": "/n/app/samtools/1.3.1/bin",
	"adapterpath": "/home/qz64/cutrun_pipeline/adapters", 
	"picardbin": "/n/app/picard/2.8.0/bin",
	"picardjarfile": "picard-2.8.0.jar",
	"macs2bin": "/n/app/macs2/2.1.1.20160309/bin",
	"macs2pythonlib": "/n/app/macs2/2.1.1.20160309/lib/python2.7/site-packages",
	"kseqbin": "/home/qz64/cutrun_pipeline", 
	"memebin": "/n/app/meme/4.12.0/bin", 
	"bedopsbin": "/n/app/bedops/2.4.30", 
	"bedtoolsbin": "/n/app/bedtools/2.26.0/bin",
	"makecutmatrixbin": "/home/qz64/.local/bin",
	"bt2idx": "/n/groups/shared_databases/bowtie2_indexes",
	"genome_sequence": "/home/qz64/chrom.hg19/hg19.fa",
	"extratoolsbin": "/home/qz64/cutrun_pipeline", 
	"extrasettings": "/home/qz64/cutrun_pipeline", 
	"input/output": {
		"fastq_directory": "/n/scratch2/qz64/Nan_18_aug23/Nan_run_19",
		"workdir": "/n/scratch2/qz64/workdir",
		"fastq_sequence_length": 42,
		"organism_build": "hg19"
	},
	"motif_finding": {
		"num_bp_from_summit": 150,
		"num_peaks": 5000,
		"total_peaks": 15000,
		"motif_scanning_pval": 0.0005,
		"num_motifs": 20
	},
	"cluster": {
		"email": "johndoe@gmail.com",
		"step_alignment": {
			"queue": "short",
			"memory": 32000,
			"time_limit": "0-12:00"
		},
		"step_process_bam": {
			"queue": "short",
			"memory": 32000,
			"time_limit": "0-12:00"
		},
		"step_motif_find": {
			"queue": "short",
			"memory": 32000,
			"time_limit": "0-12:00"
		},
		"step_footprinting": {
			"queue": "short",
			"memory": 32000,
			"time_limit": "0-12:00"
		}
	}
}
```

Note the important settings above are: **adapterpath** (line 8), **bt2idx** (line 17); **genome_sequence** (line 18);
section **input/output**: **fastq_directory** (line 22), **workdir** (line 23), **fastq_sequence_length** (line 24),
**organism_build** (line 25); section **cluster**: **email** (line 35).

*  `fastq_directory` is the directory containing paired-end CUT&RUN sequences (with _R1_001.fastq.gz and _R2_001.fastq.gz suffix). 
*  `organism_build` is one of supported genome assemblies: hg38, hg19, mm10, and mm9. 
*  `adapterpath` contains Illumina Truseq3-PE adapter sequences (we provide them). 
*  `genome_sequence` is the whole-genome **masked** sequence which matches with the appropriate organism build.

### Create job submission scripts
```bash
./create_scripts.py config.json
```
This creates a set of Slurm job-submission scripts based on the configuration file above (named config.json) in the `workdir` directory. The scripts can be directly, easily executed.
```bash
./validate.py config.json
```
This script checks that your configuration file is correct and all paths are correct.

### Four-step process

With the scripts created, we can next perform the analysis.

Step 1. **Read trimming, alignment.** We suppose the `workdir` is defined as `/n/scratch2/qz64/workdir`
```bash
cd /n/scratch2/qz64/workdir
sbatch ./integrated.sh CR_BCL11A_W9_r1_S17_R1_001.fastq.gz
```
The parameter is the fastq file. Even though we specify the _R1_001.fastq.gz, CutRunTools actually checks that both forward and reverse fastq files are present. Always use the _R1_001 of the pair as parameter of this command.

Step 2. **BAM processing, peak calling.** It marks duplicates in bam files, and filter fragments by size.
```bash
cd aligned.aug10
sbatch ./integrated.step2.sh CR_BCL11A_W9_r1_S17_aligned_reads.bam
```

Step 3. **Motif finding.** CutRunTools uses MEME-chip for de novo motif finding on sequences surrounding the peak summits.
```bash
cd ../macs2.narrow.aug18
sbatch ./integrate.motif.find.sh CR_BCL11A_W9_r1_S17_aligned_reads_peaks.narrowPeak
```
By default, CutRunTools keeps duplicate fragments. If instead users wish to use deduplicate version, 
```bash
cd ../macs2.narrow.aug18.dedup
sbatch ./integrate.motif.find.sh CR_BCL11A_W9_r1_S17_aligned_reads_peaks.narrowPeak
```

Step 4. **Motif footprinting.**
```bash
cd ../macs2.narrow.aug18
sbatch ./integrate.footprinting.sh CR_BCL11A_W9_r1_S17_aligned_reads_peaks.narrowPeak
```
Beautiful footprinting figures will be located in the directory `fimo.result`. Footprinting figures are created for every motif found by MEME-chip, but only the right motif (associated with TF) will have a proper looking shape. Users can scan through all the motifs' footprints.


### Outputs

Please see [USAGE.md](USAGE.md) for more details for the output files.

Briefly, CUT&RUNTools will generate the following directory structure in the output folder.

```
+ macs2.narrow.aug18
  + random.10000
    + MEME_GATA1_HDP2_30min_S13_aligned_reads_shuf
      ++ summary.tsv
  + fimo.result
    + GATA1_HDP2_30min_S13_aligned_reads_peaks
      + fimo2.DREME-10.CCWATCAG
        ++ fimo.png
        ++ fimo.logratio.txt
        ++ fimo.bed
      + fimo2.DREME-11.CTCCWCCC
        ++ fimo.png
        ++ fimo.logratio.txt
        ++ fimo.bed
      + fimo2.DREME-12.CACGTG
        ++ fimo.png
        ++ fimo.logratio.txt
        ++ fimo.bed
      + fimo2.DREME-13.CAKCTGB
        ++ fimo.png
        ++ fimo.logratio.txt
        ++ fimo.bed
      + fimo2.DREME-14.CWGATA
        ++ fimo.png
        ++ fimo.logratio.txt
        ++ fimo.bed
      + fimo2.DREME-1.HGATAA
        ++ fimo.png
        ++ fimo.logratio.txt
        ++ fimo.bed
      + fimo2.DREME-20.CCCTYCC
        ++ fimo.png
        ++ fimo.logratio.txt
        ++ fimo.bed
```

Files are denoted by "++", and directories are denoted by "+".
Here the sample name is called GATA1_HDP2_30min_S13.
The summary.tsv lists all the motif found by motif searching analysis. The files within "fimo2..." folder shows the motif footprinting figure (.png), the motif sites for the motif (.bed), and the logratio binding scores at these sites (.logratio.txt).
