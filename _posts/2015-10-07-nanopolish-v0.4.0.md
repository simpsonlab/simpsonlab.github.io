---
layout: post
title: Nanopolish v0.4.0
author: jared
draft: true
comments: true
---

This is a long overdue post describing recent changes to [nanopolish](https://github.com/jts/nanopolish).

### SQK006 Support ###

Last month Oxford Nanopore released a new sequencing kit, SQK006. As Nick [notes](http://lab.loman.net/2015/09/24/first-sqk-map-006-experiment/) there are two important changes that effect how we model the data. Most importantly, ONT changed their basecaller to use 6-mers rather than 5-mers. The relationship between the DNA sequence in the pore and the measured signals is not simple (to understate the problem). By moving to a signal model with longer context the effect of long-range interactions that are not captured by shorter k-mer is hopefully reduced, which may improve read accuracy. In nanopolish v0.4.0 we now support the 6-mer model. This required reworking our data structures to allow either a 5-mer or 6-mer model depending on what sequencing kit was used to generate the data.

SQK006 also brings an increase in speed - from 30bp/s (SQK005) to 70bp/s (SQK006). With the sampling rate fixed to 3kHz we naturally expect more events to be missed by the event detector. We observed this in our analysis of SQK006 data and updated the initial transition probabilities in our HMM accordingly.
 
#### SQK006 Accuracy ####

We were quite curious to see how well the new data performs in practice. Nick and Josh made four runs with SQK006 kits - two of native E. coli DNA and two of PCR-treated E. coli DNA to remove DNA damage, base modifications and other artefacts. I downloaded one of the native runs and one of the PCR-treated runs and used it to make a new consensus sequence for the draft assembly in our [paper](http://www.nature.com/nmeth/journal/v12/n8/full/nmeth.3444.html). As before we use dnadiff from [mummer](http://mummer.sourceforge.net/) to calculate percent identity, the number of SNPs and the number of indels with respect to the E. coli K12 reference genome. 

| Kit, Coverage     | Percent Identity | # SNPs   | # Indels  |
| ----------------- | ---------------- | ------   | --------  |
|  SQK005, 29X      |          99.48%  | 1,343    | 22,601    |
|  SQK006, 48X      |          99.78%  |   644    |  9,697    |
|  SQK006-PCR, 30X  |          99.82%  |   222    |  8,200    |

The accuracy of our assembly is much better with SQK006 data. Most of this improvement is due to better representation of homopolymer sequences. Interestingly the PCR-treated run gave the best results despite being lower coverage than the native DNA run. It is tempting to attribute this to providing cleaner DNA to the nanopore but this requires further exploration first.

### Eventalign ###

In a previous post I introduced the [eventalign](http://simpsonlab.github.io/2015/04/08/eventalign/) submodule, which uses the hidden Markov model to align events to a reference genome. In nanopolish v0.4.0 there are a few eventalign changes and new features. The bulky `model_name` field has been removed from the output - it bloated the file as every row for a particular read would contain the same information. This read-level information has been moved to a secondary output file which can be enabled with the `--summary FILE` option. The summary file contains the path to the FAST5 file and model name for each read, along with the number of aligned events and other metadata that may be useful for exploring the alignments.

#### Experimental SAM output ####

eventalign now has an optional `--sam` flag which will write the alignment in a modified version of the SAM format. By example:

    example_2d_read.template    0   chr1    10001   60  9950S4M1I1M2I1M1... *   0   0   *   * ES:i:1

Most of these fields should be familiar - the alignment contains a read name (annotated with the sequenced strand) followed by flags, a reference name and the alignment start position. Similar to a base-to-base alignment the CIGAR string describes the alignment of events to reference k-mers. `9950S4M` indicates `9950` events should be skipped. The next event, `9951`, aligns to the k-mer starting at reference position `10001`, followed by three consecutive matches and so on. Sequence and quality information is not stored. The `ES` auxiliary tag is the *event stride* - whether the event indices are increasing along the reference sequence (`ES:i:1`) or whether they are decreasing (`ES:i:-1`). The nanopolish parser for this format is [here](https://github.com/jts/nanopolish/blob/master/src/alignment/nanopolish_alignment_db.cpp#L347).

There are a few reasons for using this format. First, it is much smaller than the eventalign tsv output. Second, by adopting sam/bam instead of creating a new binary format we can use the existing `samtools/htslib` ecosystem. Crucially, we can use `htslib` to provide random access into the alignments for an indexed bam file. This format was developed through discussions with Vadim Zalunin, Ewan Birney, Mick Watson and others. Comments welcome.
