---
layout: post
title: Approximate Mapping of Nanopore Squiggle Data with Spatial Indexing
author: jonathan
draft: true
comments: true
---

### Nanopore data

Here at the Simpsonlab we work with signal-level data from nanopore
sequencers, particularly those from ONT like the MinION.  Signal-level
data looks something like this:

![Idealized Example Squiggle Plot](/assets/kdtreemapping/squiggle.png)

where the data comes in as a stream of events (the red lines) with
means, standard deviations, and durations, with each black point
being an individual signal reading.  We use the signal-level data
partly for accuracy - because converting each red line and dozens
of black basecalling into one of four letters invariably loses lots
of information - and partly because those basecalls might not be
available in, say, the field, where the portability of the nanopore
sequencers really shines.

This data has to be interpreted in terms of a particular model of
the interaction of the pore and the strand of DNA; such a pore model
gives, amongst other things, an expected signal mean and standard
deviation for currents through the pore when a given $$k$$mer is in
the pore (for the MinION devices, pore models typically use $$k$$=5 or
6).  So a pore model might be, for k=5, a table with 1024 entries
that looks something like this:


| kmer  | mean  |  std dev  |
| ----- | ----- | --------- |
| AAAAA | 70.24 | &plusmn; 0.95 pA |
| AAAAC | 66.13 | &plusmn; 0.75 pA |
| AAAAG | 70.23 | &plusmn; 0.76 pA |
| AAAAT | 69.25 | &plusmn; 0.68 pA |
| ...   | ...   | ...       |

and using such a pore model it&#8217;s fairly straightforward to
go from a sequence to an expected set of signal levels from the
sequencer: so, say, for the 10-base sequence AAAAACGTCC and the
above 5-mer pore model we&#8217;d expect 6 events (10-5+1) that look like:

| event  |  kmer | mean     | range (1$$\sigma$$)|
| ------ | ----- | -------- | ---------------- |
| *e1* | AAAAA |  70.2 pA |  69.3 -  71.2 pA |
| *e2* | AAAAC |  66.1 pA |  65.4 -  66.9 pA |
| *e3* | AAACG |  62.8 pA |  62.0 -  63.6 pA |
| *e4* | AACGT |  67.6 pA |  66.3 -  68.9 pA |
| *e5* | ACGTC |  56.6 pA |  54.5 -  58.7 pA |
| *e6* | CGTCC |  49.9 pA |  49.1 -  50.7 pA |

