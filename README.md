assemblage [əˈsɛmblɪdʒ]  
*noun*

1. a number of things or persons assembled together; collection; assembly
2. (Cookery) a list of dishes served at a meal or the dishes themselves
3. the act or process of assembling or the state of being assembled
4. (Fine Arts & Visual Arts / Art Terms) a three-dimensional work of art that combines various objects into an integrated whole

From Collins English Dictionary – Complete and Unabridged © HarperCollins Publishers 1991, 1994, 1998, 2000, 2003

About
=====

This is a set of scripts for working with genome assemblies, making taxon-annotated GC-coverage plots (a.k.a blob plots), extracting reads belonging to the blobs, etc

Example Data Sets
=================

As an example, we can use the following study from the Short Read Archive:

* ERP001495 - De novo whole-genome sequence of the free-living nematode Caenorhabditis sp. 5 strain JU800 DRD-2008

This study has one sample:

* ERS147916 - Caenorhabditis sp. 5 JU800 DRD-2008  

Two libraries (or "Experiments" in SRA terminology) were run for this sample:

* ERX114449 - 300 bp library with Illumina HiSeq2000 101 bp PE sequencing
* ERX114450 - 600 bp library with Illumina HiSeq2000 101 bp PE sequencing

And these are the four files

* `g_ju800_110714HiSeq300_1.txt.gz` - 300 bp library forward read
* `g_ju800_110714HiSeq300_2.txt.gz` - 300 bp library reverse read
* `g_ju800_110714HiSeq600_1.txt.gz` - 600 bp library forward read
* `g_ju800_110714HiSeq600_1.txt.gz` - 600 bp library reverse read

How-Tos
=======

This list of How-Tos demonstrates how the scripts in this repository can be used in conjunction with other installed software to assemble nematode (and other small metazoan genomes).

How to sample sequences at random for test read sets
----------------------------------------------------

The example read files are between 5 and 8 GB each, and processing the complete data set could take a long time. If you just want to run all the How-Tos in this README on smaller files to test how the scripts work, use the `pick_random.pl` script:

    zcat g_ju800_110714HiSeq300_1.txt.gz | pick_random.pl 4 0.1 > g_ju800_110714HiSeq300_1.txt.r0.1

This command will gunzip the gzipped fastq file (`zcat file.gz` is the same as `gunzip -c file.gz`) and pipe the output through `pick_random.pl`. The script takes two arguments - the first tells you how many lines at a time should be treated as a block, while the second tells you what proportion of the file to pick (0.1 or 10% in this case)

