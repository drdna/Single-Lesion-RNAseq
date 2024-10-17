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
singularity run --app bedtools2300 /share/singularity/images/ccs/conda/amd-conda2-centos8.sinf bedtools genomecov -bam SSID116_T16_ALIGN/accepted_hits_sortedRG.bam -bga > T16_genomecov.bed
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
8. When all of the alignments are finished, use sambamba to merge bamfiles for those datasets that were run in duplicate (i.e. on two HiSeq lanes). Duplicate runs were either performed in lanes L001 & L002, or in L003 & L008. Therefore, we can merge those datasets using these commands:
```bash
for file in $(ls alignments/*L001*bam); do sbatch /project/farman_uksr/BASH_SCRIPTS/Sambamba-merge.sh $file ${file/L001/L002}; done
```
then:
```bash
for file in $(ls alignments/*L003*bam); do sbatch /project/farman_uksr/BASH_SCRIPTS/Sambamba-merge.sh $file ${file/L003/L008}; done
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