Using those models, plus HMMs and a tonne of math, Jared&#8217;s
[nanopolish](https://github.com/jts/nanopolish) tool has had enormous
success improving the quality of nanopore reads and assemblies of
nanopore data.  As one can see from the &lsquo;range&rsquo; column above, and
from the histogram on the side of the first plot, which shows the
numbers of kmers which fall into 1pA bins in the pore model, this
isn&#8217;t necessarily particularly easy.  There are over 70 kmers which
are expected to have means in the range 69.5pA&nbsp;-&nbsp;70.5pA;
that plus the wide standard deviations means that there is enormous
ambiguity going back from signal levels to sequence.

### Mapping

This ambiguity introduces complication into another part of the
bioinformatics pipeline - mapping.  While several mappers (BWA MEM,
LAST) work quite well with basecalled ONT data, and one mapper[^1]
has been written specifically for basecalled ONT data, mapping
signal-level data directly is a little tougher.  It is one thing
for a seed sequence in the read to occur in multiple locations in
the reference index; it is another thing for a sequence of
floating-point currents to plausibly represent one of hundreds of
seed sequences.

On the other hand, a flurry of recent important papers and software[^2]
[^3] [^4], discussed in some informative
[blog](http://robpatro.com/blog/?p=248)
[posts](https://liorpachter.wordpress.com/2015/11/01/what-is-a-read-mapping/),
have demonstrated the usefulness of approximate read mapping.  In
our case, mapping to an approximate region of a reference would
allow exact but computationally expensive methods like `nanopolish
eventalign` to produce precise alignments, or simple assignment may
be all that is necessary for other applications.

So an output approximate mapping would be valuable, but the issues remains of how to disambiguate the signal level values.  

### Spatial Indexing

Importantly, while any individual event is very difficult to assign
with precision, sequences of events are less ambiguous, as any given
event is followed, with high probability, by one of only four other
events.  Thus, by examining tuples of events, one can greatly
increase specificity.

And there are well-developed sets of techniques for looking up lists
of $$d$$ floating point values to look for candidate matches: [spatial
indexes](https://en.wikipedia.org/wiki/Spatial_database), where
each query or match is considered as a point in $$d$$-dimensional
space, and the goal is to return either some number of nearest
points (k-Nearest-Neighbours, or [kNN queries](https://en.wikipedia.org/wiki/K-nearest_neighbors_algorithm)), or
all matches within some, typically euclidean, distance.

To see how this would work, consider indexing the sequence above,
AAAAACGTCC, in a spatial index with $$d=2$$.  There are a total of 6
events, so wed&#8217; have 5 points (6-2+1), each points in 2-dimensional
space; 4 are plotted below (the other falls out of the range of the
plot)

![2D Spatial Query](/assets/kdtreemapping/2d-spatial-query.png)

and a query for $$d$$ read events that could be matched by (69.5, 67), the blue point, would return the nearest match (70.2, 66.1) corresponding to the $$k+d-1$$-mer AAAAAC.

So to map event data, one can:

* Generate an index:
    * Generate all overlapping tuples of $$d$$ kmers from a reference
    * Using a pore model, convert those into points in $$d$$-space
    * Create a spatial index of those points, 
* For every read:
    * Take every overlapping set of $$d$$ events
    * Look it up in a spatial index
    * Find a most likely starting point for the read in the reference, with a quality score.

Note that one has to explicitly index both the forward and reverse
strands of the reference, since you don&#8217;t _a priori_ know
what the &ldquo;reverse complement&rdquo; of 65.5&nbsp;pA is.  One
also has to generate multiple indices; for 2D ONT reads, there is
a pore model for the template strand of a read, and typically two
possible models to choose from for the complement strand of a read,
so one needs three indexes in total to query.


### What dimension to choose?

To test this approach, you have to choose a $$d$$ to use.  On the one
hand, you would like to choose as large a dimension as you can; the
larger $$k+d-1$$ is, the more unique each seed is and the fewer places
it will occur in any reference sequence.

On the other hand, two quite different constraints put a strong upper limit on the dimensions that will be useful:

* The [curse of dimensionality](https://en.wikipedia.org/wiki/Curse_of_dimensionality) -
in high spatial dimensions, nearest-neighbour searching is extremely
inefficient, because distance loses its discriminative power.  (Most
things are &ldquo;close&rdquo; to each other in 100-dimensional
space; there are a lot of routes you can take!)  As a practical
matter, for most spatial index implementations, for $$d > 15-20$$ or
so you might as well just do a linear search over all possible
items;
* Current approaches to segmentation mean that ONT data has very
frequent &lsquo;stops&rsquo; and &lsquo;skips&rsquo; - that is,
events are spuriously either inserted or deleted from the data.
Exactly as with noisy basecalled sequences, this strongly limits
the length of sequence that can be usefully used as seeds.  As we
see below for one set of _E. coli_ data, there is probably not much
point in using $$d \ge 10$$ even for template-strand data, as the
median &ldquo;dmer&rdquo; will have an insertion/deletion in it.

![Distribution of lengths of continuous move events](/assets/kdtreemapping/events-per-stopskip.png)

For these two reasons, we&#8217;ve been playing with $$d = 7-8$$.  Il&#8217;l
note that while increasing $$d$$ is the most obvious knob to turn to
increase specificity of the match, higher $$k$$ helps as well.

### Normalizing the signal levels

One issue we haven&#8217;t mentioned is that the raw nanopore data needs
calibration to be compared to the model; there is an additive shift
and a multiplicative scale that has to be taken into account.  (There
is also a drift over time, which is typically not important for
single reads but matters if one is comparing across reads).

A very simple &ldquo;methods of moments&rdquo; calculation works
surprisingly well for long-enough reads, certainly well enough to
start an iterative process; for any given model one is trying to
fit a read to, rescaling the mean and standard deviation of read
events to model events gives a good starting point for calibration.

![Simple initial rescaling of current values by method of moments](/assets/kdtreemapping/initial-rescale.png)

### Proof of concept

A simple proof of concept of using spatial indexing to approxmiately map squiggle data can be found [on github](https://github.com/ljdursi/simple-squiggle-pseudomapper).  It&#8217;s written in python, and has `scipy` and `h5py` as dependencies.

As a spatial index, it uses a version of a [k-d tree](https://en.wikipedia.org/wiki/K-d_tree) (`scipy.spatial.cKDTree`), which is a very versatile and widely used (and so well-optimized) spatial index widely used in machine learning methods amongst others; different structures may have advantages for this application.

Running the `index-and-map.sh` script generates an index for the provided `ecoli.fa` reference - about 1 minute per pore model - and then maps the 10 reads provided of both older 5mer and newer 6mer MAP data.  Mapping each read takes about 6 seconds per read per pore model; this involves lots of python list manipulations so could fairly straightforwardly be made much faster.  

Even doing the simplest thing possible for mapping works surprisingly well.  Using the same sort of approach as the first steps of the Sovic _et al._ method[^1], we just:

* Use the default k-d tree parameters (which almost certainly isn&#8217;t right, particularly the distance measure) 
* Consider bins of starting positions on the reference, of size ~10,000bp, a typical read size 
* For each $$d$$-point in the read,
    * Take the closest match to each $$d$$-point (or all within some distance) 
    * For each match, add a score to the bin corresponding to the implied starting position of the read on the reference; a higher score for a closer match
* Report the best match starting point

And even this is enough to reach something like 95% mapping accuracy
on the full set of 5mer ecoli data (here an accurate map is considered
to be within a couple of bins of the BWA MEM location mapped from
the basecalled results; within that range, a number of finishing
procedures could efficiently get a precise alignment if needed).

With a few other tweaks - keeping track of the top 10 candidates,
and for each re-testing by recalibrating given the inferred mapping
and rescoring - we get 99% accuracy on the 5mer data - the 6mer
data is more problematic because of higher skip/stop rates.

![kd-tree approximate mapping vs BWA MEM mapping positions](/assets/kdtreemapping/dotplot.png)

Of course, while 95-99% (Q13-Q20) mapping accuracy on _E. coli_ is
a cute outcome from such a simple approach, it isn&#8217t; nearly
enough; with $$d=8$$ and $$k=6$$, wer&#8217;e working with seeds of
size 14, which would typically be unique in the _E. coli_ reference,
but certainly wouldn&#8217;t be in the human genome, or for metageomic
applications.

To improve accuracy, we need to go further than just summing scores
of individual seed matches, about which we plan to write more shortly.

---

### References

[^1]: [Fast and sensitive mapping of error-prone nanopore sequencing reads with GraphMap](http://biorxiv.org/content/early/2015/06/10/020719) (2015) by Sovic, Sikic, Wilm, _et al._

[^2]: [Near-optimal RNA-Seq quantification](http://arxiv.org/abs/1505.02710) (2015) by Bray, Pimentel, Melsted, and Pachter

[^3]: [Salmon: Accurate, Versatile and Ultrafast Quantification from RNA-seq Data using Lightweight-Alignment](http://biorxiv.org/content/early/2015/06/27/021592) (2015) by Patro, Duggal, and Kingsford

[^4]: [RapMap](https://github.com/COMBINE-lab/RapMap), COMBINE lab
