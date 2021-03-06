Seminar 5: BAM to count table
================

In order to use RNA-Seq data for the purposes of differential expression analysis, we need to calculate an expression value for each gene. The digital nature of RNA-seq data allows us to use the number of reads which align to a specific feature as its expression value. In this seminar, we will use BAM and SAM files to generate a count table of reads aligning to the human transcriptome.

By the end of this seminar, you should be able to:

-   use samtools to sort and index a BAM file
-   use HTSeq to count the number of reads that align to each gene/transcript

BAM/SAM - Aligned Sequence data format
======================================

SAM (sequence alignment map) and BAM (the binary version of SAM) files are the preferred output of most alignment tools. SAM and BAM files carry the same information. The only difference is that SAM is human readable, meaning that you can visually inspect each of the reads. You can learn more about the SAM file format [here](http://biobits.org/samtools_primer.html#UnderstandingtheSAMFormat).

The samtools package is one of many that have been developed for working with SAM/BAM alignment files. HTSeq is a tool that summarizes SAM/BAM files into count tables.

To make it as easy as possible to learn how these tools work, each of you should have access to a GSC account with the following packages installed:

-   samtools v1.5
-   HTSeq v0.9.1

Each student should have received individualized login information from one of the TAs.

**Login instructions for Mac/linux users**

To login to the account, open a command line program (i.e Terminal) and type in:

    ssh -X your_username@orca-wg.bcgsc.ca

When prompted, type in your password. Note: you will not see your password when you type it in.

**Login instructions for PC users**

Download PuTTy.

When it has finished downloading, open it up and type in <'you_username@orca-wg.bcgsc.ca>' in the host name input box.

When prompted, type your password.

**Orienting yourself**

When you log in, type:

    ls

You should see an empty folder. This is where you will do all of your work. But first - explore a little bit.

There is a common folder that is one level above your personal folder. Type:

    cd ..
    ls

to see the folders: your\_personal\_folder, stat540 and linuxbrew. The stat540 folder contains data that we will be analyzing in this seminar.

Now navigate back to your personal folder by typing

    cd your_folder_name

The alignment BAM file we will be working with is the transcriptome of the HeLa cell line from the [ENCODE project](http://genome.crg.es/encode_RNA_dashboard/hg19/). HeLa cells are immortalized cervical cancer cells from Henrietta Lacks. For more information on how these cells were procured, and their legacy, read [The Immortal Life of Henrietta Lacks](http://rebeccaskloot.com/the-immortal-life/).

In this tutorial we will be dealing with a subset of the RNA-seq analysis pipeline. Here is an overview of the entire pipeline from biobits.org.

![Conesa et al, 2016](figures/conesa2016.jpg)

Part a) deals with pre-processing, part b) deals with transcriptome profiling, differential gene expression and functional profiling, and part c) deals with other RNAseq visualization strategies. Credit to [Conesa et al, 2016](https://genomebiology.biomedcentral.com/articles/10.1186/s13059-016-0881-8) for the above figure.

*In this tutorial, we are only going to deal with the first part of b). Specifically, we are going to learn how to convert a SAM/BAM file to a count table that can be used for differential expression analysis. Along the way, we will learn more about the SAM and BAM files.*

Working on the command line
===========================

Samtools and HTSeq are both *command-line tools*, meaning that you will have to access them on the command line. There are a variety of ways to do this using R. However, to keep it simple, we will be using the command line directly. Along the way, I will provide screenshots so you can make sure you are on track.

Understanding basic UNIX commands
---------------------------------

Let's dive into basic UNIX commands just to get oriented.

Every command in unix follows the same basic grammar:

    utility flag argument

The **utility** tells the command line what to do. The **flag** is like an option or a preference that modifies how the utility interprets the argument. The **argument** is some extra piece of information that the utility needs to do the required action (i.e a file name).

Changing your directory
-----------------------

Note: substitute **path\_to\_new\_directory** with the relative or absolute path of the new directory you want to navigate to.

    cd path_to_new_directory

To learn more about absolute and relative paths, see Seminar 0a.

Copying a file
--------------

This makes a copy of the file in the same directory.

    cp old_name new_name

Moving a file
-------------

    mv file path_to_new_directory

