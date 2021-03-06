# sppIDer Manual  

---
See [README](README.md) for less detailed explanation.  
See [example datasets](examples) for data to test with and the end of manual for usage of these examples.  
See [workflow](workflow.pdf) for a cartoon flowchart of the steps involved.   

Note: The needed inputs must be in the directory you run the docker from. All outputs will also be saved here.
The largest test dataset (SRR1119201) is 587.8Mb and took ~22 minutes to run with 4 cores and 8GB.

---
## combineRefGenomes.py  
This script requires all the reference genomes to be in one location with a key between the file names and what the reference genome should be named in the combined genome. Additionally, a desired output name for the reference fasta is required and the optional trim threshold is allowed. The order the genomes are concatenated follow the order in the key file, the order of the chromosome/scaffolds/contigs will remain as they are in the given reference file, but will be renamed by the name in the key file and numbered sequentially. The key should be a text file with a list of the “desired reference name” and the actual reference fasta separated by a tab. The “desired reference name” cannot include any hyphens (-) an example is given as “SaccharomycesRefKey.txt”. The output will be a concatenated reference with the “desired reference name” and chromosome number (as an Arabic numeral) separated by a hyphen. This format is necessary for the plotting scripts which parse the chromosome names. There is an optional –trim flag so that any contigs shorter than the given interger are not included. For genomes with many short uninformative contigs this reduces the memory usage and speeds things up.  Two steps in sppIDer require index files for the genome used, thus this custom script will also make these required files.  

### Inputs:  
   1. Tab separated text file key to reference genomes to combine.  
   2. Each of those reference genomes as fastas, e.g.:  
        S288c.fasta  
        GCA_002079055.1.fasta  


### Example: executing a combineRefGenome.py


``` 
docker run \
--rm -it \
--mount type=bind,src=$(pwd),target=/tmp/sppIDer/working \
--user "$UID:$(id -g $USERNAME)" \
glbrc/sppider \
  combineRefGenomes.py
  --out REF.fasta \ 
  --key KEY.txt

```  

An optional --trim can be used to trim short uninformative contigs for reference genomes with many short contigs. All contigs shorter than the supplied interger will be ignored.
The KEY.txt file must be tab delimited and the reference genome unique name cannot contain hyphens. See example.

### Outputs:  

####   For downstream use:  
  - REF.fasta  
  - REF.fasta.bwt  
  - REF.fasta.pac  
  - REF.fasta.ann  
  - REF.fasta.amb  
  - REF.fasta.sa  
  - REF.fasta.fai  

####   For humans:    
  - comboLength_REF.fasta.txt - text file with summary of the length of each chromosome, species reference, and total combination genome.

---    
    
## sppIDer.py
 This script requires the combination reference genome made with combineGenomes.py and fastq formatted short-read sequence files for the test of interest. Additionally, you can choose to run it so that depth is analyzed by basepair (-byBP) or grouped by coverage (-byGroup). This script will create a file (output\_sppIDerRun.info) that will print all the argument choice and the time to run each step. Additionally, all the normal standard outputs for each step will be printed to screen along with the time to run each step.   
   This script will then run bwa –mem to map the reads to the combination reference genome and outputs a sam file. This same file is then passed to a custom python script (parseSamFile.py) to parse the data by which genome the reads map to and the mapping quality (MQ) the output of this is passed a Rscript (MQscores\_sumPlot.R) that will plot the percentage of reads that map to each genome and unmapped reads, the same plot without the unmapped bar, for those datasets where most the reads don’t map, and a violin plot showing the distribution of mapping qualities for each genome.  
   The sam output will also be used for samtools view which will only retain MQ >3 and then samtools sort that will order the reads to match the reference order. Next bedtools genomeCoverageBed is called to determine depth of coverage either by basepair (-d) or grouped by coverage (-bga). The –d option give is more accurate but takes longer for large genomes there for the –byGroup option is appropriate for larger genomes. This output is parsed by meanDepth\_sppIDer(-d or –bga).R. These scripts average depth by species, chromosome, and in 10,000 windows and prints these out to text files. The two scripts do the same thing but depend on the input from bedtools. Average depth by species is plotted by sppIDer\_depthPlot\_forSpc.R. The windowed average depth is plotted by sppIDer_depthPlot(-d or –bga).R  

