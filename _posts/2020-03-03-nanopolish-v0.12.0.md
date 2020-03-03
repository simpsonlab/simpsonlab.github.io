---
layout: post
title: Live methylation calling
author: jared
draft: false
comments: true
---

Nanopolish operates on the raw signal data produced by Oxford Nanopore's sequencers. This requires us to keep track of the signal data for each basecalled read, in a step we call `nanopolish index`. The index makes a map from read name to FAST5 file, for the entire sequencing run. A typical workflow is merging the run's FASTQ files together, mapping the reads to a reference genome, building the index then calling methylation, or polishing a genome. Recently we wanted to see how far we could streamline this workflow and came up with a new nanopolish feature - live methylation calling. In this mode, you can point nanopolish at the run directory for a live sequencing run and it will automatically detect the data as it is produced, align it to a reference genome with minimap2 and call methylation, without any configuration or intervention by the user. For example if your PromethION is writing to `/mounts/nanopore/run_20200303/`, all you have to do is:

```
nanopolish call-methylation -g reference_genome.fasta -t 8 --watch /mounts/nanopore/run_20200303/
```

This command will write out methylation calls for each fast5/fastq pair that it detects (note: live basecalling must be running).

This mode supports multi-threading (`-t`) but to keep up with high throughput runs you might want to point multiple processes against the same sequencing directory. You can do this with the `--watch-process-index`/`--watch-process-count` options. These options tell each process to call methylation for a subset of files. For example if you run these commands on four separate hosts, you can parallelize beyond a single machine:

```
{hpc-node-a}~> nanopolish call-methylation -g reference_genome.fasta -t 8 --watch /mounts/nanopore/run_20200303/ --watch-process-index 0 --watch-process-count 4
{hpc-node-b}~> nanopolish call-methylation -g reference_genome.fasta -t 8 --watch /mounts/nanopore/run_20200303/ --watch-process-index 1 --watch-process-count 4
{hpc-node-c}~> nanopolish call-methylation -g reference_genome.fasta -t 8 --watch /mounts/nanopore/run_20200303/ --watch-process-index 2 --watch-process-count 4
{hpc-node-d}~> nanopolish call-methylation -g reference_genome.fasta -t 8 --watch /mounts/nanopore/run_20200303/ --watch-process-index 3 --watch-process-count 4
```

Running across independent hosts is recommended as it helps avoid the [global lock in libhdf5, as each process will have its own copy of the library](https://twitter.com/Hasindu2008/status/1198150331948924929).

As a bonus, you can provide the `--watch-write-bam` option to save the minimap2 alignments to disk, saving you the step of having to map your data after the run completes. Thanks to Heng Li for [libminimap2](https://github.com/lh3/minimap2/example.c), which made it easy to integrate mapping into nanopolish.

Also in v0.12 is a fixed threshold for methylation calling in CpG dense regions (see issue [here](https://github.com/jts/nanopolish/issues/695)) and a new VCF INFO tag which contains strand bias metrics for each variant call (see [here](https://github.com/jts/nanopolish/issues/634)). Strand bias metrics are very effective at filtering out low quality variant calls, as [Tim Gilpatrick and Winston Timp found](https://www.nature.com/articles/s41587-020-0407-5). We've also lowered the default methylation calling log-likelihood ratio threshold from 2.5 to 2.0, as after extensive testing and feedback we determined a threshold of 2.5 is overly conservative. This will increase the proportion of sites that can be called. Finally, reads with mapping quality less than 20 are no longer used for methylation calling.
