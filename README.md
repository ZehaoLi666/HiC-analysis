# Tools and scripts you will need for general HiC analysis:
*	HiC-Pro
*	Selfish
*	Juicebox/juicertools
*	Bowtie2
*	Samtools
*	hic_CPM.py
*	plot_hicpro.py
*	plot_selfish.py
*	max_reads.py
* create_sizes.py

HiC-Pro – Although there are many available Hi-C processing pipelines, most use similar inputs (fastq files, chromosome sizes and restriction enzyme map), alignment tools (BWA or bowtie2), post-processing tools (Samtools) and normalization methods (KR or ICED). The outputs of these pipelines can often be converted to utilize other tools if necessary.

# HiC-Pro
## 1.	Load HiC-Pro if using the cluster.
```
module load hic-pro
```
## 2.	Sort paired-end fastq files into folders by sample.
```
hic_dir/input_dir/sample1/sample1_R1.fastq
hic_dir/input_dir/sample1/sample1_R2.fastq
hic_dir/input_dir/sample2/sample2_R1.fastq
hic_dir/input_dir/sample2/sample2_R2.fastq
```
## 3.	Generate file containing chromosome sizes.
```
python3 create_sizes.py genome_files_dir/organism_genome.fasta
```
## 4.	Generate bowtie2 indices.
```
bowtie2-build genome_files_dir/organism_genome.fasta genome_files_dir/organism_genome
```
## 5.	Use HiC-Pro utility to generate list of restriction fragments.

The script will change depending on restriction enzyme used to generate HiC library.
```
python3 HICPRO_PATH/bin/utils/digest_genome.py -r ^GATC -o genome_files_dir/organism_genome_MboI.bed genome_files_dir/organism_genome.fasta
```
## 6.	Modify config_hicpro.txt. The following are the recommended parameters to modify:
```
N_CPU = number of CPU cores  #  dependent on processor or cluster allocation
SORT_RAM = RAM allocation    #  dependent on available RAM or cluster allocation
MIN_MAPQ = quality cutoff for read mapping   # a value of 30 ensures that any ambiguous reads are removed
BOWTIE2_IDX_PATH = path to bowtie2 indices generated from parent genome
REFERENCE_GENOME = base name of reference genome
GENOME_SIZE = path to tab-delimited file containing chromosome sizes
GENOME_FRAGMENT = digested genome file from step 3
LIGATION_SITE = ligation motif for restriction enzyme   # usually duplicate of cut site
MboI cut site = GATC
ligation motif = GATCGATC
BIN_SIZE = binning resolution      # dependent on number of fragments within digested genome, median length of digested fragments, number of aligned reads and several other factors.
#With a P. falciparum Hi-C library 10kb resolution is used if we have 10 million aligned and paired reads
#The number of aligned reads is unknown until after running the pipeline; however, you can generally assume that 25-40% of sequenced reads will be available after processing a high-quality library.
```
## 7.	Run HiC-Pro.
```
HiC-Pro -c hic_dir/config-hicpro.txt -i hic_dir/input_dir -o hic_dir/output_dir
```
## 8.	Several output files will be generated. 
The following are a few of the most important files needed for downstream analysis, all other files can be deleted to save space:
```
hic_dir/output_dir/hic_results/data/sample/sample.allValidPairs = all paired and aligned reads after filtering
hic_dir/output_dir/hic_results/matrix/sample/raw/10kb/sample.matrix = un-normalized read count matrix
hic_dir/output_dir/hic_results/matrix/sample/raw/10kb/sample_abs.bed = numbered fragments based on binning resolution
hic_dir/output_dir/hic_results/matrix/sample/iced/10kb/sample_iced.matrix = ICED normalized read count matrix
hic_dir/output_dir/hic_results/matrix/sample/iced/10kb/sample_iced.matrix.biases = per-bin biases within normalized data
```

# Generate heatmaps 

Convert the .allValidPairs file to .hic format for use in Juicebox visualization tool or generate the heatmaps manually using the normalized .matrix output.
## 1.	Method 1 – visualize with Juicebox:
```
Download juicertools .jar file - https://github.com/aidenlab/juicer/wiki/Download
Convert .allValidPairs file to .hic.
sh HICPRO_PATH/bin/utils/hicpro2juicebox.sh
-i hic_dir/output_dir/hic_results/data/sample/sample.allValidPairs
-g genome_files_dir/organism_genome.sizes
-j juicebox_dir/juicer_tools.jar
-r genome_files_dir/organism_genome_MboI.bed
Download juicebox  and import .hic file - https://github.com/aidenlab/Juicebox
```
## 2.	Method 2 – manually generate heatmaps:
Calculate max values for each chromosome for each sample so that samples could be directly compared.
```
python3 max_reads.py genome_files_dir/organism_genome.sizes
Paste results into list on line 106 of plot_hicpro.py.
Plot heatmaps. The following are the recommended arguments, but all arguments can be shown using the -h flag.
python3 plot_hicpro.py -ch genome_files_dir/organism_genome_MboI.bed 
-m hic_dir/output_dir/hic_results/matrix/sample/iced/10kb/sample_iced.matrix -n
```
# Selfish 
Like the Hi-C pipeline tools, there are numerous tools available for differential interaction analysis. These tools utilize a wider variety of algorithms to identify differential interactions and some may not work with all organisms due to genome specific features.
## 1.	Counts per-million normalize the ICED normalized matrices.
```
python3 hic_CPM.py hic_dir/output_dir/hic_results/matrix/sample/iced/10kb/sample_iced.matrix
```
## 2.	Download and run [Selfish](https://github.com/ay-lab/selfish)

Sample 1 in this example should be the conditional and sample 2 is the control.
```	
selfish -m1 hic_dir/output_dir/hic_results/matrix/sample1/iced/10kb/sample1_iced_cpm.matrix
-m2 hic_dir/output_dir/hic_results/matrix/sample2/iced/10kb/sample2_iced_cpm.matrix
-bed1 hic_dir/output_dir/hic_results/matrix/sample1/raw/10kb/sample1_abs.bed
-bed2 hic_dir/output_dir/hic_results/matrix/sample2/raw/10kb/sample2_abs.bed
-t 0.05 -r 10000 -ch chr1
-o hic_dir/sample1_sample2_chr1_diff.txt
```
## 3.	Merge outputs for all chromosomes.
```	
awk '(NR == 1) || (FNR > 1)' *_diff.txt  > sample1_sample2_diff.txt
```
## 4.	Plot differential heatmaps.

The plotting script hasn’t been updated to include a help flag or verbosity. The three arguments are {1} sample name, {2} chromosome sizes file and {3} merged differential output from step 3.
```
python3 plot_selfish.py sample1_sample2 genome_files_dir/organism_genome.sizes sample1_sample2_diff.txt
```