### Inputs:
   1. Combined reference genome produced by combineRefGenomes.py
   2. fastq(s) of interest to test  


### Example: executing sppIDer.py

```  
docker run \
--rm -it \
--mount type=bind,src=$(pwd),target=/tmp/sppIDer/working \
--user "$UID:$(id -g $USERNAME)" \
glbrc/sppider \
  sppIDer.py \
  --out OUT \
  --ref REF.fasta \
  --r1 R1.fastq \
  --r2 R2.fastq  

```  

An optional --byGroup flag can be used for very large combination genomes. This produce a bedfile that doesn't have coverage information for each basepair but by groups. Which speeds up the run.  


### Steps and Outputs:  

- output\_sppIDerRun.info – human readable file tracking the time of each step.    

*bwa mem*  
- Inputs: reference genome, fastq sequence files  
- Output: output.sam - Human readable output of where reads map to reference 


*parseSamFile.py*  
- Inputs: output.sam  
- Outputs: output_MQ.txt - Text file of read counts by Species and Mapping Quality score

*MQscores_sumPlot.R*  
- Inputs: output\_MQ.txt   
- Outputs:   
   + output\_MQsummary.txt - Text file with summary of how many and how well reads map to each genome  
   + output\_plotMQ.pdf - Plot of reads mapped per genome and Mapping Quality per genome   
   
*samtools view*    
- Inputs: output.sam    
- Outputs: output.view.bam - Binary file of just reads with mapping quality >3    

*samtools sort*  
- Inputs: output.sam  
- Oputputs: output.sort.bam - Binary file of reads ordered by reference genome  
  
*bedtools genomeCoverageBed*  
- Inputs: output.sort.bam  
- Outputs: output-(d/g).bedgraph - Coverage of reference genome, per base pair position (d) or grouped by coverage (g)  
  
*meanDepth_sppIDer(-d/-bga).R*  
- Inputs: output-(d/g).bedgraph  
- Outputs:  
   - output\_speciesAvgDepth-(d/g).txt - Text file summary of coverage for each species including: mean, relativeMean (speciesMean/globalMean), max, and median coverage  
   - output\_chrAvgDepth-(d/g).txt - Text file summary of coverage for each chromosome of each species  
   - output\_winAvgDepth-(d/g).txt - Text file summary of coverage of the genome split into 10,000 windows  
   
*sppIDer_depthPlot_forSpc.R*  
- Inputs: output\_speciesAvgDepth-(d/g).txt  
- Outputs: output\_speciesDepth.pdf - Plot of coverage by species  
  
*sppIDer_depthPlot-d.R*  
- Inputs: output\_winAvgDepth-(d/g).txt
- Outputs: output\_sppIDerDepthPlot-(d/g).pdf - Plot of coverage by genome split into 10,000 windows  
     
---
     
# mitoSppIDer  

For mitoSppIDer the combineRefGenomes.py scripts again must be run just for mitochondrial genomes desired. Additionally, regions of interested, e.g. coding regions, can be highlighted on the final output if the combineGFF.py script is also run.     

## combineGFF.py   
   This script combines gff style files that include information of the regions desired to be highlighted on the final plot. See examples. The key should be a text file with a list of the “desired reference gff name” and the actual reference gff separated by a tab.    
   
### Inputs:  
   1. Tab separated text file key to reference genomes to combine.  
   2. Each of those reference genome gffs, e.g.:  
      S288c.gff  
      GCA_002079055.1.gff  
	   
### Example: executing a combineRefGenome.py    

