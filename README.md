Hands on materials for Nanopore data analysis: from basecalling to structural variant detection

## 0. Preliminary
To use Shirokanedai HPC, we have the following two modes:
* Interactive mode
```
qlogin -l os7,s_vmem=16G,mem_req=16G
# if you need to use GPU
qlogin -l os7,s_vmem=16G,mem_req=16G,v100=1 
```

* Job submission mode. Write a shell script with the following head lines in your SCRIPT-NAME.sh file.
```
#!/bin/sh
#$ -S /bin/sh
#$ -cwd
#$ -o <FOLD-To-SAVE-OUTPUT>
#$ -e <FOLD-To-SAVE-ERROR-OUTPUT>

pwd
hostname
date
echo arg1=$1
sleep 10

YOUR-COMMAND-PIPELINE-SCRIPTS
```
Submit your script file:
```
qsub -l os7,s_vmem=16G,mem_req=16G SCRIPT-NAME.sh
```


## 1.  Nanopore data analysis enviroment setup
1.1 Get into the interactive mode
```
qlogin -l os7,s_vmem=16G,mem_req=16G
```

1.2. Build singularity images with pre-prepared docker file
```
singularity build --sandbox <YOUR-IMAGE-NAME> docker://yaozhong/nanopore_analysis
```

1.3. Get into the singularity image
```
singularity shell --writable <YOUR-IMAGE-NAME>
# if GPU is intended to use, please first specify qlogin v100 option, and using the --nv option like
singularity shell --nv --writable <YOUR-IMAGE-NAME>
```

1.4. Example data and configuration files
We provide 100 reads of raw fast5 signals in the data fold.
Configurations used in guppy is also places there.


## 2. Base-calling

```
guppy_basecaller -i data/fast5/lambda/ -s output/basecalling/lambda -c /opt/ont/guppy/data/dna_r9.4.1_450bps_fast.cfg 

```
For 4,000 lambda fast5 reads, it takes real 35m49.934s.

## 3. Assembly
3.1 Read-read mapping
Find overlaps between long reads
```
# k: k-mer size, w: minmize winodw length 
minimap2 -x ava-ont -k15 -w5 marge_${i}_par.fastq marge_${i}_par.fastq > reads_{i}.paf
# generate contigs
miniasm  -f merge_{i}_par.fastq reads_{i}.paf > raw_contigs_{i}.gfa
```

3.2 Contig Generation
- Miniasm is used to generate contigs
- The mapping information between all reads (*.paf) is used to generate consensus contigs.
```
miniasm -f $READS_fastq $ASM_Folder/$ASM_Prefix.paf > $ASM_Folder/$ASM_Prefix.gfa

# extract contig fastq file
awk '$1 ~/S/ {print ">"$2"\n"$3}' $ASM_Folder/$ASM_Prefix.gfa > $ASM_prefix.contigs.fasta
```

3.3 Read-Contig mapping
Find the overlaps between contigs
```
minimap2 $ASM_Folder/$ASM_Prefix.contigs.fasta $READS > $ASM_Folder/$ASM_Prefix.contigs.paf
```

3.4 Polishing
Repeat the following steps 10 times to get more accurate long contigs
```
minimap2 contigs.fasta $READS > contigs.paf
racon $READS configs.paf contigs.fasta > contigs.fasta
```

```
## 10 rounds of  polishing
for j in {0..10}
do
minimap2  ${out_file}consensus_${i}_${j}.fasta ${out_file}merge_${i}_par.fastq > ${out_file}map${i}_${j}.paf
racon ${out_file}merge_${i}_par.fastq ${out_file}map${i}_${j}.paf  ${out_file}consensus_${i}_${j}.fasta >  ${out_file}consensus_${i}_$((j+1)).fasta
done
```

## 4. Structural variant detection





