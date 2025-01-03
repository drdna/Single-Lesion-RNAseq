# Map T16 reads to the SSID116 reference genome
1. Create a bowtie index of the SSID116 genome:
```bash
singularity run --app bowtie235sralinux /share/singularity/images/ccs/conda/amd-conda4-centos8.sinf bowtie2-build /project/farman_uksr/SSID116_index/SSID116.fasta /project/farman_uksr/SSID116_index/SSID116
```
2. Align reads to the reference:
```bash
sbatch /project/farman_uksr/BASH_SCRIPTS/BWT2-GATK.sh SSID116 /project/farman_uksr T16
```
3. Assess genome coverage:
```bash
singularity run --app bedtools2300 /share/singularity/images/ccs/conda/amd-conda2-centos8.sinf bedtools genomecov -ibam SSID116_T16_ALIGN/accepted_hits_sortedRG.bam -bga > T16_genomecov.bed
```
4. Extract genome regions with zero coverage spanning 2 kb or more:
```bash
awk '$4 == 0 && $3 - $2 > 2000' T16_genomecov.bed > T16_no_coverage_regions.bed
```

# Map RNAseq reads to the B71 reference genome
1. Create a directory named RNAseq in the /scratch/ directory on MCC:
```bash
mkdir /scratch/rjwh222/RNAseq
```
2. Copy the RNAseq reads from the farman_uksr pscratch directory into the RNAseq directory:
```bash
cp /pscratch/farman_uksr/SINGLE_LESIONs/FASTQ/*gz /scratch/rjwh222/RNAseq/
```
3. Check that the reads were copied correctly and can be accessed:
```bash
/scratch/rjwh222/RNAseq/*gz
```
4. Check that the hisat2.sh script can be accessed:
```bash
cat /project/farman_uksr/BASH_SCRIPTS/hisat2.sh
```
5. Use HISAT2 to align the RNAseq reads to the B71 reference genome:
```bash
for file in $(ls *gz); do echo "Working on dataset" $file; sbatch /project/farman_uksr/BASH_SCRIPTS/hisat2.sh  $file /project/farman_uksr/B71v2sh_index/B71v2sh; done
```
#### Note: the available indexes are in the following directories in the /project/farman_uksr directory: B71v2sh_index/B71v2sh; Osativa_index/Osativa; Pi9-BAC_index/Pi9_BAC (note hyphen and underscores); and AvrGenes_index_AvrGenes
The script will generate an output directory inside the current working directory (eg. B71v2sh_alignments). Inside this directory, it will name the alignments and summary files using the formats: DatasetID_ReferenceID_accepted_hits.bam and DatasetID_ReferenceID_summary.txt
6. Check progress of scripts periodically to make sure that all have been completed. Running scripts can be listed using the following command. When all are completed the list will be empty.
```bash
squeue | grep rjwh222
```
7. Check that the relevant bam files were created:
```bash
ls alignments
```
8. If needed, filter bamfiles to remove entries for reads that did not align to the relevant references:
```bash
for f in `ls alignments/*bam`; do sbatch NoUnaligned.sh $f; done
```
9. After checking for successful completion of the NoUnaligned.sh script (either by long listing the directory, or using samtools view to view teh bam files), delete the original bamfiles:
```bash
ls -l alignments/
rm alignments/*hits.bam
```
10. When all of the alignments are finished, use sambamba to merge bamfiles for those datasets that were run in duplicate (i.e. on two HiSeq lanes). Duplicate runs were either performed in lanes L001 & L002, or in L003 & L008. Therefore, we can merge those datasets using these commands:
```bash
for file in $(ls alignments/*L001*_noUnal.bam); do sbatch /project/farman_uksr/BASH_SCRIPTS/sambamba-merge.sh $file ${file/L001/L002}; done
```
then:
```bash
for file in $(ls alignments/*L003*bam); do sbatch /project/farman_uksr/BASH_SCRIPTS/sambamba-merge.sh $file ${file/L003/L008}; done
```
# Map reads to transcripts
1. Create a directory named transcripts:
```bash
mkdir transcripts
```
2. Run stringtie to map reads to features on gff file:
```bash
for f in `ls AvrGenes_alignments/*bam`; do sbatch stringtie.sh $f /project/farman_uksr/AvrGenes.gff; done
```
# Merge transcripts from different assemblies
1. Create a list of paths to the newly created gtf files:
```bash
ls transcripts/*gtf > transcript_assemblies.txt
```
2. Run stringtie-merge on resulting gtf files:
```bash
sbatch stringtie-merge.sh path/to/AvrGenes.gff path/to/transcript_assemblies.txt merged-transcripts.gtf
```
The merge gtf file will have refined coordinates for the various transcripts.
# Merge bamfiles from different RNAseq samples
```bash
infiles=$(ls SampleCultivarDir/*bam | tr '\n' ' '); singularity run --app sambamba082 /share/singularity/images/ccs/conda/amd-conda5-rocky8.sinf sambamba merge mergedfiles.bam $infiles
```
# Count reads mapping to genes that have been annotated in the B71 reference genome
1. Make sure you are in the RNAseq directory
2. Provide HTSeq a list of alignment files and a GFF file containing the gene annotations:
```bash
infiles=$(ls B71v2sh_alignments/*bam -1 | tr '\n' ' '); sbatch /project/farman_uksr/BASH_SCRIPTS/HTSeq.sh $infiles /project/farman_uksr/B71Ref2.gff B71GeneCounts.txt
```
#### Note: the relevant gff files are inside the /project/farman_uksr directory: B71Ref2.gff OsativaRefSeq.gff Pi9_BAC.gff AvrGenes.gff (this last gff has not yet been created)
3. Check B71GeneCounts.txt output file after completion:
```bash
cat B71GeneCounts.txt
```
# Combine read counts for exons from same gene
The counts file may look something like this:
| Gene                     | Value 1 | Value 2 |
|--------------------------|---------|---------|
| B71d_011844-T1.exon1    | 3       | 2       |
| B71d_011844-T1.exon2    | 9       | 5       |
| B71d_011845-T1.exon2    | 1       | 1       |
| B71d_011845-T1.exon3    | 1       | 5       |
| B71d_011845-T1.exon4    | 2       | 2       |
| B71d_011846-T1.exon1    | 65      | 25      |
| B71d_011846-T1.exon2    | 56      | 38      |
| B71d_011847-T1.exon1    | 2       | 0       |
| B71d_011855-T1.exon2    | 3       | 4       |
| B71d_011855-T1.exon3    | 3       | 1       |
| B71d_011864-T1.exon1    | 6       | 1       |
| B71d_011864-T1.exon2    | 11      | 6       |
| B71d_011864-T1.exon3    | 8       | 4       |
| B71d_011871-T1.exon1    | 2       | 0       |
| B71d_011875-T1.exon1    | 1       | 1       |
| B71d_011878-T1.exon1    | 1       | 1       |
| B71d_011884-T1.exon1    | 4       | 2       |

To sum counts by gene (and not exon), we need to combine dataframe rows. One way to do this is to use gsub to remove the suffix from the exon entries and store this in a new column name "prefix." Then one can sum rows that have the same value in the prefix column.
```bash
library(dplyr)
df <- read.table("~/B71counts.txt", row.names=1, header=TRUE)
df$prefix = gsub(".exon.*", "", rownames(df))
dfcollated <- df %>% group_by(prefix) %>% summarise(across(everything(), sum, na.rm = TRUE))
```
# Build lists of genes to query
Refer to slides from IRBC07 meeting and find names of housekeeping genes. Use grep to search for the corresponding B71d_0XXXXX identifiers in the B71Ref2.gff3 file
# Filter dataframe to retrieve rows for the genes of interest
```bash
gene_list <- c("B71d_008098", "B71d_008099")
pattern_list <- paste0("\\b(", paste(gene_list, collapse = "|"), ")\\b")
filtered_df <- dfcollated[grepl(pattern_list, dfcollated$prefix), ]
```
# Retrieve list of secreted proteins from B71 gff file
```bash
grep -i secrete B71Ref2.gff | grep -v TransMembrane | awk -F 'ID=|;' 'NF {print $2}' | sort -u | paste -sd ',' - > SecretedProteins.txt
```
This list can be used to grab secreted proteins counts from the counts file by reading it into a new gene_list object:
```bash
gene_list <- read.csv(Secreted_proteins.txt)
```



