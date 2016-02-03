---
layout: post
title: Finding Ebola SNPs
author: jared
draft: false
comments: true
---

Today Nature published our [paper](http://dx.doi.org/10.1038/nature16996) describing the use of the Oxford Nanopore MinION to track the Ebola outbreak in West Africa. Over on Nick Loman's [blog](http://lab.loman.net/2016/02/03/behind-the-paper-real-time-portable-sequencing-for-ebola-surveillance/) you can read an excellent account of how this project started and progressed. For my part, I worked on the method that we used to call SNPs from the sequenced Ebola samples, which I'll describe in this post.

Background
----------

Nanopore sequencing measures the changes in electric current (termed *events*) caused by DNA translocating through the pore. Previously I developed a hidden Markov model for calculating the probability of observing a particular sequence of events given a DNA sequence. We've described this HMM in our genome assembly [paper](http://www.nature.com/nmeth/journal/v12/n8/full/nmeth.3444.html) and in previous blog posts [here](http://simpsonlab.github.io/2015/03/30/optimizing-hmm/) and [here](http://simpsonlab.github.io/2015/04/08/eventalign/). I won't describe the details again but the important thing to understand for this post is that we have a function $$ P(D \; \vert \; S) $$ that calculates the probability of observing some data $$D$$ (measured by the nanopore sequencer) given a sequence $$S$$ (that we want to determine).  In our genome assembly paper we used the HMM to optimize a consensus sequence by iteratively editing $$S$$ until we couldn't make any further improvements.

The first algorithm
-------------------

Last summer Nick and Matt Loose organized a hackathon in Birmingham where I started to work on the SNP calling problem. I thought this would be straightforward as SNP calling is a special case of calculating a consensus sequence. If SNPs are well-separated from each other we can simply calculate the likelihood of a haplotype containing a SNP and compare it to the likelihood of the reference haplotype:

$$P(D \; \vert \; \texttt{...ACGATCGTACA...})$$ (reference)  
$$P(D \; \vert \; \texttt{...ACGATAGTACA...})$$ (snp haplotype)

If $$P(D \; \vert \; \texttt{...ACGATAGTACA...}) > P(D \; \vert \; \texttt{...ACGATCGTACA...})$$ we would call a C to A SNP  and output it in a VCF file with a corresponding quality score. This simple approach worked well for most SNPs in our test case. We observed however that we would miscall regions that had multiple nearby SNPs. Recall from the previous posts that the nanopore sequencer does not measure signals from individual bases; it effectively reads k-mers, not bases. When trying to call nearby SNPs unless we present the complete true haplotype sequence to the algorithm it may either only find a subset of the true SNPs or worse, make false calls. Clearly we had to jointly test SNPs in groups rather than individually.

The failed algorithm
--------------------

I spent the remainder of the hackathon working on an idea that would more generally solve the consensus problem. Let's say we know the true sequence of some partial region of the genome, $$S’$$. Can we determine the base immediately following $$S'$$? On the surface this is simple: we can select the extension with the greatest likelihood out of the four possibilities:

$$P(D \; \vert \; \texttt{S'A})$$  
$$P(D \; \vert \; \texttt{S'C})$$  
$$P(D \; \vert \; \texttt{S'G})$$  
$$P(D \; \vert \; \texttt{S'T})$$

To do this calculation we need to modify our HMM. Previously we required the event sequence to be completely aligned - we knew the first and last event observation for the region of interest. For this problem, we need to relax this constraint and allow uncertainty about the endpoint of the sequence of events. We modified our HMM to handle this by allowing unmatched events at the beginning and end of the matching region, similar to how bases may be soft-clipped in nucleotide alignments.

With the HMM modified we can calculate likelihoods for the one-base extensions of S'. By iterating the procedure we would hopefully arrive at the true sequence of the region including any variants. Selecting the single best extension at each step worked extremely poorly. If an error was made (the wrong extension was selected) the error would propagate and we would have a nonsense reconstruction of the region. I tried various heuristics for keeping track of the N best solutions to avoid forcing the algorithm to take a single choice. This can be incredibly expensive however and I did not find a solution that worked well in all cases. When this algorithm goes wrong we would get a dense cluster of false positive SNPs, which is not a good property of an algorithm. I abandoned this approach shortly after I left the hackathon.

The final algorithm
-------------------

The iterative algorithm which performed reasonably well, but not perfectly, for variant calling ignores an additional source of information: the basecalled sequences of the reads. We decided to find candidate SNPs from the nanopore reads aligned to the reference genome and test combinations of these candidates. We batch SNPs into clusters that are separated by about 10bp, a value I determined that allows the clusters to be treated independently. We then simply generate all possible haplotypes ($$2^n$$ for $$n$$ SNPs) and calculate the likelihood for each haplotype using the HMM. The haplotype with the greatest likelihood is selected as the sequence for the region and its set of SNPs is output in the VCF file.

I sent this version of the algorithm to Nick who tested it on the Mayinga strain of Ebola which has ~500 mutations when compared to the Makona reference (in 19,000 bases, i.e. >2.6% divergence - samples from the West Africa outbreak typically have 20-40 SNPs and are much easier to call). This haplotype-based version was far better than the previous algorithms and forms the basis of the method we used in the paper.

We made a final improvement which used a new 6-mer pore model from Oxford Nanopore. To use this model on our data, which was sequenced with the older SQK005 sequencing kit (and basecalled using 5-mers), we had to calculate a new scaling parameter to transform the events to the 6-mer model. Using 6-mers was more accurate in difficult sequence contexts and completed the algorithm development. The calls from this algorithm are presented in the paper. The SNP caller is implemented in the new “variants” module of [nanopolish](https://github.com/jts/nanopolish).
