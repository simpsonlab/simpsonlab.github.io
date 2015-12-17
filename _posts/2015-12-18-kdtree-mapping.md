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
[^3] [^4] [^5], the first of which are discussed in some informative
[blog](http://robpatro.com/blog/?p=248)
[posts](https://liorpachter.wordpress.com/2015/11/01/what-is-a-read-mapping/),
have demonstrated the usefulness of approximate read mapping.  In
our case, mapping to an approximate region of a reference would
allow exact but computationally expensive methods like `nanopolish
eventalign` to produce precise alignments, or simple assignment may
be all that is necessary for other applications.

So an output approximate mapping would be valuable, but the issues
remains of how to disambiguate the signal level values.

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

<img src="/assets/kdtreemapping/2d-spatial-query.png" alt="2D Spatial Query" style="width: 478px; margin-left:auto; margin-right:auto;"/>

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

<img src="/assets/kdtreemapping/events-per-stopskip.png" alt="Distribution of lengths of continuous move events" style="width: 550px; margin-left:auto; margin-right:auto;"/>

For these two reasons, we&#8217;ve been playing with $$d \approx
8$$.  I&#8217;ll note that while increasing $$d$$ is the most obvious
knob to turn to increase specificity of the match, higher $$k$$
helps as well.

### Normalizing the signal levels

One issue we haven&#8217;t mentioned is that the raw nanopore data needs
calibration to be compared to the model; there is an additive shift and dirft over time,
and a multiplicative scale that has to be taken into account.  

A very simple &ldquo;methods of moments&rdquo; calculation works
surprisingly well for long-enough reads, certainly well enough to
start an iterative process; for any given model one is trying to
fit a read to, rescaling the mean and standard deviation of read
events to model events gives a good starting point for calibration,
and drift is initialyy ignored.

![Simple initial rescaling of current values by method of moments](/assets/kdtreemapping/initial-rescale.png)

### Proof of concept

A simple proof of concept of using spatial indexing to approxmiately map squiggle data can be found [on github](https://github.com/ljdursi/simple-squiggle-pseudomapper).  It&#8217;s written in python, and has `scipy`, `h5py`, and `matplotlib` as dependencies.  Note that as implemented, it is absurdly memory-hungry, and definitely requires a numpy and scipy built against a good BLAS implementation.

As a spatial index, it uses a version of a [k-d tree](https://en.wikipedia.org/wiki/K-d_tree) (`scipy.spatial.cKDTree`), which is a very versatile and widely used (and so well-optimized) spatial index widely used in machine learning methods amongst others; different structures may have advantages for this application.

Running the `index-and-map.sh` script generates an index for the provided `ecoli.fa` reference - about 1 minute per pore model - and then maps the 10 reads provided of both older 5mer and newer 6mer MAP data.  Mapping each read takes about 6 seconds per read per pore model; this involves lots of python list manipulations so could fairly straightforwardly be made much faster.  
Doing the simplest thing possible for mapping works surprisingly well.  Using the same sort of approach as the first steps of the Sovic _et al._ method[^1], we just:

* Use the default k-d tree parameters (which almost certainly isn&#8217;t right, particularly the distance measure) 
* Consider overlapping bins of starting positions on the reference, of size ~15,000bp, a typical read size 
* For each $$d$$-point in the read,
    * Take the closest match to each $$d$$-point (or all within some distance) 
    * For each match, add a score to the bin corresponding to the implied starting position of the read on the reference; a higher score for a closer match
* Report the best match starting point

Let&#8217;s take a look at the initial output for the older SQK005 ecoli data and newer SQK006 data:
{% highlight bash %}
$ ./index-and-map.sh

5mer data: 

Indexing : model models/5mer/template.model, dimension 8
python spatialindex.py --dimension 8 ecoli.fa models/5mer/template.model indices/ecoli-5mer-template

real    0m53.685s
user    0m48.460s
sys 0m1.092s
Mapping reads: starting with ecoli/005/LomanLabz_PC_Ecoli_K12_R7.3_2549_1_ch182_file148_strand.fast5
time python ./mapread.py --plot save --plotdir plots --closest   --maxdist 3  --templateindex indices/ecoli-5mer-template.kdtidx \
     ecoli/005/LomanLabz_PC_Ecoli_K12_R7.3_2549_1_ch182_file148_strand.fast5 ...  > template-only-005.txt

real    1m9.141s
user    1m6.236s
sys 0m0.612s
Template Only Alignments
Read           Difference  BWA      KDTree   zscore
ch401_file98   19          2328329  2328348  6.160000
ch34_file53    228         1128120  1128348  11.235000
ch464_file15   2031        2195379  2193348  14.782000
ch182_file148  2441        2767555  2769996  17.120000
ch461_file9    3105        2195243  2198348  8.749000
ch277_file143  5819        3079177  3084996  17.164000
ch498_file171  6130        3793866  3799996  11.916000
ch222_file28   6765        2803231  2809996  11.152000
ch80_file64    10649       4369347  4379996  7.469000
ch395_file89   10828       2069168  2079996  9.517000
Indexing : model models/5mer/complement.model, dimension 8
python spatialindex.py --dimension 8 ecoli.fa models/5mer/complement.model indices/ecoli-5mer-complement

real    0m50.793s
user    0m47.380s
sys 0m0.840s
Mapping reads: starting with ecoli/005/LomanLabz_PC_Ecoli_K12_R7.3_2549_1_ch182_file148_strand.fast5
time python ./mapread.py --plot save --plotdir plots --closest   --maxdist 3  --templateindex indices/ecoli-5mer-template.kdtidx\
     --complementindex indices/ecoli-5mer-complement.kdtidx ecoli/005/LomanLabz_PC_Ecoli_K12_R7.3_2549_1_ch182_file148_strand.fast5 \
     ...  > template-complement-005.txt

real    1m37.843s
user    1m37.424s
sys 0m0.756s

Template+Complement Alignements
Read           Difference  BWA      KDTree   zscore
ch34_file53    228         1128120  1128348  11.235000
ch464_file15   2031        2195379  2193348  14.782000
ch182_file148  2441        2767555  2769996  17.120000
ch461_file9    3105        2195243  2198348  10.378000
ch401_file98   5019        2328329  2333348  7.516000
ch277_file143  5819        3079177  3084996  17.164000
ch498_file171  6130        3793866  3799996  11.916000
ch222_file28   6765        2803231  2809996  11.152000
ch80_file64    10649       4369347  4379996  7.469000
ch395_file89   10828       2069168  2079996  9.517000
{% endhighlight %}
The 6mer data gives similar results. 

We see a couple of things here:

* Adding the complement strand does almost nothing for the accuracy,
but requires substantially more memory and compute time, as multiple
indicies must be loaded and used up, and all candidate complement
indicies must be compared against each other.  Because of this and
the typically higher skip/stay rates for complement strands, we will use the template
strand only for the rest of this post;
* Since we are simply assigning starting bins at this point, 
any assignments within the bin size are equally accurate; here, all
of the reads were correctly assigned to the starting bin.
* The zscore here is a very crude measure of how much the assignment
stands out over the background (but not necessarily how it compares
to other candidate mappings); some of these very simple pseudo-mappings 
are relatively securely identified, and others less so.  The sum-of-scores for
the reads with the best and worst zscore results are plotted below; no prize for
guessing which is which:

<table>
<tr>
<td> <img src="/assets/kdtreemapping/ch182_file148_simple.png" alt="ch182_file148" style="width: 400px;"/> </td>
<td> <img src="/assets/kdtreemapping/ch401_file98_simple.png" alt="ch182_file148" style="width: 400px;"/> </td>
</tr>
</table>


Because many levels are clustered near ~60-70pA, many dmers are quite
close to each other, and choosing simply the closest $$d$$-point is unlikely
to give a robust result.  Examining all possible matches in the spatial
index within some given radius reduces the noise somewhat, at a modest
increase in compute time:

{% highlight bash %}
$ ./index-and-map.sh noclosest templateonly

5mer data: 

Mapping reads: starting with ecoli/005/LomanLabz_PC_Ecoli_K12_R7.3_2549_1_ch182_file148_strand.fast5
time python ./mapread.py --plot save --plotdir plots    --maxdist 3  --templateindex indices/ecoli-5mer-template.kdtidx \
     ecoli/005/LomanLabz_PC_Ecoli_K12_R7.3_2549_1_ch182_file148_strand.fast5 ...  > template-only-005.txt

real    1m32.260s
user    1m30.804s
sys 0m0.948s
Template Only Alignments
Read           Difference  BWA      KDTree   zscore
ch34_file53    228         1128120  1128348  17.545000
ch461_file9    1895        2195243  2193348  11.832000
ch464_file15   2031        2195379  2193348  19.064000
ch401_file98   5019        2328329  2333348  11.723000
ch498_file171  6130        3793866  3799996  15.583000
ch222_file28   6765        2803231  2809996  15.847000
ch182_file148  7441        2767555  2774996  20.122000
ch80_file64    10649       4369347  4379996  12.202000
ch277_file143  10819       3079177  3089996  18.217000
ch395_file89   10828       2069168  2079996  12.179000
{% endhighlight %}

Note the increase in zscores; the same two reads are plotted:

<table>
<tr>
<td> <img src="/assets/kdtreemapping/ch182_file148_noclosest.png" alt="ch182_file148" style="width: 400px;"/> </td>
<td> <img src="/assets/kdtreemapping/ch401_file98_noclosest.png" alt="ch182_file148" style="width: 400px;"/> </td>
</tr>
</table>

With a few other tweaks - keeping track of the top 10 candidates,
and for each re-testing by recalibrating given the inferred mapping
and rescoring - we can get almost 99% accuracy on the 5mer data - the 6mer
data is more problematic because of higher skip/stop rates.

![kd-tree approximate mapping vs BWA MEM mapping positions](/assets/kdtreemapping/dotplot.png)

Of course, while 95-99% (Q13-Q20) mapping accuracy on _E. coli_ is
a cute outcome from such a simple approach, it isn&#8217;t nearly
enough; with $$d=7$$ and $$k=6$$, wer&#8217;e working with seeds of
size 12, which would typically be unique in the _E. coli_ reference,
but certainly wouldn&#8217;t be in the human genome, or for metageomic
applications.

### EM Rescaling

One limiting factor is the approximate nature of the rescaling that is
being performed at the beginning; for reads that have been basecalled,
the inferred shift values above can be off by several picoamps from the
Metrichor values, which clearly causes both false negatives and false
positives in the index lookup.  This can be addressed by doing a more
careful rescaling step, using EM iterations:

* For the E-step, provisionally assign probabilities of read events
correspondinng to model levels, based on the gaussian distributions 
described in the model and a simple transition matrix between events;
* For the M-step, perform a weighted least squares regression to
re-scale the read levels.

This gives answers that are quite good when compared with Metrichor,
at the cost of substantially more computational effort (much greater
than the spatial index lookup!), particularly for long reads and
larger \(k\) where the number of model levels is larger:

{% highlight bash %}
$ ./index-and-map.sh noclosest templateonly rescale

5mer data: 

Mapping reads: starting with ecoli/005/LomanLabz_PC_Ecoli_K12_R7.3_2549_1_ch182_file148_strand.fast5
time python ./mapread.py --plot save --plotdir plots  --rescale  --maxdist 3  --templateindex indices/ecoli-5mer-template.kdtidx \
     ecoli/005/LomanLabz_PC_Ecoli_K12_R7.3_2549_1_ch182_file148_strand.fast5 ...  > template-only-005.txt

real    5m30.113s
user    5m35.216s
sys 0m23.236s
Template Only Alignments
Read           Difference  BWA      KDTree   zscore
ch34_file53    228         1128120  1128348  18.596000
ch461_file9    1895        2195243  2193348  12.868000
ch464_file15   2031        2195379  2193348  20.071000
ch182_file148  2441        2767555  2769996  22.090000
ch401_file98   5019        2328329  2333348  15.125000
ch395_file89   5828        2069168  2074996  14.260000
ch498_file171  6130        3793866  3799996  20.984000
ch222_file28   6765        2803231  2809996  19.572000
ch277_file143  10819       3079177  3089996  22.064000
ch80_file64    15649       4369347  4384996  17.210000
{% endhighlight %}

Note again the increase in zscores; replotting gives:

<table>
<tr>
<td> <img src="/assets/kdtreemapping/ch182_file148_rescale.png" alt="ch182_file148" style="width: 400px;"/> </td>
<td> <img src="/assets/kdtreemapping/ch401_file98_rescale.png" alt="ch182_file148" style="width: 400px;"/> </td>
</tr>
</table>

### Extending the seeds

So far we have in no way taken into account any of the locality
information in the spatial index lookup results; that is, that a long
series of hits close together, in the same order on the read and on the
reference, is much stronger evidence for a good mapping than a haphazard
series of hits in random order.  

Keeping with the do-the-simplest-thing approach that has worked so far,
we can try to extend these "seed" matches by attempting to stitch
them together into longer seeds; here we build 15-mers out of sets of
4 neighbouring 12-mers, allowing one skip or stay somewhere within them,
using as a score for the result the minimum of the constitutent scores,
and dropping all hits that cannot be so extended:

{% highlight bash %}
$ ./index-and-map.sh noclosest templateonly rescale extend

5mer data: 

Mapping reads: starting with ecoli/005/LomanLabz_PC_Ecoli_K12_R7.3_2549_1_ch182_file148_strand.fast5
time python ./mapread.py --plot save --plotdir plots  --rescale --extend --maxdist 3  --templateindex indices/ecoli-5mer-template.kdtidx \
     ecoli/005/LomanLabz_PC_Ecoli_K12_R7.3_2549_1_ch182_file148_strand.fast5 ...  > template-only-005.txt

real    5m44.699s
user    5m48.304s
sys 0m24.000s
Template Only Alignments
Read           Difference  BWA      KDTree   zscore
ch401_file98   19          2328329  2328348  25.848000
ch34_file53    228         1128120  1128348  28.754000
ch498_file171  1130        3793866  3794996  28.390000
ch464_file15   2031        2195379  2193348  28.489000
ch461_file9    3105        2195243  2198348  29.191000
ch222_file28   6765        2803231  2809996  28.578000
ch182_file148  7441        2767555  2774996  29.820000
ch277_file143  10819       3079177  3089996  29.665000
ch395_file89   10828       2069168  2079996  24.867000
ch80_file64    15649       4369347  4384996  29.631000
{% endhighlight %}

<table>
<tr>
<td> <img src="/assets/kdtreemapping/ch182_file148_extend.png" alt="ch182_file148" style="width: 400px;"/> </td>
<td> <img src="/assets/kdtreemapping/ch401_file98_extend.png" alt="ch182_file148" style="width: 400px;"/> </td>
</tr>
</table>

Now that we seem to have much stronger and more specific results,
we can investigate letting go of binning, and instead string these
extended seeds into longest possible matches.  Continuing to extend
dmer-by-dmer is too challenging due to missing points (due to
mis-calibration, too large noise in the dmer falling out of the
maxdist window in the kdtree, or several skip/stays in a row), so
following the ideas of graphmap[^1] and minimap[^5], we trace
collinear seeds through the set of available seeds; rather than
just taking longest increasing sequences in inferred start positions,
however, we make a graph of seeds with edges connecting  seeds that
are monotonically increasing both in read location and reference
location with 'small enough' jumps, and extract longest paths through
this graph.  The cost of this is smaller than the rescaling of the read.

Running this, the zscores now somewhat change meaning; only the extracted
paths are scored, meaning that the scores are only amongst plausible
mappings, not over all bins across the reference.  Similarly, the differences
in mapping locations are somewhat more meaningful - rather than being compared
to bin centres, they are actually the differences between mapping locations.

We get:



A proper implementation of these ideas would strip out the multiple,
copies of large, reference-sized data structures that are being used
and avoid the extensive python list manipulations that are used in the
mapping.  We will examine a C++ implementation of this basic approach
in the new year, which will also have a a rather more sophisticated approach
to assessing likelihoods.

---

### References

[^1]: [Fast and sensitive mapping of error-prone nanopore sequencing reads with GraphMap](http://biorxiv.org/content/early/2015/06/10/020719) (2015) by Sovic, Sikic, Wilm, _et al._

[^2]: [Near-optimal RNA-Seq quantification](http://arxiv.org/abs/1505.02710) (2015) by Bray, Pimentel, Melsted, and Pachter

[^3]: [Salmon: Accurate, Versatile and Ultrafast Quantification from RNA-seq Data using Lightweight-Alignment](http://biorxiv.org/content/early/2015/06/27/021592) (2015) by Patro, Duggal, and Kingsford

[^4]: [RapMap](https://github.com/COMBINE-lab/RapMap), COMBINE lab

[^5]: [Minimap and miniasm: fast mapping and denovo assembly for noisy long sequences](http://arxiv.org/abs/1512.01801), Heng Li
