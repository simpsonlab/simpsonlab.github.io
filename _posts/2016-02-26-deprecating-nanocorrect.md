---
layout: post
title: Deprecating Nanocorrect
author: jared
draft: true
comments: true
---

When Nick, Josh and I got together at the Newton Institute [hackathon](2015-03-30-optimizing-hmm.md) in 2014 we wanted to see if assembling nanopore data was possible. Over the week we put together a pipeline inspired by [pbdagcon](https://github.com/PacificBiosciences/pbdagcon) which used [DALIGNER](https://github.com/thegenemyers/DALIGNER) and [poa](http://sourceforge.net/projects/poamsa/) to correct sequencing errors in nanopore reads. This software, which we called [nanocorrect](https://github.com/jts/nanocorrect), improved the accuracy of our nanopore reads to around 97\% after two rounds of correction. The corrected reads worked well in the Celera Assembler and we had very long contigs soon after the hackathon.

To complete our de novo assembly paper we wrote a second, much more powerful, software package called [nanopolish](https://github.com/jts/nanopolish) that uses the nanopore signal data to improve the accuracy of the assembly. Nanopolish has since grown to support [aligning signal events to a reference genome](2015-04-08-eventalign.md) and [calling SNPs](2016-02-03-ebola-snps.md). Where nanocorrect was a simple, quick solution to error correction, nanopolish is a long term project that I've spent most of my time on in the last year.

Nanocorrect's major flaw is that it is very slow (specifically the poa step which constructs a partial order alignment graph between sets of overlapping reads). In the year since we published our paper a few other nanopore assemblers has appeared, namely [Canu](http://canu.readthedocs.org/en/latest/quick-start.html) and [miniasm](http://arxiv.org/abs/1512.01801). These programs are clearly better than our nanocorrect + CA pipeline so we've decided to deprecate nanocorrect. Our current suggestion is to start by building contigs with canu and follow up with nanopolish to compute the consensus sequence. 

Currently nanopolish requires about 250 CPU hours to polish the E. coli genome from 30X data. We have ideas to improve this using the lessons we've learned from calling SNPs on Ebola. This will hopefully be the subject of a new post in the next few months.
