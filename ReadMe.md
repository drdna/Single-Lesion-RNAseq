# Map RNAseq reads to the B71 reference genome
1. Create a directory named RNAseq in the /scratch/ directory on MCC:
```bash
mkdir /scratch/rjwh222/RNAseq
```
2. Copy the RNAseq reads from the farman_uksr pscratch directory into the RNAseq directory:
```bash
cp /pscratch/farman_uksr/*gz /scratch/rjwh222/RNAseq/
```
3. Change into the RNAseq directory:
```bash
cd /scratch/rjwh222/RNAseq
```
4. Align the RNAseq reads to the B71 reference genome:
```bash
for file in $(ls *gz); do echo "Working on dataset" file; sbatch /project/farman_uksr/BASH_SCRIPTS/HISAT2.sh $file; done
```
5. Merge bamfiles for datasets that were run in duplicate (two HiSeq lanes). Duplicate runs were either performed in lanes L001 & L002, or in L003 & L008. Therefore, we can merge those datasets accordingly:
```bash
for file in $(ls *L001*gz); do sbatch /project/farman_uksr/BASH_SCRIPTS/MergeBams.sh L001 L002; done
for file in $(ls *L003*gz); do sbatch /project/farman_uksr/BASH_SCRIPTS/MergeBams.sh L003 L008; done
```
 
