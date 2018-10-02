### stacks_full_fasta_out  

A bit of code to recreate the "--phylip_var_all" style output from stacks v1 with stacks v2 output. 
This output includes SNPs of interest as well as flanking sequence of each catalog locus around the SNPs,
and as of 20180928 this option hasn't been implemented in Stacks v2.  

#### Requires:  

* `populations.samples.fa`: standard fasta output from populations  
* `../all_mach_20badsRmvd`: a popmap-style list of all individuals (can create from popmap with `awk '{print $1}'`)  
* `Consensus.pl`: I use this to create strict IUPAC consensus sequences from all alleles per individual. Can use other utility, but this is what I had sitting around (source: [Joseph Hughes](https://github.com/josephhughes/Sequence-manipulation/blob/master/Consensus.pl)).  
(Note, that many IUPAC consensus utilities, e.g. [Bio.motifs](http://biopython.org/DIST/docs/tutorial/Tutorial.html) in BioPython, do not create strict consensus sequences, but often use the rules to determine when IUPAC codes are introduced (e.g. [Cavener 1987](https://academic.oup.com/nar/article-lookup/doi/10.1093/nar/15.4.1353)).  
* [parallel](https://www.gnu.org/software/bash/manual/html_node/GNU-Parallel.html)

To be run on moana cluster, using 32 slots (`parallel -j`).

```#!/bin/sh
#$-N fastaFAST
#$-S /bin/sh
#$-o ./fastaFAST_$JOB_ID.out
#$-e ./fastaFAST_$JOB_ID.err
#$-q all.q
#$-pe smp 32
#$-cwd
#$-V

module load parallel

# make list of all "CLocus" IDs in stacks --fasta_samples output
grep "CLocus" populations.samples.fa | awk 'BEGIN {FS="_"; OFS="_"} ; {print $1,$2}' | sed 's/^.//g; s/$/_/g' | sort -u > CLocus_list

# creates fasta files for every possible CLocus x individual combination (creates empty files for non-existant combos)
parallel -j 32 "grep {1} -A1 populations.samples.fa | grep {2} -A1 > {1}.{2}.fastaTEMP" :::: CLocus_list ../all_mach_20badsRmvd

# removes empty fastaTEMP files
ls ./*.fastaTEMP | parallel -j 32 "[ -s {} ] || rm {}" 

# creates consensus sequences for every CLocus x individual combination
ls ./*.fastaTEMP | parallel -j 32 "perl /home/jrdupuis/Random_scripts/Consensus.pl -in {} -out {}.consensus -iupac && sed -i 's/\.\///g' {}.consensus"

# creates a combined output fasta for every CLocus, removes some funny naming business in the fasta headers (generated by Consensus.pl, which uses the file name as a fasta header), and removes the TGCAG from the beginning of every read
cat CLocus_list | parallel -j 32 "cat ./{}*.fastaTEMP.consensus > {}.combined.fasta && sed -i 's/{}\.//g; s/_consensus//g; s/^TGCAG//g' {}.combined.fasta" 

# clean up
rm ./*.fastaTEMP
rm ./*.fastaTEMP.consensus
  ### could also use parallel to do these rm's from the CLocus_list file? Not hitting any arg list errors though ( -bash: /usr/bin/ls: Argument list too long)```
