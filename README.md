# SPeW: SeqPipeWrap 
Pitt-NCBI Hackathon Team
Amanda Poholek (Lead), Argus Sun, Zhou Fang (Ark), Natalie Rittenhouse, Rahil Sethi, Ramya Mallela 

## Introduction
Taking a sequencing pipeline from individual pieces on your workstation to a seamless pipeline that can be run by any user is currently a challenge. Here, we create a proof-of-principle simple RNA-seq pipeline using separate Bash shell scripts that are linked together using NextFlow as a pipeline management tool. This is then wrapped using Docker for distribution and seamless running on other workstations without dependency issues. 

SPeW is a framework for taking a NextGen Seq pipeline (such as RNA-seq, ChIP-seq or ATAC-seq) in any language, and using NextFlow as a pipeline management system to create a flexible, user-friendly pipeline that can be shared in a container platform.

This project was part of the September 2017 Pitt-NCBI Hackathon in Pittsburgh PA

## Dependencies
Docker https://www.docker.com/

samtools http://www.htslib.org/

FastQC https://www.bioinformatics.babraham.ac.uk/projects/fastqc/

cutadapt https://github.com/marcelm/cutadapt

tophat2 http://cole-trapnell-lab.github.io/projects/tophat/

bowtie2 http://bowtie-bio.sourceforge.net/manual.shtml

cufflinks https://github.com/cole-trapnell-lab/cufflinks

R https://www.r-project.org/

NextFlow https://www.nextflow.io/

## Methods 

![ScreenShot](SPeW_workflow.jpg)

To create the proof-of-principle simple RNA-seq pipeline, we started with writing simple Bash shell scripts for each step required in the analysis. These individual steps were then combined together by integrating them into Nextflow. In order to allow for seamless running on any workstation, Docker was then used to wrap the Nextflow code. By wrapping into a container such as Docker, all dependencies required for each step are automatically on the users workstation. Docker has the ability to be used by Singularity, allowing the code to be utilized on a High Computing Cluster(HPC).

### Using Nextflow to String together Bash Scripts
Nextflow is easily installed by using curl:

```
curl -fsSL get.nextflow.io | bash
```

After successfully installing Nextflow, all the bash scripts are moved into a bin folder and the execute permission is granted for all the files.

```
mkdir -p bin 
mv *.sh bin
chmod +x *.sh
```
With all the bash files in the same folder, the Nextflow script can now be written. First, the environment is set and the initial input files are set as parameters: 

```
#!/usr/bin/env nextflow

/*
* input parameters for the pipeline
*/
params.in = "$baseDir/data/*.fastq.gz"
```

A file object is then created from the string parameter: 

``` 
inFiles = file(params.in)
```

By creating the file object, the file can now be used as the input file for the first process. Such as: 

```
process trimming{

input: 
file reads from inFiles
}
```

The output specifies a file ('trimmed_*') that is then put into a variable(trimmedFiles) to be used in the next process. 

```
process trimming{

input: 
file reads from inFiles

output:
file 'trimmed_*' into trimmedFiles
}
```

The script is now ready to be called. These scripts include the ability to use conditional statements: 

```
process trimming{

input: 
file reads from inFiles

output:
file 'trimmed_*' into trimmedFiles

script:
singleEnd = true
adapter1 = "ADATPER_FWD"
adapter2 = "ADATPER_REV"

if (singleEnd==true)
"""
trimming.sh -i reads -s -a1 adapter1 -a2 adapter2
"""
else
"""
trimming.sh -i reads -a1 adapter1 -a2 adapter2
"""
}
```

The output from the trimming step of RNA-seq can easily be accessed by the aligning step: 

```
process trimming{

input: 
file trim from trimmedFiles

output:
file '*.bam' into trimmedFiles

script:
align.sh trim
```

Nextflow seamlessly goes from one process to another by creating a 'work' folder in which all the intermediate files are placed. This work folder also ensures that if/when you run the script multiple times, the results will not be overwritten.  


## Discussion Notes
### Overview

This repo was created as part of an NCBI-hackathon @ Univerisity of Pittsburgh in September of 2017. During this 3 day event, several disucssions occured about the best option to accomplish our goal of creating a method to take a NextGen Seq pipeline into a flexible format that can be easily shared. This is a summary of those discussions, edited by those who were present.

Most Seq pipelines are inherently a string of pre-made programs linked together by the creator according to their personal preferences or prior experience, including comfort of specific programming languages, knowledge of what algorithms exist, and word of mouth about what is "best" option to use. This typically limits sharing the pipeline to another user who may want to make small modifications, or requires installing new software in order to run. In addition, many users may choose to run their pipeline on a local machine, cloud computing or a HPC. Thus, putting a pipeline into a format or framework that manages the pipeline and allows for easy modifications for future users as well as ability to share, would be of tremendous use to many beginner or intermediate bioinformatics users. 

### Test case

As a starting point, we generated a basic RNA-seq pipeline that was made up of individual modules in a bash shell script. These modules are as follows:
1) FastQC
2) Trimming
3) Alignment
4) Annotation
5) Differential Gene Expression (DGE)

Inputs and outputs were defined in Nextflow to link processess together. All outputs were placed in a final directory that had final products as well as intermediate files. 

### Workflow Management Strategy Discussion with a Group of ~25 Computational Biologists and Data Scientists

Next, we needed to determine a method to link these modules together where inputs and outputs are managed for you, and that a future user could add or change a module. Our ideal scenario includes an option where the pipeline can make decisions for you. For example, given output X, th next module used where output X becomes input X could be either module Y or module Z. In addition, we considered how to package the pipeline. This included the option of either wrapping each module in a container (ie docker) that then are linked together within a larger container, or making one whole pipeline and putting it into one container.

The majority of the discussion was around how to link the modules together. Several pipeline management systems were discussed, including Nextflow, snakemake, CWL (common workflow language) and Jupyter notebooks. This conversation included the entire Hackathon group (5 teams) of roughly 25 people total. Ultimately we selected Nextflow, based on the fact that we think Nextflow has the best options for our needs. There was not universal agreement about this, and we believe this will a trial and error process. One downside was no one in the room had any experience with Nextflow, thus we were testing to see if this was indeed a good option.

CWL was widely dismissed by pretty much all members present, as being too labor intensive to use. A few people with CWL experience relayed how difficult and frustrating it was to use, and the time it took to learn considered not worth the effort. 

Snakemake was dismissed as being less flexible than Nextflow.  Many users thought that it is mostly Python oriented, although others confirmed that is not the case.  

Nextflow was chosen because it can use any language, manages inputs and outputs and is meant to be easily wrapped.  

A large part of the discussion included Jupyter notebooks as an alternative to Nextflow. This was considered to be a good in-between for intermediate-level bioinformaticians who want to crack the containers and customize them for particular use cases. Many raised issues of needing to install Jupyter, however no installation should be necessary if Jupyter is "headless" in a container, and Jupyter would only be used by those cracking the containers. However, the we feel it is important to be able to encompass all languages, and therefore this option may have inherent limitations, but perhaps be attractive for others in the future.

Based on these discussions, we chose Nextflow and here report how useful and flexible it ended up being. We hope this may help other users make more informed choices to manage their pipelines and wrap for distribution.

## Futher Directions 
We hope to add additional modules as well as modify current modules to the pipeline, and add in the ability to enter the pipeline at any point. 
