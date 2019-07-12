# ChIP-Seq-pipeline-Flux-
SnakeMake pipeline for ChIP-Seq alignment, filtering, display files, peak calling and motif analysis

This pipeline was originally written by Travis Saari for use on the U-M ARC-TS Flux cluster.

The pipeline can be generally broken down into the following steps:
1.	Alignment
2.	Pre-processing
3.	Results output
a.	Display files
b.	Called Peaks
c.	FindMotifsGenome 

Alignment
bwa-aln / bwa-sampe are used to align each lane’s reads from a given library. In this example, the libraries have been run over four sequencing lanes. Each lane’s resulting sam files are then sorted using picard-tools SortSam, and then the lanes merged into a single bam file, retaining lane information in the readgroup tags.
Note: The use of bwa aln / bwa sampe is a preference which has been carried over from the BEAGle pipeline from the epigenetics group at the BROAD institute. A functionally equivalent snakefile has been created using bwa mem, which can be included in the ATACseq snakefile in-place of the bwa aln snakefile if preferred.
Note: These steps are identical to those in ATAC-seq pipeline (they use the same alignment snakefile)

Pre-processing
The libraries’ bam files are run through picard-tools MarkDuplicates to flag PCR and optical duplicates. These duplicate-marked bam files are then indexed and pruned using samtools. The pruning step is used to only include properly paired reads on the autosomes & sex chromosomes, and exclude unmapped reads, non-primary alignments, PCR/Optical duplicates, & supplementary alignments. 
Note: The above are identical to those in the ATAC-seq pipeline. The next pre-processing steps are unique to the ChIP-seq pipeline.
The pruned bam files are then name-sorted so that pairs of reads are adjacent in the files. This is needed for the next step – X0 pair filtering – which uses a threshold of the X0 bam tag (roughly represents uniqueness of alignment) in each read in the pair, and discards the pair if both reads exceed the threshold.
Note: The X0 pair filtering script is flexible in both the numeric threshold and in how the decision to discard a pair is made (e.g. can also discard pair if only one in the pair exceeds threshold)
These X0-pair-filtered bam files are then sorted by coordinates and indexed using samtools. Once this is complete, we have successfully pre-processed bam files for both the input and the ChIP. Downstream peak-calling and motif-finding steps are performed with the Homer suite, which uses its own TagDirectory inputs for many of its processing steps. Therefore, these pre-processed bams are each fed into Homer MakeTagDirectory.pl to create tag directories required for Homer.

Display file generation
To create BigWig display files, igvtools count is first used to create .wig files from the pre-processed bams. These are fed into UCSC’s wigToBigWig command-line tool to create BigWigs which can be displayed in a genome browser. Note: these steps are also used in the ATAC-seq pipeline.
Note: Igvtools has the option of producing either .wig or .tdf outputs. Keeping igvtools run settings constant, either format should produce equivalent display results.

Peak calling
Homer findPeaks is used for peak-calling. Homer findPeaks uses the tag directories for both the ChIP-seq library and the input library in combination to call peaks on the ChIP-seq experiment. Homer outputs the peak calls in its own .hpeaks format. In order to filter the peaks against blacklist regions, the hpeak file is converted to the standard .bed format using Homer’s pos2bed script. This bed file is then filtered against a blacklist file using bedtools intersect. Finally, the filtered bed file is used to filter the original .hpeaks file using the keepBedEntriesInHpeaks.py script. This allows compatibility with downstream Homer utilities such as findMotifsGenome.
Note: blacklists are sourced from http://mitra.stanford.edu/kundaje/akundaje/release/blacklists/

Motif analysis
findMotifsGenome is run using the called peaks to find over-represented motifs within the peak regions.
