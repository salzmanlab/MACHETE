# MACHETE
More Accurate vs Mismatched Alignment CHimEra Tracking Engine

MACHETE is a fusion detection software that is in development.

PREREQUISITE SOFTWARE: 
1. KNIFE - https://github.com/lindaszabo/KNIFE
2. R version 3.0.2 - or a later version but with the package "data.table" installed
3. Bowtie2 
4. python version 2.7.5

INSTRUCTIONS BEFORE YOU RUN
1. Paired fastq files must named identically and end in _1.fq and _2.fq
Example:
ACCEPTABLE -- MySample_1.fq, MySample_2.fq
UNACCEPTABLE -- MySample1_001.fq, MySample2_001.fq

2.Generate or download a directory of necessary pickles
Pickles are a method of storing serialized data in Python.  We use pickles to store annotated exon information.  

A: Generating the pickles directory from a gtf and fasta file
Use makeExonDB.py to create the pickles directory.  The OutputDirectory is the pickles directory.
usage: python makeExonDB.py -f <genome.fasta> -a <genome.gtf> -o <OutputDirectory>

This can be adapted to any build of any genome.  The downloadable version is the hg19 genome
Genome.fasta is the name of the fasta file used, ex: hg19_genome.fasta
Genome.gtf is the name of the gtf file used, ex: hg19_genome.gtf
OutputDirectory is a path to the pickle directory which is chosen by the user

B: Downloading the pickles directory
At Stanford -- For Sherlock users who are using the HG19 genome, copy or point MACHETE at this directory "/scratch/PI/horence/gillian/HG19exons/"
Outside of Stanford - please email glhsieh@stanford.edu.  The Pickles directory is too large to fit on GitHub.

3. Generate or download an index of indels made from the KNIFE linear junction index. 
Generate the index.  The RegIndelsIndices will be created in a subfolder of the OutputDirectory called “IndelIndices”.
 Run MakeRegIndelsIndex.sh using the following command

Sh MakeRegIndelsIndex.sh <linear junctions fasta> <output directory> <# indels desired> <genome name> <Resource flag (optional)>

Linear junctions fasta is the path to the fasta file containing all linear junctions that is created for KNIFE
Output directory is the path to a directory where the user plans to store linear junction indels
# indels desired is the integer number of indels that will be used in searching for alignment artifact. We have chosen 5
Genome is the name of the genome that is being used. This is only used to name output files.  For example, if “HG19” is entered, output files will be named HG19_reg_indels_1.fa, etc.
Resource flag is an optional field for users of the Stanford SLURM network to specify which queue should be requested, eg “-p owners” or “-p horence” or can be left blank.

Download the index
At Stanford -- for Sherlock users who are using the HG19 genome, copy or point MACHETE at this directory “/scratch/PI/horence/gillian/HG19_reg_indels/IndelIndices/”
Outside of Stanford - please email glhsieh@stanford.edu.  The RegIndels directory is too large to fit on GitHub

PREPARING THE SHELL SCRIPT
Open createFarJunctions_SLURM.sh

line40 - change your INSTALLDIR to the full path to the MACHETE script. 
line42 - change CIRCREF to the path to the reference libraries generated for KNIFE e.g. directory that contains hg19_genome, hg19_transcriptome, hg19_junctions_reg and hg19_junctions_scrambled bowtie indices.
line44  - change REGINDELINDICES to the directory above under “INSTRUCTIONS BEFORE YOU RUN”, bullet #3.  This path should end with “IndelIndices”
line59 - change PICKLEDIR to the directory above under “INSTRUCTIONS BEFORE YOU RUN”, bullet #2


USING A DIFFERENT GENOME OTHER THAN HG19
Change line 44 REGINDELINDICES to the chosen new genome linear junction indices that were generated in “INSTRUCTIONS BEFORE YOU RUN”, bullet #3.
either change or duplicate lines 57 to lines 60.  Choose a genome name to replace “HG19”, and change the PICKLEDIR as needed.  
Either change or duplicate lines 200-206.  Choose a genome name to replace “HG19” and change the paths to the KNIFE generated reference indices to reflect the genome change.

RUNNING MACHETE:

First run KNIFE script completely to generate linear and scrambled junction reports and alignments.

sh createFarJunctions_SLURM.sh <1. KNIFE parent directory> <2. output directory> <3. discordant read distance> <4. ref genome> <5. #indels to use> <6. special queue> 

