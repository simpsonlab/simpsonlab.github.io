---
layout: post
title: Packing ONT Fast5 Files
author: matei
draft: true
comments: false
---

### Motivation

The data produced by Oxford Nanopore Technologies (ONT) sequencers is stored in `.fast5` files, based on the [HDF5](https://www.hdfgroup.org/HDF5/) file format, with one file per sequenced read. Fast5 files are (currently) notoriously large. Taking a random sample of 10,000 fast5 files from the [public NA12878 ONT dataset](https://github.com/nanopore-wgs-consortium/NA12878), we observe a ratio of ~340 bytes per sequenced base pair, which translates into ~100 GB per 1X coverage of the human genome. This is almost 2 orders of magnitude larger than the space requirements for storing Illumina sequencing data.

The core of ONT sequencing data consists of electrical current level measurements (in picoamps) taken while biological molecules (DNA, and more recently, RNA) are threaded through a sequencing pore. The process of *basecalling* translates these currents into the more usual base pairs. However, current levels are arguably richer than just base pairs. For example, in [recent work](http://www.nature.com/nmeth/journal/vaop/ncurrent/full/nmeth.4184.html), we demonstrate how current levels (more specifically, event-level data, as detailed below) can be used to detect methylation. For this reason, when trying to reduce the size of fast5 data, it might be desirable to archive current levels in addition to, or independent of, base pairs.

### A New Tool

Motivated by these concerns, we introduce a new tool `f5pack` for the purpose packing fast5 files. This tool is part of the [fast5 library](https://github.com/mateidavid/fast5), more specifically, it is distributed with the Python wrapper of this library. Support for packing is built into the library itself, so any tools that rely on (the updated version of) this library for interacting with fast5 files (e.g. [Nanopolish](https://github.com/jts/nanopolish)) will be able to use packed files transparently.

Using this tool, **we are able to reduce the storage space required by** 10,000 **fast5 files** from the NA12878 dataset **by a factor of ~10**. Here is the detailed log of this packing run:

```
bp_seq_count           92857415                   # number of bases sequenced
rs_count               2483038052                 # number of raw samples
rs_bits                16331772672  6.58  175.88
rs_total_bits          16331772672  6.58  175.88
ed_count               451912927                  # number of eventdetection events
ed_skip_bits           452604512    1.00  4.87
ed_len_bits            3243960992   7.18  34.93
ed_total_bits          3696565504   8.18  39.81
fq_count               95063899                   # number of bases in fastq entries
fq_bp_bits             218731072    2.30  2.36
fq_qv_bits             547456248    5.76  5.90
fq_total_bits          766187320    8.06  8.25
ev_count               369403783                  # number of basecall events
ev_rel_skip_bits       2111901000   5.72  22.74
ev_move_bits           464575320    1.26  5.00
ev_p_model_state_bits  738838088    2.00  7.96
ev_total_bits          3315314408   8.97  35.70
rs_total_duration      620760.76                  # duration of raw samples, in seconds
rs_called_duration     597422.90                  # duration of raw samples that were basecalled
rs_frac_called         0.96
bp_per_sec             155.43                     # average sequencing speed
input_bytes            31950088342                # size of input
output_bytes           3361740258                 # size of output
output_overhead_bytes  348010270                  # size of output, minus bits accounted for above
```

In order to understand how `f5pack` works, and to explain some of the fields above, we need to talk about the specific types of data found in fast5 files.

### Fast5 Data

Conceptually, there are 3 main types of data stored in fast5 files:

- **Raw samples data** consists of raw electrical current measurements taken by the sequencer. This is what travels through the USB/LAN cable connecting the sequencer to a computer. The measurements are usually taken at a rate of 4,000 per second, and each measurement is encoded in fast5 files as a 16-bit integer. Note: fast5 files might use HDF5 internal compression to store data, so it might take less than 16 bits to store each such integer.

  To pack raw samples data, we use the observation that raw samples change rather slowly during sequencing. As such, we compute differences between consecutive raw samples (in integral space), and we encode these differences using a Huffman code based on the predetermined distribution of values, computed from several training files. On this dataset, we achieve a packing rate of 6.58 bits per raw sample (see `rs_bits` above).

- **Event-level data** aggregates raw samples data into *events*, each of which ideally corresponds to a specific DNA context found in the pore. To understand this, first observe that DNA is threaded through the sequencing pore using a biological process at a stochastic rate. Since raw samples are measured at a fixed rate, the number of raw samples corresponding to a specific DNA context is a stochastic quantity. The process of *event detection* takes as input the sequence of raw samples, and it produces a sequence of events. Ideally, each event corresponds to exactly one DNA context, but the process is noisy, so some DNA contexts can correspond to either 0 events (i.e., they are missed) or 2 or more events (i.e., they are over-separated). Fast5 files might contain both *pre-basecall* and *post-basecall* event data.

  Pre-basecall events, found at internal fast5 paths such as `/Analyses/EventDetection_000/Reads/Read_291/Events`, contain the following fields:
  - `start`, `length`: (conceptual) indexes in the raw samples sequence,
  - `mean`, `stdv`: summaries of the corresponding raw samples.
  
  Post-basecall events, found at internal fast5 paths such as `/Analyses/Basecall_1D_000/BaseCalled_template/Events`, contain all the data in pre-basecall events, plus annotations made by the basecalling process such as:
  - `move`: the number of bases that have changed from the DNA context in the previous event,
  - `state`: the DNA context the event corresponds to,
  - `p_model_state`: the probability that `state` is correct.

  In `f5pack`, we encode event-level data as follows:
  - `start` is encoded as `skip`, a difference from previous event end. `skip` is usually 0, but not always, with non-0 values corresponding to instances where the event detection process decided to skip 1 or more raw samples for various reasons (e.g., perhaps they were too error-prone to decode). On this dataset we achieve 1.00 bits per event (see `ed_skip_bits` above).
  - `length` is encoded using a Huffman code based on a predetermined distribution of values, computed from several training files. On this dataset we achieve 7.18 bits per event (see `ed_len_bits` above).
  - `mean` and `stdv` are not encoded at all. Instead, `f5pack` requires the existence of raw samples (packed or not) to pack event-level data, and it recomputes `mean` and `stdv` as part of the unpacking process.
  - When both pre- and post-basecall events are present, for each post-basecall event we also encode `rel_skip`, the relative skip in post-basecall events with respect to the pre-basecall event indexes. This value is 0 if no pre-basecall events are skipped, but it can be non-0 as well. On this dataset we achieve 5.72 bits per event (see `ev_rel_skip_bits` above).
  - `move` is encoded using a Huffman code based on a predetermined distribution of values, computed from several training files. On this dataset we achieve 1.26 bits per event (see `ev_move_bits` above).
  - `state` is not encoded at all. Instead, `f5pack` requires the existence of base-level data (packed or not) to pack event-level data, and it recomputes `state` using `move` as part of the unpacking process.
  - `p_model_state` is a probability, and `f5pack` exposes an option as to how many bits of precision to encode from it, defaulting to 2 bits (see `ev_p_model_state_bits` above).

  In addition to event tables, event-level data also includes 2D alignment data in the case that the sequencing and basecalling are operating in 2D mode. The dataset above is 1D only.

- **Base-level data** is produced by the basecalling process, and it consists of [fastq](https://en.wikipedia.org/wiki/FASTQ_format) entries. For each base pair, the fastq format stores the base itself, and a quality value. We encode these as follows:

  - The bases are encoded using a Huffman code in which we allow for non-ACGT bases. On this dataset, we achieve 2.30 bits per base (see `fq_bp_bits` above).
  - For the base quality value, `f5pack` imposes an upper limit of 31 on the quality value, and it exposes an option as to how many (most significant) bits of precision to encode from it, defaulting to all 5 bits (see `fq_qv_bits` above).

- In addition to these main types of data, fast5 files also contain **configuration data** for various pipelines that the file is run through, as well as **summary data** for the results of these pipelines. Currently, `f5pack` does not save either configuration or summary data, though that could be added as an option if interest arises.

### Future Directions

By inspecting the log of the packing run above, more specifically the third column of the `_bits` fields, we observe that the bulk of the packed data consists of packed raw samples: for every sequenced base pair, we store ~170 bits for its corresponding raw samples, ~75 bits for its event-level data, and only ~8 bits for its fastq entry. Therefore, assuming we want to preserve raw samples for future reproducibility, it is the packing of raw samples that can yield most further improvements. In this sense, there are 2 directions we are currently investigating: using Arithmetic coding instead of Huffman coding to help with power-of-2 rounding issues, and allowing for dynamic codeword maps.

### Command Line Options

- Input: `f5pack` accepts as inputs any combination of: directories, individual fast5 files, or files of fast5 file names. If no input is given, a file of fast5 file names is read from stdin. Input directories are traversed recursively if `--recurse` is given.

- Output: `f5pack` requires an output directory to be specified with `--output`. Output files are all placed in this directory. For every input directory, if `--recurse` is given, the subdirectory structure is recreated in the output directory. Fast5 files specified individually on the input line and those coming from files of fast5 file names are placed in the output directory without any other subdirectories.

- Packing and Unpacking: `f5pack` recognizes 5 types of data: `rs` (raw samples), `ed` (eventdetection events), `fq` (fastq entries), `ev` (basecall events), and `al` (basecall alignments). For each of these, one of the following individual policies can be specified: `drop`, `pack`, `unpack`, and `copy`. If any individual policy is specified, the remaining unspecified ones default to `drop`: `--rs pack` implies `drop` for all other 4 types of data.

  The program will automatically check that unpacking produces reasonable results, and will exit with an error code if that is not the case. This design will catch invalid combinations of policies. For example, `--ev pack --rs drop` is invalid because raw samples are required to unpack basecall events.

- Presets: `f5pack` recognizes the following presets:

  - `--pack`: Pack all data. This is the default if nothing else is specified.
  - `--unpack`: Unpack all data.
  - `--archive`: Pack raw samples only, drop rest.
  - `--fastq`: Pack fastq only, drop rest.

- Tweak packing:

  - `--qv-bits`: Number of (most significant) fastq base quality value bits to store. Default: 5.
  - `--p-model-state-bits`: Number of (most significant) bits to keep from `p_model_state`. Default: 2.

- Errors and Return value: When `f5pack` fails to process (pack/unpack) an input fast5 file, it produces a log message and continues. At the end, `f5pack` exits with an error code if and only if the processing of any of the input fast5 files has failed.

### Examples

```
# drop everything but raw samples
f5pack --archive --output archive.dir/ input.dir/

# unpack raw samples, so that the files can be rerun through Metrichor
f5pack --unpack --output uploaded.new.dir/ archive.dir/

# drop everything other than fastq entries
f5pack --fastq --output fq-only.dir/ input.dir/

# show exactly what is being packed and what is being dropped from a given file
# in terms of internal fast5 paths
f5pack --pack --output pack.dir/ input.dir/file.fast5
f5pack --unpack --output unpack.dir/ pack.dir/file.fast5
diff <(h5dump -n 1 input.dir/file.fast5) <(h5dump -n 1 unpack.dir/file.fast5)
```

### Limitations and Quirks

There are several different existing ONT basecallers: Metrichor, MinKNOW (local), Albacore, Nanonet, and possibly others. Unfortunately, the format for event-level data is not standardized, and as such they each encode it in slightly different ways. We used the first 3, and Metrichor is the only one that produces both pre- and post-basecalling event data, while MinKNOW and Albacore produce only post-basecalling event data. In principle, `f5pack` is designed to work with both pre-and-post and post-only event-level data.

However, current versions of MinKNOW and Albacore both contain a show-stopper issue from the point of view of `f5pack`, in that they do not store enough bits to allow the unambiguous reconstruction of a raw samples index from the event `start` field. We reported this issue to ONT, but pending a design change, **Metrichor is the only basecaller for which we support packing event-level data**. Raw samples and base-level data are not affected by this problem. Currently, when `f5pack` encounters (during a packing run) event-level data written by a basecaller other than Metrichor, it ignores this data (i.e., it `drop`s it), emits a warning, and continues.

Moreover, the current version of Metrichor contains a known (by ONT) bug in the post-basecall events `move` field: sometimes the value stored in fast5 files is wrong. `f5pack` currently works around this issue by trusting the fastq entry and the `state` field more than the `move` field. When the `move` value is detected to be wrong, `f5pack` corrects this value if possible, emits a warning, and continues. If no reasonable correction is available, the packing of that file is considered to have failed.

### Disclaimer

This software is experimental, and it is distributed under the [MIT License](https://github.com/mateidavid/fast5/blob/master/LICENSE). The fast5 format is a moving target. We've tried to ensure that `f5pack` is fault tolerant and safe to use, but please note that you use this software at your own risk.

