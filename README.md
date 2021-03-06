# Assembler Components

Components of genome sequence assembly tools. 

# Rationale

Genome, metagenome and transcriptome assemblers range from fully integrated to fully modular. Fully modular assembly has a number of benefits. This repository is ongoing work to define some important checkpoints in a modular assembly pipeline, along with standard input/output formats. For now we have a bias towards Illumina-type sequencing data (single reads, paired reads, mate-pairs, 10x), but we aim to make the components also compatible with 3rd generation reads. 

Feel free to contribute via pull-requests.

# File formats

- FASTQ reads
- FASTA contigs
- [GFA](http://gfa-spec.github.io/GFA-spec/) assembly graph
- [SAM/BAM](http://samtools.github.io/hts-specs/) or [PAF](https://github.com/lh3/miniasm/blob/master/PAF.md) alignments of reads to draft assembly

**FASTA/FASTQ**: Optional attributes are found in the comment field, and formatted as in SAM, `XX:x:xxxx`. The comment field follows a space in the header.

**FASTQ**: Paired-end reads are interleaved and may be compressed. Linked read barcodes may be indicated with the `BX` tag.

**FASTA**: The sequence should not be line wrapped. The FASTA file should be indexed (FAI).

**GFA**: GFA2 is preferred over GFA1. The sequence fields may be empty (`*`). The sequences may be stored in an adjacent FASTA file, named for example `assembly.gfa` and `assembly.fa`.

**SAM/BAM**: SAM/BAM files are sorted by position and indexed, unless otherwise stated. Different sequencing libraries may be indicated with the read group `RG` attribute.

## GFA record types

- GFA (S): sequence segments
- GFA (E): overlap edges
- GFA (G): gap edges
- GFA (U): unordered groups of sequence segments
- GFA (O): ordered paths of sequence segments

## GFA sequence segment attributes

- GFA (S[RC]): read counts
- GFA (S[CN]): copy number estimate

## File names

An assembly is contained in a single directory. The files are named according to the pattern `[0-9]+_[a-z]+\.[a-z.]+`. The numeric prefixes are zero-padded and identical in length, and they indicate the stage of the assembly. A descriptive name and file type extension follow. The files may be compressed.

### Example

```
0_pe.fq.gz
1_unitig.gfa
2_denoise.gfa
3_debulge.gfa 3_debulge.bam 3_debulge.bam.bai
4_link.gfa
5_scaffold.gfa
6_assembly.gfa 6_assembly.fa 6_assembly.fa.fai 6_assembly.bam 6_assembly.bam.bai
```

# Notation

type1(record[attributes],&hellip;) + &hellip; &rarr; type2(record[attributes],&hellip;) + &hellip;

This stage requires a file of type1 and produces a file of type2. For example, estimate copy number of unitigs. A GFA file of sequence segments and edges and a BAM or PAF file of mapped reads produces a GFA file with estimated copy numbers of unitigs.

GFA (SE) + BAM/PAF &rarr; GFA (S[CN],E)

# Tools

- [ABySS](https://github.com/bcgsc/abyss#readme)
- [BCALM2](https://github.com/GATB/bcalm#readme)
- [BCOOL](https://github.com/Malfoy/BCOOL#readme)
- [BFC](https://github.com/lh3/bfc#readme)
- [lh3/gfa1](https://github.com/lh3/gfa1#readme)
- [SGA](https://github.com/jts/sga#readme)
- [Unicycler](https://github.com/rrwick/Unicycler#readme)

A tool may combine multiple assembly stages in a single tool.

# Stages of genome assembly

## Preprocess reads

Remove sequencing artifacts specific to each sequencing technology. And overall, improve the quality of input reads with minimal loss of information (e.g. variants).

FASTQ &rarr; FASTQ

- Trim adapter sequences
- Split chimeric reads
- Merge overlapping paired-end reads
- Extract barcode sequences

Tools are specific to each sequencing technology and numerous and so will not be listed here.

- Correct sequencing errors in reads
     - BFC `bfc`
     - BCOOL `Bcool.py`
     - SGA `sga index | sga correct`

## Unitig

Assemble unitigs by de Bruijn graph assembly.

FASTQ &rarr; GFA (SE)

- Count *k*-mers
- Filter *k*-mers by abundance
- Compact the graph

Tools:

- ABySS `ABYSS` or `ABYSS-P` or `abyss-bloom-dbg` then `AdjList` or `abyss-overlap`
- BCALM2  `bcalm | convertToGFA.py`
- SGA `sga index | sga filter | sga overlap | sga assemble`


## Denoise

Remove exclusively sequencing errors from the graph, keep variants.

GFA (SE) &rarr; GFA (SE)

- Prune tips
- Collapse bulges due to sequencing errors

Tools:

- ABySS `abyss-filtergraph`
- lh3/gfa1 `gfaview -t`

## Simplification

Identify and/or remove variants from the graph.

GFA (SE) &rarr; GFA (SE)

- Identify bulges: GFA (SE) &rarr; GFA (SEU)
- Collapse bulges: GFA (SEU) &rarr; GFA (SE)

Collapsed bulges may select a single sequence or may compute a consensus sequence, using IUPAC ambiguity codes.

Tools:

- ABySS `PopBubbles | MergeContigs`
- lh3/gfa1 `gfaview -b`

## Repeat-resolution

### Read mapping

Align reads to the assembly graph.

FASTQ + FASTA &rarr; FASTQ + FASTA + BAM

Tools:

- Unicycler `unicycler_align`
- ABySS `abyss-map`
- BWA `bwa mem -p`
- Minimap2 `minimap2 -xsr` for short reads
- BGREAT2 `bgreat`

### Read threading

Resolve repeats in the graph

FASTQ + GFA (SE) + FASTQ &rarr; FASTQ + GFA (SE) + PAF

Tool:

- ???

### Estimate copy number

Estimate the copy number of each unitig.

GFA (SE) + BAM/PAF &rarr; GFA (S[CN],E)

- Count reads per unitig: GFA (SE) + BAM/PAF &rarr; GFA (S[RC],E)
- Estimate copy number: GFA (S[RC],E) &rarr; GFA (S[CN],E)

Tools:

- SGA `sga-astat.py`

## Scaffold

Scaffolding is the combination of the three stages of link unitigs, order and orient, and contract paths.

FASTA/GFA(S) + BAM/PAF &rarr; FASTA/GFA(S)

### Link unitigs

Identify pairs of unitigs that are proximal. Estimate their relative order, orientation, and distance.

GFA (SE) + BAM/PAF &rarr; GFA (SEG)

Tools:

- ABySS `abyss-fixmate | DistanceEst` for paired-end and mate-pair reads
- ABySS `abyss-longseqdist` for long reads
- ARCS `arcs` for linked reads

### Order and orient

Order and orient paths of unitigs. Contigs are created by contiguous paths of sequence segments. Scaffolds are created by discontiguous paths of sequence segments.

GFA (SEG) &rarr; GFA (SEO)

Tools:

- ABySS `abyss-scaffold` or `SimpleGraph | MergePaths`
- SGA `sga scaffold`
- BESST `runBESST`
- and so many other scaffolders that take a BAM file as input..

### Contract paths

Glue vertices of paths and replace each path with a single sequence segment.

GFA (SEO) &rarr; GFA (SE)

Tools:

- ABySS `MergeContigs`
- SGA `sga scaffold2fasta`

# Pipelines

## ABySS

Assemble paired-end reads using ABySS. This ABySS pipeline is a minimal subset of tools run by the complete `abyss-pe` pipeline.

This data is from Unicycler: "These are synthetic reads from plasmids A, B and E from the Shigella sonnei 53G genome assembly". [Shigella sonnei plasmids (synthetic reads)](https://github.com/rrwick/Unicycler/tree/master/sample_data#shigella-sonnei-plasmids-synthetic-reads), [short_reads_1.fastq.gz](https://github.com/rrwick/Unicycler/raw/master/sample_data/short_reads_1.fastq.gz), [short_reads_2.fastq.gz](https://github.com/rrwick/Unicycler/raw/master/sample_data/short_reads_2.fastq.gz)

```sh
# Install the dependencies
brew install abyss curl pigz samtools seqtk
# Download the data
seqtk mergepe <(curl -Ls https://github.com/rrwick/Unicycler/raw/master/sample_data/short_reads_1.fastq.gz) <(curl -Ls https://github.com/rrwick/Unicycler/raw/master/sample_data/short_reads_2.fastq.gz) | gzip >0_pe.fq.gz
# Unitig
gunzip -c 0_pe.fq.gz | ABYSS -k100 -t0 -c0 -b0 -o 1_unitig.fa -
AdjList --gfa2 -k100 1_unitig.fa >1_unitig.gfa 
# Denoise
abyss-filtergraph --gfa2 -k100 -t200 -c3 -g 2_denoise.gfa 1_unitig.gfa
# Collapse bulges
PopBubbles --gfa2 -k100 -p0.99 -g 3_debulge.gfa 1_unitig.fa 2_denoise.gfa >3_debulge.path
MergeContigs --gfa2 -k100 -g 3_debulge.gfa -o 3_debulge.fa 1_unitig.fa 2_denoise.gfa 3_debulge.path
# Map reads
gunzip -c 0_pe.fq.gz | abyss-map - 3_debulge.fa | pigz >3_debulge.sam.gz
# Link unitigs
gunzip -c 3_debulge.sam.gz | abyss-fixmate -h 4_link.tsv | samtools sort -Osam | DistanceEst --dot -k100 -s500 -n1 4_link.tsv >4_link.gv
# Order and orient
abyss-scaffold -k100 -s500-1000 -n5-10 3_debulge.gfa 4_link.gv >5_order.path
# Contract paths
MergeContigs --gfa2 -k100 -g 6_assembly.gfa -o 6_assembly.fa 3_debulge.fa 3_debulge.gfa 5_order.path
# Compute assembly metrics
abyss-fac 6_assembly.fa
# Convert GFA2 to GFA1
abyss-todot --gfa1 6_assembly.gfa >6_assembly.gfa1
# Visualize assembly graph using Bandage
Bandage load 6_assembly.gfa1 &
```