However, running this command independently on the forward (`_1`) and reverse (`_2`) read files will result in different reads being picked. To sample both pairs at the same time, first interleave the files, and then pick 8 lines at a time (4 for the forward read, and 4 for the reverse read), and then use `pick_random.pl` to get 10% of the reads:

    shuffleSequences_fastx.pl 4 <(zcat g_ju800_110714HiSeq300_1.txt.gz) <(zcat g_ju800_110714HiSeq300_2.txt.gz`) |
    pick_random.pl 8 0.1 > g_ju800_110714HiSeq300_interleaved.txt.r0.1

Note: The <() syntax is for bash process substitution. Anything inside <(...) will be executed and its output will be treated by the containing command as if it is a file.

`shuffleSequences_fastx.pl` is based on shuffleSequences_fastq that used to ship with Velvet. The difference is that it can be used to shuffle both fasta (set 1st argument to 2) and fastq (set 1st argument to 4) sequences.

How to adapter- and quality-trim Illumina fastq reads using sickle and scythe in one command with no intermediate files
-----------------------------------------------------------------------------------------------------------------------

(Hmm, that's a bit of a long How-To title, almost like article titles in the 1600s, such as this one: [An Observation and Experiment Concerning a Mineral Balsom, Found in a Mine of Italy by Signior Marc-Antonio Castagna; Inserted in the 7th. Giornale Veneto de Letterati of June 22. 1671, and Thence English'd as Follows](http://dx.doi.org/10.1098/rstl.1671.0068))

Requirements:

1. Install scythe for adapter trimming from https://github.com/ucdavis-bioinformatics/scythe 
2. Install sickle for quality trimming from https://github.com/ucdavis-bioinformatics/sickle
3. Install GNU Parallel (optional, but highly recommended!) from http://www.gnu.org/software/parallel/
4. A file with Illumina adapters - you can use `adapters.fa` from this repository

Command:

    sickle pe -t sanger -n -l 50 \
        -f <(scythe -a adapters.fa <(zcat g_ju800_110714HiSeq300_1.txt.gz) -q sanger \
            2> g_ju800_110714HiSeq300_1.scythe.err | perl -plne 's/^$/A/; s/ 1.*/\/1/')  \
        -r <(scythe -a adapters.fa <(zcat g_ju800_110714HiSeq300_2.txt.gz) -q sanger \
            2> g_ju800_110714HiSeq300_2.scythe.err | perl -plne 's/^$/A/; s/ 2.*/\/2/')  \
        -o >(gzip >g_ju800_110714HiSeq300_1.clean.txt.gz) \
        -p >(gzip >g_ju800_110714HiSeq300_2.clean.txt.gz) \
        -s >(gzip >g_ju800_110714HiSeq300_s.clean.txt.gz) \
            &>g_ju800_110714HiSeq300.sickle.err
        
The command above will first run scythe to search for adapters.fa in `g_ju800_110714HiSeq300_1.txt.gz` and `g_ju800_110714HiSeq300_2.txt.gz` simultaneously. The stderr stream of scythe is stored in `g_ju800_110714HiSeq300_1.scythe.err` and the output of scythe is parsed through a perl one liner that replaces blank lines with a single A, and adds "/1" or "/2" to the read header. This perl one liner is needed because scythe screws up the read names, and because the next script sickle can't deal with blank sequence lines where the whole sequence has been adapter-trimmed.

`sickle pe -t sanger -n -l 50` then runs sickle on the files output by the scythe and perl one liner above:

* `pe` means treat corresponding reads from the `-f` and `-r` files as being parts of a pair, so if a read is discarded, its pair is discarded too.
* `-t sanger` tells it that the quality values are in sanger fastq encoding
* `-n` discards reads/pairs with any Ns in them
* `-l 50` discards reads/pairs with fewer than 50 bp

Additional scythe and sickle parameters can be set as needed (minimum number of matches to adapter sequence, trim bases from start of read, etc).

If you do have GNU Parallel installed, you can run multiple libraries at the same time like this:

    parallel "sickle pe -t sanger -n -l 50 \
        -f <(scythe -a adapters.fa <(zcat {}_1.txt.gz) -q sanger \
            2> {}_1.scythe.err | perl -plne 's/^$/A/; s/ 1.*/\/1/')  \
        -r <(scythe -a adapters.fa <(zcat {}_2.txt.gz) -q sanger \
            2> {}_2.scythe.err | perl -plne 's/^$/A/; s/ 2.*/\/2/')  \
        -o >(gzip >{}_1.clean.txt.gz) \
        -p >(gzip >{}_2.clean.txt.gz) \
        -s >(gzip >{}_s.clean.txt.gz) \
            &>{}.sickle.err" ::: g_ju800_110714HiSeq300 g_ju800_110714HiSeq600 

How to run khmer to digitally normalise reads
---------------------------------------------

Requirements:

1. Install screed from https://github.com/ctb/screed
2. Install khmer  from https://github.com/ctb/khmer - this tutorial site might help: http://ged.msu.edu/angus/diginorm-2012/tutorial.html

Typically, you might run *khmer* with a high-pass and a low-pass filter. First remove high coverage reads that don't contribute new k-mers beyond coverage 20, and then remove low coverage reads that have infrequent k-mers (e.g, if coverage <5 is likely to be sequencing errors or non-abundant contaminants). In some cases, only the high-pass filter is used.

khmer can be run on _1 forward and _2 reverse files separately, but we recommend that the files are interleaved before running, otherwise you might end up with almost all _2 reads discarded if all k-mers are found in the _1 file.

    shuffleSequences_fastx.pl 4 <(zcat g_ju800_110714HiSeq300_1.clean.txt.gz) <(zcat g_ju800_110714HiSeq300_2.clean.txt.gz`) | gzip >g_ju800_110714HiSeq300.clean.txt.gz

