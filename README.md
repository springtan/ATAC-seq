## Table of Contents
[ATAC-seq overview](#overview)<br>
[Experimental design](#design)<br>
[Compute access / Odyssey](#odyssey)<br>
[Sequence reads](#reads)<br>
[Quality control](#qc)<br>
[Alignment](#alignments)<br>
[Peak calling](#peak)<br>
[Next steps](#next)<br>
[References](#references)<br>


## ATAC-seq overview <a name="overview"></a>

ATAC-seq (Assay for Transposase-Accessible Chromatin with high-throughput sequencing) is a method for determining chromatin accessibility across the genome.  It utilizes a hyperactive Tn5 transposase to insert sequencing adapters into open chromatin regions (Fig. 1).  High-throughput sequencing then yields reads that indicate these regions of increased accessibility.

<figure>
  <img src="overview.png" alt="ATAC-seq" width="600">
  <figcaption><strong>Figure 1.</strong>  ATAC-seq overview (<a href="https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4374986/">Buenrostro <i>et al.</i>, 2015</a>).</figcaption>
</figure>
<br>

## Experimental design <a name="design"></a>

The developers of the ATAC-seq method have published a [detailed protocol](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4374986/) for the laboratory procedure (Buenrostro *et al.*, 2015).

Here are a few additional things to consider when planning an ATAC-seq experiment:

### 1. Replicates

Like most high-throughput sequencing applications, ATAC-seq requires that biological replicates be run.  This ensures that any signals observed are due to biological effects and not idiosyncracies of one particular sample or its processing.  To begin with, two replicates per experimental group are sufficient.

### 2. Controls

With ATAC-seq, control groups are not typically run, presumably due to the expense and the limited value obtained.  A control for a given sample would be genomic DNA from the sample that, instead of transposase treatment, is fragmented (e.g. by sonication), has adapters ligated, and is sequenced along with the ATAC sample.  Such a control could be useful to help define regions of the genome that are more challenging to sequence or to align reads unambiguously.

### 3. PCR amplification

In preparing libraries for sequencing, the samples should be amplified using as few PCR cycles as possible.  This will help to reduce PCR duplicates, which are exact copies of DNA fragments that can interfere with the biological signal of interest.

### 4. Sequencing depth

Sequencing depth will vary based on the size of the reference genome and the degree of open chromatin expected.  For studies of human samples, [Buenrostro *et al.* (2015)](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4374986/) recommend more than 50 million mapped reads per sample.

### 5. Sequencing mode

For ATAC-seq, we recommend **paired-end sequencing**, for several reasons.

* More sequence data leads to better alignment results.  Many genomes contain numerous repetitive elements, and failing to align reads to certain genomic regions unambiguously renders those regions inaccessible to the assay.  Additional sequence data, such as with paired-end sequencing, helps to reduce these alignment ambiguities.

* With ATAC-seq, we are interested in knowing the full span of the DNA fragments generated by the assay.  A DNA fragment generated by the ATAC is typically longer than a sequence read, so a read will define only one end of the fragment.  Therefore, with single-end sequencing, we would have to guess where the other end of the fragment is.  Since paired-end sequencing generates reads from both ends, the full span of the DNA fragment is known precisely.

* PCR duplicates are identified more accurately.  As noted above, PCR duplicates are artifacts of the procedure, and they should be removed as part of the analysis pipeline (see below for more details).  However, computational programs that remove PCR duplicates (e.g. Picard's [MarkDuplicates](http://broadinstitute.github.io/picard/command-line-overview.html#MarkDuplicates)) typically identify duplicates based on comparing ends of aligned reads.  With single-end reads, there is only one position to compare, and so any reads whose 5' ends match will be considered duplicates.  Thus, many false positives may result, and perfectly good reads will be removed from further analysis.  On the other hand, after paired-end sequencing, both ends of the original DNA fragments are defined.  To be declared a duplicate, both ends of one fragment would need to match both ends of another fragment, which is far less likely to occur by chance.  Therefore, paired-end sequencing leads to fewer false positives.


## Compute access / Odyssey <a name="odyssey"></a>

This document assumes that you have an account on the [Odyssey computer cluster](https://www.rc.fas.harvard.edu/training/introduction-to-odyssey-online/) of Harvard University.  An account can be requested [here](https://portal.rc.fas.harvard.edu/request/account/new).

Programs, like those listed below (e.g. FastQC, Bowtie2, MACS2), are run on Odyssey by submitting jobs via the [SLURM management system](https://www.rc.fas.harvard.edu/resources/running-jobs/).
The jobs take the form of shell scripts, which are submitted with the [sbatch command](https://www.rc.fas.harvard.edu/resources/running-jobs/#Submitting_batch_jobs_using_the_sbatch_command).  The shell scripts request computational resources (time, memory, and number of cores) for a job; it is better to request more resources than expected, rather than risk having a job terminated prematurely for exceeding its limits.


## Sequence reads <a name="reads"></a>

The raw sequence files generated by the sequencing core are in [FASTQ format](https://en.wikipedia.org/wiki/FASTQ_format).  They are gzip-compressed, with '.gz' file extensions.  It is unnecessary, not to mention wasteful of time and disk space, to decompress the sequence files; all common bioinformatics tools can analyze compressed files.

For paired-end sequencing, there are two files per sample: `<sample>.R1.fastq.gz` and `<sample>.R2.fastq.gz`.

Samples that were sequenced on multiple lanes may have separate files for each lane; these should be concatenated using the `cat` command:

    cat  <sample>.lane1.R1.fastq.gz  <sample>.lane2.R1.fastq.gz  >  <sample>.R1.fastq.gz
    cat  <sample>.lane1.R2.fastq.gz  <sample>.lane2.R2.fastq.gz  >  <sample>.R2.fastq.gz

However, different replicates should not be concatenated, but instead should be processed separately.


## Quality control <a name="qc"></a>

### FastQC

It is generally a good idea to generate some quality metrics for your raw sequence data.  One tool that is commonly used for this purpose is [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/).

On Odyssey, each sequence file would be analyzed like this:

    module load fastqc
    fastqc <sample>.R1.fastq.gz


FastQC is efficient; it can process a file of 20 million reads in about 5 minutes with less than 250MB memory used.  The output from FastQC is an HTML file, which can be examined via a web browser.  The report lists various statistics about your reads, such as their count and lengths.  It also provides graphical representations of your data based on a number of categories, including quality scores, GC levels, PCR duplicates, and adapter content.

The FastQC report is there to alert you to potential issues with your data, but it is not the final determinant of the outcome of your ATAC-seq experiment.  Do not be overly concerned if your FastQC reports contain one or more red 'X' marks; this is not a reason to delete your data and start all over again.


### Adapter removal

For reads derived from short DNA fragments, the 3' ends may contain portions of the Illumina sequencing adapter.  This adapter contamination may prevent the reads from aligning to the reference genome and adversely affect the downstream analysis.  If you suspect that your reads may be contaminated with adapters (either from the FastQC report ["Overrepresented sequences" or "Adapter content" sections], or from the size distribution of your sequencing libraries), you should run an adapter removal tool.  Here are two options:

#### 1. Cutadapt

One of the most widely used adapter removal programs is [cutadapt](http://cutadapt.readthedocs.io/en/stable/guide.html).  Cutadapt searches input reads for a given adapter sequence.  When it finds the adapter, it removes the adapter and everything that follows it.  Reads that do not match the adapter remain unaltered.

Some things to note when using cutadapt:

* The adapter sequences need to be provided via the `-a` argument.  If you do not know which adapters were used for your samples, consult the sequencing core.

* Cutadapt will attempt to match a minimal length of the provided adapter sequence.  The default value for this argument (`-O`) is 3bp.  The downside of using such a small value is the possibility of false positives (trimming reads' good sequences that happen to match part of the adapter).  On the other hand, increasing this parameter will result in more false negatives, since reads with adapter contamination may contain sequencing errors that prevent a match.

#### 2. NGmerge

An alternative approach to adapter removal is provided by [NGmerge](https://github.com/harvardinformatics/NGmerge), which was developed in the Informatics Group.  Unlike cutadapt, NGmerge does not require that the adapter sequences be provided, nor does it require a parameter for the minimum length of adapter to match (in fact, it does not perform adapter matching at all).  However, it works only with paired-end sequencing, so those with single-end sequencing should stick with cutadapt.

NGmerge is based on the principle that, with paired-end sequencing, adapter contamination occurs only when both reads fully overlap.  The program tests each pair of reads for overlap, and in cases where they do, it clips the 3' overhangs of both reads (Fig. 2).  Reads that align without 3' overhangs (or do not align at all) remain unaltered.

<figure>
  <img src="adapter_removal.png" alt="Adapter removal" width="300">
  <figcaption><strong>Figure 2.</strong>  The original DNA fragment contains sequencing adapters on both ends (gray boxes).  Because the fragment is short, the paired-end reads (R1, R2) extend into the sequencing adapters.  NGmerge aligns the reads, and clips the 3' overhangs.</figcaption>
</figure>
<br>
<br>

NGmerge is available on Odyssey in the ATAC-seq module:

    module load ATAC-seq
    NGmerge -a  -1 <sample>.R1.fastq.gz  -2 <sample>.R2.fastq.gz  -o <name>

The output files will be `<name>_1.fastq.gz` and `<name>_2.fastq.gz`.  Of the many arguments available with NGmerge, here are the most important ones for this application:

| Argument   | Description                                  |
|:----------:|----------------------------------------------|
| `-a`       | Adapter-removal mode (**must** be specified) |
| `-e <int>` | Minimum length of overlap, i.e. the minimum DNA fragment length (default 50bp) |
| `-n <int>` | Number of cores on which to run              |
| `-v`       | Verbose mode                                 |


For more information about the parameters and options of NGmerge, please consult the [UserGuide](https://github.com/harvardinformatics/NGmerge/blob/master/UserGuide.pdf) that accompanies the [source code](https://github.com/harvardinformatics/NGmerge) on GitHub.

For input files of 20 million paired reads, NGmerge should run in less than one hour on a single core, with minimal memory usage.  Of course, the run-time will decrease with more cores (`-n`).

Other than adapter removal, we do not recommend any trimming of the reads.  Such adjustments can complicate later steps, such as the identification of PCR duplicates.


## Alignment <a name="alignments"></a>

The next step is to align the reads to a reference genome.  There are many programs available to perform the alignment.  Two of the most popular are [BWA](http://bio-bwa.sourceforge.net/bwa.shtml) and [Bowtie2](http://bowtie-bio.sourceforge.net/bowtie2/manual.shtml).  We will focus on Bowtie2 here.

### Genome indexing

In order to align reads to a genome, the reference sequence must be indexed.  This is a time- and memory-intense procedure, but it needs to be done only once for a given genome.

For many model organisms, the genome and pre-built reference indexes are available from [iGenomes](https://support.illumina.com/sequencing/sequencing_software/igenome.html).  Otherwise, Bowtie2 indexes are made from a FASTA genome file using the program `bowtie2-build`:

```
module load bowtie2
bowtie2-build  <genome.fa>  <genomeIndexName>
```

### Alignment

Once the indexes are built, the reads can be aligned using Bowtie2.  A brief look at the [manual](http://bowtie-bio.sourceforge.net/bowtie2/manual.shtml) reveals the large number of parameters and options available with Bowtie2.  Here are a few to consider with ATAC-seq datasets:

<table>
  <tr>
    <th align="center">Argument</th>
    <th>Description</th>
  </tr>
  <tr>
    <td align="center"><code>-X &lt;int&gt;</code></td>
    <td>Maximum DNA fragment length (default 500bp).  If you anticipate that you may have DNA fragments longer than the default value, you should increase this parameter accordingly; otherwise, alignments from such fragments will be considered not properly paired (see Fig. 3B below).</td>
  </tr>
  <tr>
    <td nowrap align="center"><code>--very-sensitive</code></td>
    <td>Bowtie2 has a number of alignment and effort parameters that interact in complex (and sometimes unexpected) ways.  Preset collections of these parameters are provided for convenience; the default is <code>--sensitive</code>, but better alignment results are frequently achieved with <code>--very-sensitive</code>.</td>
  </tr>
  <tr>
    <td align="center"><code>-p &lt;int&gt;</code></td>
    <td>Number of cores on which to run</td>
  </tr>
</table>


The output is a [SAM file](https://samtools.github.io/hts-specs/SAMv1.pdf), which contains alignment information for each input read.  The SAM can be compressed to a binary format (BAM) and sorted with [SAMtools](http://www.htslib.org/doc/samtools.html).  This is best accomplished by piping the output from Bowtie2 directly to `samtools view` and `samtools sort`, e.g.:

```
module load bowtie2   # if not already loaded
module load samtools
bowtie2  --very-sensitive  -x <genomeIndexName>  -1 <name>_1.fastq.gz  -2 <name>_2.fastq.gz \
  |  samtools view -u -  \
  |  samtools sort -  >  <BAM>
```

For input files of 20 million paired reads, this command takes around five hours.  This can be decreased by increasing the number of cores in the Bowtie2 command.  For example, one could specify eight cores for Bowtie2 with `-p 8` and adjust the request in the SLURM script to `#SBATCH -n 10` (that is, eight cores for Bowtie2 and one each for SAMtools view and sort).  The memory usage of Bowtie2 depends primarily on the genome length; enough must be requested to load the genome indexes.

Bowtie2 also provides (via stderr) a summary of the mapping results, including counts of reads analyzed, properly paired alignments, and reads that aligned to multiple genomic locations.  By default, Bowtie2 will randomly choose one of multiple equivalent mapping locations for a read.


### Alignment adjustments

#### Mitochondrial reads

It is a well-known problem that ATAC-seq datasets usually contain a large percentage of reads that is derived from mitochondrial DNA (for example, see [this discussion](http://seqanswers.com/forums/showthread.php?t=35318)).  Some have gone as far as [using CRISPR to reduce mitochondrial contamination](https://www.nature.com/articles/s41598-017-02547-w).  The recently published [Omni-ATAC method](https://www.nature.com/articles/nmeth.4396) uses detergents to remove mitochondria and is likely to be more accessible for most researchers (but, do **not** follow their computational workflow).

Regardless of your lab protocol, you will have some mitochondrial reads in your sequence data.  Since there are no ATAC-seq peaks of interest in the mitochondrial genome, these reads will only complicate the subsequent steps.  Therefore, we recommend that they be removed from further analysis, via one of the following methods:

1. Remove the mitochondrial genome from the reference genome before aligning the reads.  In human/mouse genome builds, the mitochondrial genome is labeled 'chrM'.  That sequence can be deleted from the reference prior to building the genome indexes.  The downside of this approach is that the alignment numbers will look much worse; all of the mitochondrial reads will count as unaligned.

2. Remove the mitochondrial reads after alignment.  A python script, creatively named removeChrom, is available in the ATAC-seq module to accomplish this.  For example, to remove all 'chrM' reads from a BAM file, one would run this:

```
module load ATAC-seq samtools   # if not already loaded
samtools view -h  <inBAM>  |  removeChrom - - chrM  |  samtools view -b -  >  <outBAM>
```

#### PCR duplicates

PCR duplicates are exact copies of DNA fragments that arise during PCR.  Since they are artifacts of the library preparation procedure, they may interfere with the biological signal of interest.  Therefore, they should be removed as part of the analysis pipeline.

One commonly used program for removing PCR duplicates is Picard's [MarkDuplicates](http://broadinstitute.github.io/picard/command-line-overview.html#MarkDuplicates).  It is run as follows:

    module load picard
    java -jar $PICARD_TOOLS_HOME/picard.jar MarkDuplicates I=<inBAM> O=<outBAM> M=dups.txt REMOVE_DUPLICATES=true

The output file specified by `M=` lists counts of alignments analyzed and duplicates identified.

#### Non-unique alignments

It is not uncommon for short sequence reads to align equally well to multiple locations in a reference genome, especially given the repetitive nature of genomes.  Some researchers choose to remove non-uniquely aligned reads, using the `-q` parameter of `samtools view`.  For reads with multiple alignments, Bowtie2 (or BWA) will report only one alignment (by default) and will assign it a low mapping quality (MAPQ) score, which is defined as -10 * log<sub>10</sub>Pr{mapping position is wrong}.  To eliminate alignments with MAPQ &lt; 10 (i.e., where Bowtie2 has determined Pr{mapping position is wrong} &gt; 0.1), one would run the following:

```
module load samtools   # if not already loaded
samtools view -b  -q 10  <inBAM>  >  <outBAM>
```

## Peak calling <a name="peak"></a>

Model-based Analysis of ChIP-Seq ([MACS2](https://github.com/taoliu/MACS)) is a program for detecting regions of genomic enrichment.  Though designed for ChIP-seq, it works just as well on ATAC-seq and other genome-wide enrichment assays that have narrow peaks.  The main program in MACS2 is `callpeak`, and its options are described below.  (Note that the latest version of MACS2 on Odyssey, v2.1.2_dev, is different from the most recent official [release](https://pypi.org/project/MACS2/) from March 2016.)

As input, MACS2 takes the alignment files produced in the previous steps.  However, it is important to remember that the read alignments indicate only a portion of the DNA fragments generated by the ATAC.  Therefore, one must consider how one wants MACS2 to interpret the alignments.

### Alignments to analyze

With paired-end sequencing, the types of alignments that are produced fall into two basic categories: "properly paired" and "singletons" (Fig. 3).

<figure>
  <img src="alignments.png" alt="Alignment types" width="700">
  <figcaption><strong>Figure 3.</strong>  Paired-end alignments.  <strong>A:</strong> Reads that are properly paired align in opposite orientations on the same reference sequence (chromosome).  The reads may overlap to some extent (bottom).  <strong>B:</strong> A singleton read (R1) can be not properly paired for several reasons: if its mate (R2) is unaligned (upper left), aligns to a different chromosome (upper right), aligns in the incorrect orientation (middle cases), or aligns in the correct orientation but at an invalid distance (bottom).  In all cases except the upper left, the R2 read is also a singleton.</figcaption>
</figure>
<br>
<br>

An important consideration when using MACS2 is deciding which types of alignments should be analyzed and how those alignments should be interpreted.  The analysis mode is set by the `-f` argument.  Here are the options with MACS2:

1. Analyze only properly paired alignments, but ignore R2 reads and treat R1 reads as singletons.  This is the default option (`-f AUTO`).  MACS2 creates a model of the fragment lengths and extends the 3' ends of the R1 reads to the calculated average length.  An alternative is to skip this model building and instead extend each read to a specified length (e.g., `--nomodel --extsize 300` for 300bp fragments).  The value of the length parameter is usually determined from the average size during library preparation (the default value is 200bp if no value is specified).  However, neither of these approaches utilizes the value of paired-end sequencing, which defines both fragment ends.

2. Analyze only properly paired alignments with `-f BAMPE`.  Here, the fragments are defined by the paired alignments' ends, and there is no modeling or artificial extension.  Singleton alignments are ignored.  This is the preferred option for using only properly paired alignments.

3. Analyze all alignments.  For this approach, a python script, SAMtoBED, is available in the ATAC-seq module.  This script converts the read alignments to BED intervals, treating the properly paired alignments as such and extending the singleton alignments as specified.  There are four options for the singletons: ignore them, keep them as is, extend them to an arbitrary length (similar to the `--extsize` option of MACS2), or extend them to the average length calculated from the properly paired alignments.  Here is an example command, using the "extend to average length" option (`-x`):

```
module load ATAC-seq
samtools view -h  <BAM>  |  SAMtoBED  -i -  -o <BED>  -x  -v
```

The output from SAMtoBED is a [BED file](https://genome.ucsc.edu/FAQ/FAQformat.html#format1) that should be analyzed by MACS2 with `-f BEDPE`.

(Note that the BEDTools program [bamtobed](http://bedtools.readthedocs.io/en/latest/content/tools/bamtobed.html) cannot be used here, since its output is in a nonstandard BED format that MACS2 cannot analyze.)

In deciding among these analysis options, it may help to consider the counts produced by Bowtie2, which indicate how many alignments fall into each category.  For example, if most of the reads are aligned in proper pairs, it may be sufficient to use option #2.  On the other hand, option #3 is preferred if a substantial fraction of the reads consists of singletons.


### Other arguments

In addition to the analysis mode explained above, MACS2 has a number of parameters and options to set.  Here are a few to consider:

<table>
  <tr>
    <th align="center">Argument</th>
    <th>Description</th>
  </tr>
  <tr>
    <td align="center"><code>-n &lt;str&gt;</code></td>
    <td>Name of the sample.  The output files will be named using the specified string as the prefix.</td>
  </tr>
  <tr>
    <td align="center"><code>-g &lt;int&gt;</code></td>
    <td>Effective genome size, i.e. the size of the organism’s genome that can be analyzed (not including Ns, repetitive sequences, etc.).  This will be less than the actual genome size.  Parameters are provided for some model organisms, and the default value is <code>hs</code> (for <i>Homo sapiens</i>), which corresponds to a value of 2.7e9.</td>
  </tr>
  <tr>
    <td align="center"><code>-q &lt;float&gt;</code></td>
    <td>Minimum <i>q</i>-value (adjusted <i>p</i>-value, or false discovery rate [FDR]) for peak calling (default 0.05).  Reducing this threshold will decrease the number of peaks identified by MACS2 but increase the confidence in the called peaks.</td>
  </tr>
  <tr>
    <td nowrap align="center"><code>--keep-dup &lt;arg&gt;</code></td>
    <td>How to handle PCR duplicates (default: <code>--keep-dup 1</code>, i.e. remove all potential duplicates).  If PCR duplicates have been removed by another program, such as Picard's MarkDuplicates, then specify <code>--keep-dup all</code>.</td>
  </tr>
  <tr>
    <td nowrap align="center"><code>--max-gap &lt;int&gt;</code></td>
    <td>Maximum gap between significant sites to cluster them together (default 50bp). <strong>(v2.1.2_dev only)</strong></td>
  </tr>
  <tr>
    <td nowrap align="center"><code>--min-length &lt;int&gt;</code></td>
    <td>Minimum length of a peak (default 100bp). <strong>(v2.1.2_dev only)</strong></td>
  </tr>
</table>


Note that MACS2 is not multithreaded, so it runs on a single core only.

The full MACS2 command for an ATAC-seq dataset from *C. elegans*, using all alignments (converted to BED intervals), might look like this:

    module load macs2
    macs2 callpeak  -t <BED>  -f BEDPE  -n NAME  -g ce  --keep-dup all

Calling peaks for 20 million fragments should require less than ten minutes and 1GB of memory.


### Output files

There are three output files from a standard `macs2 callpeak` run.  For a run with `-n NAME`, the output files are NAME_peaks.xls, NAME_peaks.narrowPeak, and NAME_summits.bed.  The most useful file is NAME_peaks.narrowPeak, a plain-text BED file that lists the genomic coordinates of each peak called, along with various statistics (fold-change, *p*- and *q*-values, etc.).


## Next steps <a name="next"></a>

Once the peaks have been identified by MACS2 for a set of samples, there are several follow-up steps that can be taken, depending on the experimental design. 

### Visualization

Some researchers find it useful to generate visualizations of the peaks in a genomic context.

For ATAC-seq in model organisms, the peak file (NAME_peaks.narrowPeak) can be uploaded directly to the [UCSC genome browser](https://genome.ucsc.edu/cgi-bin/hgCustom).  Note that a peak file without a header line should have the following added to the beginning of the file:

    track type=narrowPeak

An alternative visualization tool is the [Integrative Genomics Viewer](http://software.broadinstitute.org/software/igv/) (IGV).  Peak files can be loaded directly (File → Load from File).  Viewing BAM files with IGV requires that they be sorted (by coordinate) and indexed using [SAMtools](http://www.htslib.org/doc/samtools.html).  Note that the BAMs show the read alignments, but not the full fragment lengths as generated by the ATAC and analyzed by MACS2.


### Comparing peak files

Determining genomic regions that are common or different to a set of peak files is best accomplished with [BEDTools](http://bedtools.readthedocs.io/en/latest/index.html), a suite of software tools that enables "genome arithmetic."

For example, [bedtools intersect](http://bedtools.readthedocs.io/en/latest/content/tools/intersect.html) determines regions that are common to two peak files, such as replicates of the same experimental group.

Finding differences between two peak files, such as control vs. experimental groups, is accomplished via [bedtools subtract](http://bedtools.readthedocs.io/en/latest/content/tools/subtract.html).


### Annotation

It is helpful to know what genomic features are near the peaks called by MACS2.  One program that is commonly used to annotate peaks is [ChIPseeker](https://bioconductor.org/packages/release/bioc/html/ChIPseeker.html).  Like MACS2, ChIPseeker was originally designed to be used in the analysis of ChIP-seq, but it works just as well with ATAC-seq.

ChIPseeker requires that the genome of interest be annotated with locations of genes and other features.  The [ChIPseeker user guide](https://bioconductor.org/packages/release/bioc/vignettes/ChIPseeker/inst/doc/ChIPseeker.html) is extremely helpful in using this R/Bioconductor package.


### Motif finding

[HOMER](http://homer.ucsd.edu/homer/introduction/basics.html) is a suite of software designed for [motif discovery](http://homer.ucsd.edu/homer/ngs/peakMotifs.html).  It takes a peak file as input and checks for the enrichment of both known sequence motifs and de novo motifs.



## References <a name="references"></a>

Andrews S. (2010).  FastQC: a quality control tool for high throughput sequence data.  Available online at: http://www.bioinformatics.babraham.ac.uk/projects/fastqc

Buenrostro JD, Giresi PG, Zaba LC, Chang HY, Greenleaf WJ.  Transposition of native chromatin for fast and sensitive epigenomic profiling of open chromatin, DNA-binding proteins and nucleosome position.  Nat Methods. 2013 Dec;10(12):1213-8.

Buenrostro JD, Wu B, Chang HY, Greenleaf WJ.  ATAC-seq: A Method for Assaying Chromatin Accessibility Genome-Wide.  Curr Protoc Mol Biol. 2015 Jan 5;109:21.29.1-9.

Corces MR, Trevino AE, Hamilton EG, Greenside PG, Sinnott-Armstrong NA, Vesuna S, Satpathy AT, Rubin AJ, Montine KS, Wu B, Kathiria A, Cho SW, Mumbach MR, Carter AC, Kasowski M, Orloff LA, Risca VI, Kundaje A, Khavari PA, Montine TJ, Greenleaf WJ, Chang HY.  An improved ATAC-seq protocol reduces background and enables interrogation of frozen tissues.  Nat Methods. 2017 Oct;14(10):959-962.

Heinz S, Benner C, Spann N, Bertolino E, Lin YC, Laslo P, Cheng JX, Murre C, Singh H, Glass CK.  Simple combinations of lineage-determining transcription factors prime cis-regulatory elements required for macrophage and B cell identities.  Mol Cell. 2010 May 28;38(4):576-89.

Langmead B, Salzberg SL.  Fast gapped-read alignment with Bowtie 2.  Nat Methods. 2012 Mar 4;9(4):357-9.

Li H, Handsaker B, Wysoker A, Fennell T, Ruan J, Homer N, Marth G, Abecasis G, Durbin R; 1000 Genome Project Data Processing Subgroup.  The Sequence Alignment/Map format and SAMtools.  Bioinformatics. 2009 Aug 15;25(16):2078-9.

Martin M.  Cutadapt removes adapter sequences from high-throughput sequencing reads.  EMBnet.journal. 2011;17:10-2.

Montefiori L, Hernandez L, Zhang Z, Gilad Y, Ober C, Crawford G, Nobrega M, Jo Sakabe N.  Reducing mitochondrial reads in ATAC-seq using CRISPR/Cas9.  Sci Rep. 2017 May 26;7(1):2451.

Quinlan AR.  BEDTools: The Swiss-Army Tool for Genome Feature Analysis.  Curr Protoc Bioinformatics. 2014 Sep 8;47:11.12.1-34.

Yu G, Wang LG, He QY.  ChIPseeker: an R/Bioconductor package for ChIP peak annotation, comparison and visualization.  Bioinformatics. 2015 Jul 15;31(14):2382-3.