1. KNIFE parent directory - contains output from the KNIFE algorithm.  This directory is the path to the directory that contains "circReads", "orig", "logs", "sampleStats"
2. Output directory - if not already existing, will be created.
3. discordant read distance -- For testing, we have used 100000 base pairs to identify paired end reads that aligned discordantly.  
4. "HG19" is the only option currently.  If you are have created HG38 or another organism, pickles and a linear junction index must be generated above, instead of downloaded.  Additionally, lines 200-204 can be changed to point at KNIFE reference indices for that organism.
5. Currently using KNIFE convention of "8" for files with read lengths < 70 and "13" for files with read lengths > 70
6. <optional for sherlock use> -- "owners" if you want to run in owners queue, otherwise leave #6 blank


Example command:
sh createFarJunctions_SLURM.sh /scratch/PI/horence/alignments/EWS_FLI_bigmem/ /scratch/PI/horence/alignments/EWS_FLI_bigmem/FarJunc/ 100000 HG19 13 owners 


For the an explanation of outputs of MACHETE, please reference our paper, currently in submission.


=========================================
Output directories

1. DistantPEfiles - 
<STEM>_distant_pairs.txt -- list of three columns - readID / position1 / position2.  Position1 and Position2 are the windows surrounding "discordant" bases from either the genome or reg alignment files.
    Subdirectory <STEM>:
    A. file chr(1,2,3,..,X,Y)_Distant_PE_frequency.txt.  If, for the pair of windows, the upstream location is on chrA, then it will be binned into the chrA_Distant_PE_frequency.txt file.  Three columns appear -- Window1, Window2, and the number of times that these two windows were matched. 
    B. file sorted_chr(1,2,3,..)_Distant_PE_frequency.txt -- File A, sorted on the numerical value of the first column followed by the numerical values of the second column, to reduce parsing time later.
2. fasta -
    Subdirectory <STEM>:
    A: <STEM>_chr(1,2,3,..)FarJunctions.fa - a fusion reference fasta file is created from the corresponding sorted_chr(1,2,3,...)Distant_PE_Frequency. 
    <STEM>_FarJunctions.fa -- the concatenated list of all fasta entries from fasta/<STEM>/<STEM>_chr*_FarJunctions.fa
3. BadFJ/<STEM> -
    A. err/out.txt- error and output files from bowtie alignment of fasta to reference dictionaries
    B. <STEM>_BadFJto(Genome/Reg/Junc/transcriptome).sam - aligns fasta/<STEM>_FarJunctions.fa to the KNIFE reference indices Genome, Reg, Junc, and transcriptome with bowtie parameters "-f --no-sq --no-unal --score-min L,0,-0.24 --n-ceil L,0,100 -p 4 --np 0 --rdg 50,50 --rfg 50,50"
4. BadFJ_ver2/<STEM> - 
    A. err/out.txt- error and output files from bowtie alignment of fasta to reference dictionaries
    B. <STEM>_FarJunctions_R1/2.fa - contains reads from fasta/<STEM>_FarJunctions.fa where all N's are removed from reads and the first 40 nt of the original fasta read is fed into the R1 file and the last 40 nt of the original fasta read is fed into the R2 file. 
    C. <STEM>_BadFJto(Genome/Reg/Junc/transcriptome).sam - aligns BadFJ_ver2/<STEM>_FarJunctions_R1/2.fa to the KNIFE reference indices Genome, Reg, Junc, and transcriptome as if they were paired end alignments to allow a read gap.  Bowtie parameters are "--no-unal --no-mixed --no-sq -p 8 -I 0 -X 50000 -f --ff" 
5. BowtieIndex - 
    Subdirectory <STEM>
    A: <STEM>_FJ_Index*bt2 -- corresponding bowtie indices are created for each of the fasta files from the fasta/<STEM>_chr(1,2,3...)_FarJunctions.fa
6. FarJunctionAlignments - 
    Subdirectory <STEM>
    A. unaligned_<STEM>_R1/2.sam - contains reads that were unaligned to the KNIFE reference indices that have been aligned to the bowtie indices in the BowtieIndex/<STEM>/ directory
7. FarJuncSecondary - 
    Subdirectory <STEM>
    A. still_unaligned_<STEM>_R1/2.fq - contains reads that were unaligned to the KNIFE reference indices that did not align to the FarJunctions bowtie indices in the BowtieIndex/<STEM>/ directory
    Subdirectory AlignedIndels/<STEM>/
    A. still_unaligned_<STEM>_R1/2_indels(1-5).sam - contains reads that were unaligned to the KNIFE reference indices, did not align to the FarJunctions bowtie indices, but DID align to an the expanded FarJunctions indels bowtie indices.
    B. All_<STEM>_R1/2_indels.sam -- concatenates all still_unaligned_<STEM>_R1/2_indels(1-5).sam files but removes reads that aligned to multiple indel indices and keeps the one with the best alignment score and removes reads that aligned to the indel indices but did not overlap the junction by the user specified number of nt.
8. FarJuncIndels

< "IF script doesn't have "INDELS" in glmReports dir means that indels were omitted " >"'