In this case, I am using the scythed and sickled cleaned read fastq files generated above.

Now, run the high-pass filter first:

    normalize-by-median.py g_ju800_110714HiSeq300.clean.txt.gz -C 20 -k 25 -N 4 -x 2e8 -s interleaved.C20.kh -R interleaved.C20.report
    
The command above will take the gzipped cleaned interleaved fastq file specified and normalize-by-median all reads to a max-coverage of `-C 20`.  
`-k 25` specifies that k-mers of size 25 are used for normalization (this is a reasonable default).  
`-N 4 -x 2e8` specifies how many blocks are used and how big each is (this settig will take up 4 x 2e8 = 3.2 GB of memory).  
`-s` specifies the name of the file where the k-mer counts are stored and `-R` specifies a report file.

The output of this command will be a .keep file: `g_ju800_110714HiSeq300.clean.txt.gz.keep` which is a fasta file (khmer doesn't output fastq quality files).

To run the low-pass filter (in this case, remove reads with coverage less than `-C` 5X):

    filter-abund.py -C 5 interleaved.C20.kh g_ju800_110714HiSeq300.clean.txt.gz.keep

The command above takes the k-mer count file `interleaved.C20.kh` and whenever it sees reads in `g_ju800_110714HiSeq300.clean.txt.gz.keep` that have very infrequent k-mer coverage, it discards them. The final output file of this step is named `g_ju800_110714HiSeq300.clean.txt.gz.keep.abundfilt`.

One important thing to remember is that khmer works on individual reads and not on pairs. As a result, the pairing information of a read is lost if its pair is discarded. To get around this problem, we use the script `khmer_re_pair.pl` which pulls in the paired reads:

    khmer_re_pair.pl -o g_ju800_110714HiSeq300.clean.txt.gz -k g_ju800_110714HiSeq300.clean.txt.gz.keep.abundfilt > g_ju800_110714HiSeq300.clean.txt.gz.keep.abundfilt.repaired

Currently, `khmer_re_pair.pl` needs the complete file to be an interleaved fasta file (can be gzipped). For each read in the khmer'ed fasta file `g_ju800_110714HiSeq300.clean.txt.gz.keep.abundfilt`, it looks up the original interleaved fasta file and outputs both reads in the pair.

How to create a preliminary assembly using ABySS
------------------------------------------------

Requirements:

1. Install ABySS from http://www.bcgsc.ca/platform/bioinfo/software/abyss . This README was tested with versions 1.3.3 and 1.3.4. Versions before 1.3.3 did not work as well. It is possible that newer versions will have different options and work even better.

To get a preliminary assembly (PASS) with no pairing information (i.e. treating all reads as single-end),  

How to create a preliminary assembly using CLC
----------------------------------------------

How to map reads to an assembly to get coverage information for each contig/scaffold
------------------------------------------------------------------------------------

How to make a plot of insert sizes for each library
---------------------------------------------------


How to search for the best blast hit for a contig
-------------------------------------------------

How to make a taxon-annotated GC cov "blob" plot
------------------------------------------------

How to select preliminary assembly (PASS) contigs with given GC or coverage
---------------------------------------------------------------------------

How to make taxon-specific blast databases
------------------------------------------

How to separate contigs based on which taxon-specific blast database they hit better
------------------------------------------------------------------------------------

