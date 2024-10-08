# Map RNAseq reads to the B71 reference genome
1. Create a directory named RNAseq in the /scratch/ directory on MCC:
```bash
mkdir /scratch/rjwh222/RNAseq
```
2. Copy the RNAseq reads from the farman_uksr pscratch directory into the RNAseq directory:
```bash
cp /pscratch/farman_uksr/SINGLE_LESIONs/FASTQ/*gz /scratch/rjwh222/RNAseq/
```
3. Change into the RNAseq directory and create a directory to receive the alignments:
```bash
cd /scratch/rjwh222/RNAseq
mkdir alignments
```
4. Check that the reads were copied correctly and can be accessed:
```bash
for file in $(ls *gz); do echo $file; done
```
5. Check that the hisat2.sh script can be accessed:
```bash
cat /project/farman_uksr/BASH_SCRIPTS/hisat2.sh
```
6. Use HISAT2 to align the RNAseq reads to the B71 reference genome:
```bash
for file in $(ls *gz); do echo "Working on dataset" $file; sbatch /project/farman_uksr/BASH_SCRIPTS/hisat2.sh $file; done
```
7. Check progress of scripts periodically to make sure that all have been completed. Running scripts can be listed using the following command. When all are completed the list will be empty.
```bash
squeue | grep rjwh222
```
8. Check that the relevant bam files were created:
```bash
ls alignments
```
9. When all of the alignments are finished, use sambamba to merge bamfiles for those datasets that were run in duplicate (i.e. on two HiSeq lanes). Duplicate runs were either performed in lanes L001 & L002, or in L003 & L008. Therefore, we can merge those datasets using these commands:
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
infiles=$(ls alignments/*bam -1 | tr '\n' ' '); sbatch /project/farman_uksr/BASH_SCRIPTS/HTSeq.sh $infiles B71GeneCounts.txt
```
3. Check B71GeneCounts.txt output file after completion:
```bash
cat B71GeneCounts.txt
```

