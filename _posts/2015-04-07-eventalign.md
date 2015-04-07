---
layout: post
title: Aligning Nanopore Events to a Reference
author: jared
draft: true
comments: true
---

Introduction
------------

This post describes a new module I added to our [nanopolish](https://github.com/jts/nanopolish) software package that aligns the signal data emitted by a nanopore to a reference genome. This is in contrast to most approaches which align two DNA sequences to each other (for example a base-called read and a reference genome). To make sense of what aligning signal data to a reference genome means, I will describe at a high level my model of how nanopore sequencing works. For a more detailed and technical description, see the supplement of our [preprint](http://biorxiv.org/content/early/2015/03/11/015552).

Nanopore Sequencing
-------------------

A nanopore sequencer threads a single strand of DNA through a pore embedded in a membrane. The pore allows electric current to flow from one side of the membrane to the other. As DNA transits this pore it partially blocks the flow of current, which is measured by the instrument. In Oxford Nanopore's MinION system the measured current depends on the 5-mer that resides in the pore when the measurements are taken.

The MinION samples the current thousands of times per second; as 5-mers slide though the pore they should be observed in multiple samples. The MinION's event detection software processes these samples and tries to detect points where the current level changes. These jumps indicate a new 5-mer resides in the pore. To help illustrate this I've reproduced a figure from our preprint below:

![simulation](/assets/simulation.svg)

This is a simulation from an idealized nanopore sequencing process. The black dots represent the sampled current and the red lines indicate contiguous segments that make up the detected _events_. For example the mean current was around 60 picoamps, plus a bit of noise, for the first 0.5s. The current then dropped to 40 pA for 0.1s before jumping to 52 pA and so on. 

The event detection software writes the events to an HDF5 file. The raw kHz samples are typically not stored as the output files would be impractically large. Here's the table of events for this simulation:

| event index  | mean (pA) | length (s) |
| :----------: | :-------: | :--------: |
|            1 |      60.3 |      0.521 |
|            2 |      40.6 |      0.112 |
|            3 |      52.2 |      0.356 |
|            4 |      54.1 |      0.051 |
|            5 |      61.5 |      0.291 |
|            6 |      72.7 |      0.015 |
|            7 |      49.4 |      0.141 |

To help translate events into a DNA sequence, Oxford Nanopore provides a _pore model_ which describes the expected current signal for each 5-mer. The pore model is a set of 1024 normal distributions - an example might look like this:

| 5-mer  | $$\mu_k$$ | $$\sigma_k$$ |
| :----: | :-------: | :----------: |
| AAAAA  | 53.5      | 1.3          |
| AAAAC  | 54.2      | 0.9          |
| ...    | ...       | ...          |
| TTTTG  | 65.3      | 1.8          |
| TTTTT  | 67.1      | 1.4          |

This indicates that the measured current is expected to be drawn from $$\mathcal{N}(53.5, 1.3^2)$$ when AAAAA is in the pore and so on.

Inferring Bases from Events
---------------------------

Using the pore model and the observed data we can solve a number of inference problems. For example we can infer the sequence of nucleotides that passed through the pore. This is the base calling problem. We can also infer the sequence of the genome given a set of overlapping reads. This is the consensus problem, which we addressed in our paper.

These inference problems are complicated by two important factors. First, the normal distributions for 5-mers overlap. There are 1024 different 5-mers but the signals typically range from about 40-70 pA. This can make it difficult to infer which 5-mer generated a particular event. This is partially mitigated by the structure of the data; a solution must respect the overlap between 5-mers so a position that is difficult to resolve may become clear when we look at subsequent events. Second, event detection is performed in real time and inevitably makes errors. Some events may not be detected if they are too short or if the signals for adjacent 5-mers of the DNA strand are very similar. The extreme case for the latter situation occurs when sequencing through long homopolymers - here we do not expect a detectable change in current. The opposite problem occurs as well. The event detector may split what should be a single event into multiple events due to noise in the system that looks like a change in current. Handling these artefacts is key to accurately inferring the DNA sequence that generated the events.

Aligning Events to a Reference
------------------------------

The hidden Markov model we designed for the consensus problem had 5-mers of an arbitrary sequence as the backbone of the HMM, with additional states and transitions to handle the skipping/splitting artefacts. In this HMM 5-mers emitted events; a path through the HMM describes an alignment between a sequence of events and a sequence of 5-mers. When we made our E. coli assembly we weren't interested in the best alignment so we summed over all paths through the HMM using the forward algorithm. In some situations however we _are_ interested in the alignment of events to 5-mers. If we use the Viterbi algorithm instead of the forward algorithm, and make a reference genome the backbone of our HMM, we can calculate the most probable alignment of events to a reference sequence.

The new ```eventalign``` module of ```nanopolish``` exposes this functionality as a command line tool.  This program takes in a set of nanopore reads aligned in base-space to a reference sequence (or draft genome assembly) and re-aligns the reads in event space.

The pipeline uses ```bwa mem``` alignments as a guide. We start with a normal bwa workflow:

    bwa mem -x ont2d -t 8 ecoli_k12.fasta reads.fa | samtools view -Sb - > alignments.bam
    samtools sort alignments.bam alignments.sorted
    samtools index alignments.sorted.bam

We can then realign in event space                                                   using nanopolish:

    nanopolish eventalign -r reads.fa -b alignments.sorted.bam -g ecoli_k12.fasta "gi|556503834|ref|NC_000913.3|:10000-20000" > eventalign.tsv


The output looks like this:

    contig                         position  reference_kmer  read_index  strand  event_index  event_level_mean  event_length  model_kmer  model_mean  model_stdv
    gi|556503834|ref|NC_000913.3|  10000     ATTGC           1           c       27470        50.57             0.022         ATTGC       50.58       1.02
    gi|556503834|ref|NC_000913.3|  10001     TTGCG           1           c       27471        52.31             0.023         TTGCG       51.68       0.73
    gi|556503834|ref|NC_000913.3|  10001     TTGCG           1           c       27472        53.05             0.056         TTGCG       51.68       0.73
    gi|556503834|ref|NC_000913.3|  10001     TTGCG           1           c       27473        54.56             0.011         TTGCG       51.68       0.73
    gi|556503834|ref|NC_000913.3|  10002     TGCGC           1           c       27474        65.56             0.012         TGCGC       66.96       2.91
    gi|556503834|ref|NC_000913.3|  10002     TGCGC           1           c       27475        69.97             0.071         TGCGC       66.96       2.91
    gi|556503834|ref|NC_000913.3|  10003     GCGCT           1           c       27476        67.11             0.017         GCGCT       68.08       2.20
    gi|556503834|ref|NC_000913.3|  10004     CGCTG           1           c       27477        69.47             0.052         CGCTG       69.84       1.89

This is the complement strand (c) of a 2D Nanopore read from Nick's [E. coli data](http://www.gigasciencejournal.com/content/3/1/22) aligned to E. coli K12. The first event listed (event 27470) had a measured current level of 50.57 pA. It aligns to the reference 5-mer ATTGC at position 10,000 of the reference genome. The pore model indicates that events measured for 5-mer ATTGC should come from $$\mathcal{N}(50.58, 1.02^2)$$, which matches the observed data very well. The next 3 events (27471, 27472, 27473) are all aligned to the same reference 5-mer (TTGCG) indicating that the event detector erroneously called 3 events where only one should have been emitted. Note that the current for these 3 events are all plausibly drawn from the expected distribution $$\mathcal{N}(51.68, 0.73^2)$$.

This output has one row for every event. If a reference 5-mer was skipped, there will be a gap in the output where no signal was observed:

    gi|556503834|ref|NC_000913.3|   10009   GCACC   1       c       27489   67.52   0.028   GCACC   66.83   2.46
    gi|556503834|ref|NC_000913.3|   10011   ACCGC   1       c       27490   65.17   0.012   ACCGC   65.03   1.92

Here we did not observe an event for the 5-mer at position 10010.

This module will hopefully make it easier to work with signal-level nanopore data, and help the development of improved models. The ```eventalign``` module can be found in the latest version of [nanopolish](https://github.com/jts/nanopolish). 
