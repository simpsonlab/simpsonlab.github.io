---
layout: post
title: Experimental methylation-aware polishing
author: jared
draft: false
comments: true
---

Ryan Wick's excellent [nanopore basecaller comparison](https://github.com/rrwick/Basecalling-comparison) highlighted some difficulties of calculating a consensus sequence on bacterial samples that haven't been PCR amplified. His data has a ceiling on consensus accuracy of about 99.7%, which I suggested on [twitter was due to methylation](https://twitter.com/jaredtsimpson/status/914862676605640704) and that we would need a methylation-aware polisher to improve accuracy further. I've just pushed a new version of nanopolish that has _experimental_ support for methylation-aware polishing to try this out. The idea is that when a candidate consensus sequence contains a known methylation motif, like `abcGATCdef`, and the bacteria contains the Dam methyltransferase, we must consider the possibility that the motif is methylated, which changes the properties of the measured signal. Nanopolish will now sum the probability of observing the data from the unmethylated version of the sequence (`abcGATCdef`) with the probability from the methylated version (`abcGMTCdef`) when calculating the consensus. We've trained models for the Dcm and Dam methylases, along with the CpG model we trained for our [human methylation work](http://www.nature.com/nmeth/journal/v14/n4/full/nmeth.4184.html). The training data for the Dam and Dcm models is not ideal however, and the candidate consensus sequence generator may miss some methylation motifs, so this feature is considered experimental. In my hands on a small test data set it improves consensus accuracy by 0.1-0.2%. To turn on methylation-aware polishing add the `--methylation-aware=dcm,dam` option to the `variants` command. If you try it on your own data set, please let me know how it looks.

Implementing methylation-aware polishing required a fairly large refactor of the nanopolish codebase. This refactor exposed a surprising performance bottleneck. Fixing this bottleneck dropped runtime on our benchmark E. coli dataset down to 71 CPU hours. 