For more information on command-line basics, go to [this link](https://www.davidbaumgold.com/tutorials/command-line/)

Viewing BAM files
=================

The first thing we are going to do is look at the alignment file, so we can learn more about the structure of BAM/SAM files. In the common repository (stat540), you should have one file named HeLa.bam. This file is not human readable, so we are going to convert it into a SAM file, which we can read.

If you type in the following command

    pwd

the name of your working directory will print to the console.

Make sure the output reads /home/your\_username

If it does not, navigate using *cd* to the correct directory.

Now that we're in the correct directory, we can convert the BAM file into a SAM file, so we can view it.

    samtools view -o hela.sam ../stat540/HeLa.bam 

The *-o* flag signifies that hela.sam is the desired output file. The BAM file I want to analyze is located in the common folder (stat540), which is one level above my current folder. The characters ".." mean "one level above my current folder". So in "../stat540/hela.bam", I'm telling the computer to look for the file HeLa.bam in a folder called stat540 that is one level above my current directory.

Now if you type:

    ls

you should see that you now have a new file: hela.sam.

If you type:

    less hela.sam

The contents of the file will print to console.

![HeLa head](figures/hela_head.jpg)

You can see that each line corresponds to a single read, and each field refers to a different descriptor of the read. For a detailed explanation of what each field means, see [here](http://biobits.org/samtools_primer.html#UnderstandingtheSAMFormat). Understanding what each field means is not of immediate relavence to the following steps. However, it is always good to have a general understanding of the structure of the files you are working with.

Type "q" to return to the command prompt.

Sorting and indexing BAM files
==============================

In order to be able to do things like extract reads that align to a certain chromosome, or that map to a certain genomic region, we need to generate a BAM file index. To do this, we first have to sort the file. We can do this using the following command:

    samtools sort ../stat540/HeLa.bam -o hela_sorted.bam

Once the file has been sorted, we can generate an index file.

    samtools index hela_sorted.bam

*Note: you must always sort before indexing a file*

Generating a count table
========================

Let's take stock of where we are: we have a BAM file, a SAM file and a BAI (BAM alignment) file. It is hard to do much analysis with what we have, because even though we know which chromosome our reads map to, we don't know which genes they map to.

A variety of different tools can be used to generate a count table - i.e a table of how many reads align to each gene. HTSeq and RSEM are two examples of such programs. Each tool works optimally with the output of different aligners, so it is best to read up on how the BAM files you are working with were processed, and to use the feature counting tools.

Today, we are going to use HTSeq, as the BAM files we are using were generated using the STAR tool, which generates BAM files that are compatible with HTSeq. Because we are trying to correlate each read with a transcript ID, we need an alignment file that contains information on the genomic coordinates of each transcript. This alignment file already exists in your repository, and is called gencode.gtf.

    htseq-count -s yes -a 10 hela.sam ../stat540/gencode.gtf > hela_count.txt

Let's break this down.

The *-s* flag refers to whether the reads are stranded. In this case, we know they are, so we write "yes". The *-a* flag refers to the minimum quality of the read we want to allow. Our quality score filter is set at 10. The SAM and GTF files are self-explanatory. The "&gt;" operator means that we want to print the output of the commands on the left to the file on the right.

The resulting file should look like this:

``` r
HTseq_output <- read.table("hela_count.txt", sep = "\t")
head(HTseq_output)
```

    ##                   V1 V2
    ## 1 ENSG00000000003.14  0
    ## 2  ENSG00000000005.5  0
    ## 3 ENSG00000000419.12  0
    ## 4 ENSG00000000457.13  0
    ## 5 ENSG00000000460.16 16
    ## 6 ENSG00000000938.12  0

This can be used as input to a differential expression tool such as edgeR or Deseq2.

Take home exercise
==================

Now we have a table that summarizes the number of reads that align to each transcript. The is the perfect input to use for algorithms like edgeR, whose algorithm requires reads be in integer counts, not normalized by library size. Other tools, however, may require that reads are normalized first.

Your take home exercise is to convert the existing count data into Transcripts per Million (CPM), as defined by Bo Li et al in [this paper](https://academic.oup.com/bioinformatics/article/26/4/493/243395/RNA-Seq-gene-expression-estimation-with-read).

Compare your output with the file in the repository for this course.

This take home exercise will not be graded.
