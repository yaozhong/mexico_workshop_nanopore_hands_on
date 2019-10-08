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
We provided
* 4000 Lambda phage reads of raw fast5 signals in the fold of
```
~/nanopore/data/fast5/lambda
```
* 1000 reads from human chr11 in the fold of
```
~/nanopore/data/fast5/hs_chr20_1000
```

Reference files are listed in
```
~/nanopore/data/reference
```

## 2. Base-calling

```
guppy_basecaller -i data/fast5/lambda -s output/basecall/lambda -c /opt/ont/guppy/data/dna_r9.4.1_450bps_fast.cfg --cpu_threads_per_caller 24 
```
For 4,000 lambda fast5 reads, it takes real 2m 39.700s.


## 3. Assembly
3.1 Read-read mapping
Find overlaps between long reads
```
# k: k-mer size, w: minmize winodw length 
minimap2 -x ava-ont -k15 -w5 YOUR-READS.fastq  YOUR-READS.fastq > YOUR-READS-OVERLAP.paf
```

3.2 Contig Generation
- Miniasm is used to generate contigs
- The mapping information between all reads (*.paf) is used to generate consensus contigs.
```
# generate contigs
miniasm  -f YOUR-READS.fastq  YOUR-READS-OVERLAP.paf > YOUR-CONTIGS.gfa

# extract contig fastq file
awk '$1 ~/S/ {print ">"$2"\n"$3}' YOUR-CONTIGS.gfa > YOUR-CONTIGS.fasta
```

3.3 Read-Contig mapping
Find the overlaps between read and contigs
(note no specific option is needed)
```
minimap2 $READS contigs.fasta > read-contig.paf
```

3.4 Concensus correction (Polishing)
- Correct contig erros 
- Repeat the following steps 10 times to get more accurate long contigs
```
minimap2  contigs.fasta YOUR-READS.fastq  > read-contig.paf
racon YOUR-READS.fastq read-contig.paf contigs.fasta > contigs.fasta
```

```
## 10 rounds of polishing

for i in `seq 1 10`
do
	minimap2  contigs_$((i-1)).fasta YOUR-READS.fastq  > read-contig_$((i-1)).paf
	racon YOUR-READS.fastq read-contig_$((i-1)).paf contigs_$((i-1)).fasta > contigs_$((i)).fasta
done
```

Now you get the assembly of genome from your reads.


## 4. Structural variant detection


4.1 Alignment using NGMLR

```
ngmlr -x ont -t 24 -r ref.fa -q input.fastq -o output.sam
```

4.2 SV detection with Sniffies
```
samtools sort -T output/tmp -o sorted.bam output.sam
```


```
sniffles -m mapped.sort.bam -v output.vcf
```










