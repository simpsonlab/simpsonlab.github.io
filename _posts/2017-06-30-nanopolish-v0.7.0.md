---
layout: post
title: Nanopolish v0.7.0
author: jared
draft: false
comments: true
---

Many users of nanopolish [have noticed](https://twitter.com/BioMickWatson/status/870014676456927232) that it takes quite a lot of compute time to polish a genome, particularly for very deep sequencing runs that are now common with R9.4 flowcells. I promised on twitter that I would work on that and the first set of optimizations have been pushed to the [nanopolish git repository](http://github.com/jts/nanopolish). In this short post I'll explain where one of the bottlenecks was and how we fixed it.

### Polishing Algorithm ### 

Nanopolish calculates an improved consensus sequence for a draft genome assembly by evaluating candidate edits using a signal-level hidden Markov model. The first stage of the algorithm examines the read alignments to discover where the genome assembly may contain errors. There are two phases to candidate generation. Large edits, like multiple inserted or deleted bases, are harvested from the aligned basecalled reads. We found that single-base edits are often missed in this phase, so a second pass exhaustively tests all possible single base substitutions, insertions and deletions using the HMM. For a genome of length `L` with average depth `d` this will make about `9Ld` calls to the hidden Markov model, which dominates nanopolish's runtime. Often nanopolish only ends up changing 0.2-1.0% of the assembly so most of the candidate edits are rejected. The new version of nanopolish has a smarter candidate testing algorithm that stops evaluating a candidate change when the log-likelihood ratio computed by the HMM reaches a threshold, `t`. This reduces the effect of depth (`d`) on runtime as most candidate edits have very poor log-likelihood ratios and are quickly rejected. 

The new scoring algorithm uses a threshold of `t = 100` by default. For the less patient users there is a more aggressive option (`--faster`, which sets `t = 25`) but I want to test this further before making it the default.

The results of this new method are fairly dramatic on my benchmarking data set (E. coli R9.4 data at 300x depth):

{: .table .table-striped .table-post}
| Version           | Percent Identity | Runtime (CPU hours)  |
| ----------------- | ---------------- | -------------------  |
|  v0.6.3           |          99.95%  |              4323    |
|  v0.7.0           |          99.95%  |               574    |
|  v0.7.0 --faster  |          99.95%  |               424    |

With default parameters 0.7.0 is 7.5x faster than the previous version on this assembly. With the `--faster` flag, it is over 10x faster. We have other optimizations planned so I hope we can reduce runtime even further - more hopefully soon.