```
docker run \
--rm -it \
--mount type=bind,src=$(pwd),target=/tmp/sppIDer/working \
--user "$UID:$(id -g $USERNAME)" \
glbrc/sppider \
  combineGFF.py
  --out REF.gff \ 
  --key GFF_KEY.txt
   
```    
	         
### Outputs:    
   combinedRef.gff - a tab delimited text file that specifies the species, start, end, midpoint, and geneName for each intron to be delineated in the final plots.  
     
## mitoSppIDer.py    
   This pipeline is very similar to the main sppIDer.py only the input reference genomes only contains mitochondrial (mito) or other small genomes. Because of the size of most mito genomes the by-base-pair is the default. The final plots will be stacked plots of each supplied mito genome and if a combined gff is supplied then those regions will be labeled and shaded on the final plot.   
   
### Inputs:  
   1. Combined reference genome of just mitochondrial genomes made with combineRefGenome.py  
   2. Combined gff style file for adding emphasis in plots, if desired.     
   3. fastq(s) of interest to test  
     
### Example: executing a mitoSppIDer.py pipeline  

```
docker run \
--rm -it \
--mount type=bind,src=$(pwd),target=/tmp/sppIDer/working \
--user "$UID:$(id -g $USERNAME)" \
glbrc/sppider \
  mitoSppIDer.py \
  --out OUT \
  --ref MITO_REF.fasta \
  --r1 R1.fastq \
  --r2 R2.fastq
```  
  
An optional --gff can be used if you are providing a combined gff of the regions that should be marked on the final plots.

### Steps and Outputs:    
- output\_mitoSppIDerRun.info – human readable file tracking the time of each step.  
   
*bwa mem*  
- Inputs: mito reference genome, fastq sequence files, gff  
- Output: output.sam - Human readable output of where reads map to reference   
  
*parseSamFile.py*  
- Inputs: output.sam  
- Outputs: output\_MQ.txt - Text file of read counts by Species and Mapping Quality score  
  
*MQscores_sumPlot.R*  
- Inputs: output\_MQ.txt  
- Outputs:  
    - output\_MQsummary.txt - Text file with summary of how many and how well reads map to each genome  
    - output\_plotMQ.pdf - Plot of reads mapped per genome and Mapping Quality per genome  
   
*samtools view*  
- Inputs: output.sam  
- Outputs: output.view.bam - Binary file of just reads with mapping quality >3  
  
*samtools sort*  
- Inputs: output.sam  
- Oputputs: output.sort.bam - Binary file of reads ordered by reference genome  
  
*bedtools genomeCoverageBed*  
- Inputs: output.sort.bam  
- Outputs: output-d.bedgraph - Coverage of reference genome, per base pair position (d)  
   
*meanDepth_sppIDer-d.R*  
- Inputs: output-d.bedgraph  
- Outputs:  
  - output\_speciesAvgDepth-d.txt - Text file summary of coverage for each species including: mean, relativeMean (speciesMean/globalMean), max, and median coverage  
  - output\_chrAvgDepth-d.txt - Text file summary of coverage for each chromosome of each species  
  - output\_winAvgDepth-d.txt - Text file summary of coverage of the genome split into 10,000 windows  
  
*mitoSppIDer_depthPlot-d.R*  
- Inputs:  
  - output\_winAvgDepth-d.txt  
  - combine.gff  
- Outputs: output\_sppIDerDepthPlot-d.pdf - Plot of coverage by mito-genome split into 10,000 windows  


# Examples:  
Below are examples for the stand alone scripts the data or example outputs can be found in [examples](examples)
  
## combineRefGenomes.py  
For a working *Saccharomyces* combined reference genome locations can be found in [example_information.md](examples/example\_information.md).  

``` 
docker run \
--rm -it \
--mount type=bind,src=$(pwd),target=/tmp/sppIDer/working \
--user "$UID:$(id -g $USERNAME)" \
glbrc/sppider \
  combineRefGenomes.py
  --out SaccharomycesCombo.fasta \ 
  --key SaccharomycesGenomesKey.txt

```  
The output from this can be used as the reference input for sppIDer.py  
All files must be kept together in the same directory to be usable. 

