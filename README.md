# Episomizer
Episomizer is currently a semi-automated pipeline for constructing double minutes (aka. episome) 
using WGS data. The challenge to fully automate the entire process lies in the varying 
complexity of genomic rearrangements and inaccurate boundaries of copy number segments that require 
manual review.

Episomizer consists of two major components:
* Bam mining extract the reads around the boundaries of highly amplified genomic regions, 
search for evidence of soft-clipped reads, discordant reads, and bridge reads that support 
putative SVs (aka. edges) between any two segment boundaries. The putative edges are then subject 
to manual review. 
* Composer takes inputs of manually reviewed SVs associated with the segment boundaries together 
with the highly amplified genomic segments, composes the segments to form simple
cycles as candidates of circular DNA structures.

## Prerequisites
* [Perl=5.10.1](https://www.perl.org/)
* [R=3.0.1](https://www.r-project.org/)
* [Samtools=1.7](http://samtools.sourceforge.net/)
* [blat=36](https://genome.ucsc.edu/FAQ/FAQblat)
* [Python=3.6](https://www.python.org/downloads/release/python-360/)
* [NetworkX=2.1](https://networkx.github.io/)
* [pybedtools=0.7.10](https://daler.github.io/pybedtools/#getting-started)

## Installation
Installation is simply cloning of the repository to a working directory and 
adding `$EPISOMIZER_HOME/bin` to `$PATH`.
```
$ EPISOMIZER_HOME=<path_to_working_dir>
$ export PATH=$EPISOMIZER_HOME/bin:$PATH
```

## Usage
```
Usage:
    episomizer <SUBCOMMAND> [args...]
Subcommands:
    create_samtools_cmd    Create samtools command file to extract reads around boundaries of CNA segments
    create_softclip2fa_cmd Create command file to extract softclip reads
    create_blat_cmd        Create command file to blat softclip reads
    SV_softclip            Create read count matrix using softclip reads
    SV_discordant          Create read count matrix using discordant reads
    SV_bridge              Create read count matrix using bridging discordant reads
    matrix2edges           Produce edges to connect SVs based on read count matrices
    composer               Compose segments and edges to form circular double minutes
```
For details on how to run the semi-automated pipeline, see the following [Procedure](#Procedure) section. For a
concrete example of constructing double minutes on a mini-bam, see [examples](./examples/README.md) page.

## Procedure
**Step 1:** Determine a threshold for highly amplified genomic segments based on the empirical distribution
  of Log2Ratio of copy number data.

**Step 2:** Get the putative edges.
1. Generate the shell script with samtools commands to extract the reads around segment boundaries.
    ```
    $ episomizer create_samtools_cmd INPUT_BAM INPUT_CNA_BED OUTPUT_DIR
    ```
    Run the shell script.
    ```
    $ OUTPUT_DIR/run_samtools.sh 
    ```

2. Generate the shell script to extract softclip reads.
    ```
    $ episomizer create_softclip2fa_cmd INPUT_CNA_BED OUTPUT_DIR
    ```
    Run the shell script.
    ```
    $ OUTPUT_DIR/run_softclip2fa.sh
    ```
    
3. Generate the shell script to blat the softclip reads.
    ```
    $ episomizer create_blat_cmd REF_GENOME_BIT INPUT_CNA_BED OUTPUT_DIR
    ```
    The reference genome GRCh37-lite.2bit can be downloaded from 
    [St. Jude public FTP site](http://ftp.stjude.org/pub/software/cis-x/GRCh37-lite.2bit).
    
    Run the shell script (recommend parallel processing).
    ```
    $ OUTPUT_DIR/run_BLAT.sh
    ```
    
 4. Create 3 read count matrices using softclip reads, discordant reads and bridging discordant reads.
    ```
    $ episomizer SV_softclip INPUT_CNA_BED FLANK SOFTCLIP_BLAT_DIR OUTPUT_DIR
    $ episomizer SV_discordant INPUT_CNA_BED TLEN FLANK BOUNDARY_READS_DIR OUTPUT_DIR
    $ episomizer SV_bridge INPUT_CNA_BED TLEN DISTANCE BOUNDARY_READS_DIR OUTPUT_DIR
    ```
    
 5. Produce edges to connect SVs based on read count matrices.
    ```
    $ episomizer matrix2edges INPUT_CNA_BED MATRIX_FILE OUTPUT_EDGE_FILE
    ```
    
**Step 3:** Manually review the putative edges.

Putative edges generated by above steps are carefully evaluated. The manual review process 
uses the extracted boundary reads and their Blat results in the “trace” folder, and also involves examining the 
segments’ coverage using IGV to refine segment boundaries when necessary. For all 3 types of putative edges (softclip, 
discordant, and bridge), they were sorted by the total number of supporting reads from high to low. Edges with only
a few reads support on each side are usually spurious or indicate minor clones which we do not consider in our 
current study.

For the softclip edges, the edges that represent the adjacent segment are marked first. Then for each of the rest 
SVs, the Blat output is manually checked to figure out the orientations of the two connecting segments so that we 
can identify the true edge and remove the accompanying false edges. 

For the discordant edges, the edges that are already reviewed in the softclip edges are removed. Then for the 
rest of the edges, we manually check the boundary reads, especially the FLAG column in SAM format, to figure out 
the orientations of the the two connecting segments so that we identify the true edge and remove the accompanying 
false edges.

For the bridge edges, all the edges previously reviewed in the softclip edges and discordant edges are removed.
Then for the rest of the edges, we manually check the boundary reads to see if the shared region matches the 
orientations of the reported two segments (i.e. check the FLAG on both ends to see if they match the orientations 
of three segments). Also, if the reads support from both sides are off balance (for example, AtoB is 1 but BtoA 
is 100), the edge is most likely to be spurious.

**Step 4:** Compose circular double minute structures.
```
$ episomizer composer circ -c REVIEWED_SEGMENTS -l REVIEWED_EDGES -d OUTPUT_DOUBLE_MINUTES
```

## Notes
It's recommended to follow the detailed tutorial in the "examples" folder and use the "testdata" folder to reproduce 
our findings on the relapse sample from a pediatric HGG patient.  


## Maintainers
* [Liang Ding](https://github.com/adamdingliang)
* [Ke Xu](https://github.com/FromSoSimple)