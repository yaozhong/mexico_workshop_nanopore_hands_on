# Hands on materials for Nanopore data analysis: from basecalling to structural variant detection

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


## Nanopore data analysis enviroment setup
1. Get into the interactive mode
```
qlogin -l os7,s_vmem=16G,mem_req=16G
```

2. Build singularity images with pre-prepared docker file
```
singularity build --sandbox <YOUR-IMAGE-NAME> docker://yaozhong/ont_taiyaki:0.1
```

3. Get into the singularity image
```
singularity shell --writable <YOUR-IMAGE-NAME>
# if GPU is intended to use, please first specify qlogin v100 option, and using the --nv option like
singularity shell --nv --writable <YOUR-IMAGE-NAME>
```

4. Example data and configuration files
We provide 100 reads of raw fast5 signals in the data fold.
Configurations used in guppy is also places there.


## Base-calling


## Assembly


## Structural variant detection