## sppIDer.py  
Info on where example short read data can be found can be found in [example_information.md](examples/example\_information.md).  

### Pure strain isolated from North Carolina  
```  
docker run \
--rm -it \
--mount type=bind,src=$(pwd),target=/tmp/sppIDer/working \
--user "$UID:$(id -g $USERNAME)" \
glbrc/sppider \
  sppIDer.py \
  --out SRR2586160 \
  --ref SaccharomycesCombo.fasta \
  --r1 SRR2586160_1.fastq \
  --r2 SRR2586160_2.fastq  

```  
For this strain reads map predominantly to the _Saccharomyces eubayanus_ reference. Examples of the plot outputs and example text file outputs can be found in [exampleOutputs](examples/exampleOutputs.tar.gz).  
~18 minutes to run with 4 cores and 8GB  

### Lager brewing strain, two way hybrid  
```  
docker run \
--rm -it \
--mount type=bind,src=$(pwd),target=/tmp/sppIDer/working \
--user "$UID:$(id -g $USERNAME)" \
glbrc/sppider \
  sppIDer.py \
  --out SRR2586169 \
  --ref SaccharomycesCombo.fasta \
  --r1 SRR2586169_1.fastq \
  --r2 SRR2586169_2.fastq  

```  
For this strain reads map to both _Saccharomyces cerevisiae_ and _Saccharomyces eubayanus_ references, which is expected of a Lager strain. Examples of the plot outputs and example text file outputs can be found in [exampleOutputs](examples/exampleOutputs.tar.gz).  
~17 minutes to run with 4 cores and 8GB  

### Multiway hybrid
```  
docker run \
--rm -it \
--mount type=bind,src=$(pwd),target=/tmp/sppIDer/working \
--user "$UID:$(id -g $USERNAME)" \
glbrc/sppider \
  sppIDer.py \
  --out SRR1119201 \
  --ref SaccharomycesCombo.fasta \
  --r1 SRR1119201_1.fastq \
  --r2 SRR1119201_2.fastq  

```  
For this cider strain reads map to _Saccharomyces cerevisiae_, _Saccharomyces kudriavzevii_, and _Saccharomyces uvarum_ with minor contributions from _Saccharomyces eubayanus_. Examples of the plot outputs and example text file outputs can be found in [exampleOutputs](examples/exampleOutputs.tar.gz).  
~22 minutes to run with 4 cores and 8GB.  

## mitoSppIDer scripts  
Information about all files to make the combination reference fasta and gff can be found can be found in [example_information.md](examples/example\_information.md).  

### combineGFF.py
```
docker run \
--rm -it \
--mount type=bind,src=$(pwd),target=/tmp/sppIDer/working \
--user "$UID:$(id -g $USERNAME)" \
glbrc/sppider \
  combineGFF.py
  --out SaccharomycesMitoCombo.gff \ 
  --key mitoGFFKey.txt
   
```    

### combineRefGenomes.py
``` 
docker run \
--rm -it \
--mount type=bind,src=$(pwd),target=/tmp/sppIDer/working \
--user "$UID:$(id -g $USERNAME)" \
glbrc/sppider \
  combineRefGenomes.py
  --out SaccharomycesMitoCombo.fasta \ 
  --key mitoRefKey.txt

```  

### mitoSppIDer.py
```
docker run \
--rm -it \
--mount type=bind,src=$(pwd),target=/tmp/sppIDer/working \
--user "$UID:$(id -g $USERNAME)" \
glbrc/sppider \
  mitoSppIDer.py \
  --out SRR2586169mito \
  --ref SaccharomycesMitoCombo.fasta \
  --gff SaccharomycesMitoCombo.gff \
  --r1 SRR2586169_1.fastq \
  --r2 SRR2586169_2.fastq
```  


