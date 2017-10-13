# PyParanoid

**Ryan Melnyk**  
**[Haney Lab - University of British Columbia](https://haneylab.msl.ubc.ca/)**  
**v1.0 - release ???**

***ryan.melnyk@msl.ubc.ca***  
***schmelnyk@gmail.com***

![PyParanoid pipeline](src/pipeline.png "PyParanoid pipeline")

PyParanoid is a pipeline for rapid identification of homologous gene families in a set of genomes - a central task of any comparative genomics analysis. The "gold standard" for identifying homologs is to use reciprocal best hits (RBHs) which depends on performing a all-vs-all sequence comparison, usually using BLAST, to determine homology.  However, these methods are computationally expensive, requiring **O(n<sup>2</sup>)** resources to identify RBHs. This is problematic, as the modern deluge of sequencing data means that comparative genomics analyses could be performed on datasets of thousands of strains.

To circumvent this obstacle, I developed a two-step machine learning-inspired pipeline which develops gene models for the pangenome of a training dataset and then propagates those models to additional strains (the "test dataset").  The first step identifies homologs using conventional **O(n<sup>2</sup>)** RBH-based methods but relies on the recent sequence alignment program [DIAMOND](http://github.com/bbuchfink/diamond) to speed up this process by an order of magnitude over BLAST-based methods.  The second step uses the gene models generated in the first step to propagate annotations to additional strains using **O(n)** resources. I named this pipeline PyParanoid as it is written in Python and relies on the pairwise homology algorithm [InParanoid](http://inparanoid.sbc.su.se/cgi-bin/faq.cgi). Using this pipeline, it is possible to generate homology-based annotations for thousands of bacterial and archaeal genomes in hours on a local machine.

I have also included some scripts that facilitate upstream genome database management as well as downstream comparative genomics workflows using the PyParanoid database, such as scripts for synteny analysis and homology-based visualizations of genomes.

## Installation

PyParanoid is primarily written in python.  To manage python installations and environments, I highly recommend using [Miniconda](https://conda.io/miniconda.html).

PyParanoid also depends on a modified version of the InParanoid script ([Sonnhammer et al., 2015](http://inparanoid.sbc.su.se/cgi-bin/faq.cgi)) and thus also requires perl to be installed.

#### Python modules using Miniconda :snake::snake::snake:
```
conda install biopython
conda install pandas
conda install seaborn
```

#### Using Homebrew :beer::beers::beers::beer:

OSX users can install these dependencies easily using [Homebrew](https://brew.sh/) and running the following commands.

```
brew tap homebrew/science
brew install diamond
brew install hmmer
brew install mcl
brew install cd-hit
brew install muscle
```

#### Manual installation

If Homebrew doesn't work or you aren't in OSX, you will have to install the dependencies manually. Please ensure that all executables are located in a folder accessible in your $PATH.

###### DIAMOND
See http://github.com/bbuchfink/diamond for details.  Linux users can download an executable [here](https://github.com/bbuchfink/diamond/releases).

###### HMMER
http://hmmer.org/download.html - Download the newest version for your operating system.  PyParanoid should work with HMMER2 and HMMER3.

###### mcl
mcl can be found [here](https://www.micans.org/mcl/index.html?sec_software).

###### cd-hit
CD-hit can be found [here](http://weizhongli-lab.org/cd-hit/)

###### muscle
muscle can be found [here](http://www.drive5.com/muscle/)

## Running PyParanoid


#### Input

At minimum, you can run PyParanoid using just a folder of FASTA-formatted amino acid files.  These should be named with a ```.faa``` file extension and stored in a folder named ```pep```.  The "genomedb" argument for ```PyParanoid.py``` and all other scripts should point to the relative location of the folder containing ```pep```.

Within the ```pep``` folder, all ```.faa``` files should have different prefixes reflecting the strain names. Underscores, letters and numbers are permitted.  Additionally, within each strain's .faa file sequence IDs should be unique.

Alternatively, you can use ```DownloadEnsemblGenomes.py``` to download genomic data for bacteria and archaea from Ensembl into a single local database. You can download specific genera or species using the ```--names``` argument or specific taxonomy IDs using the ```--taxids``` option.  Check out [Ensembl Bacteria](http://bacteria.ensembl.org/species.html) for more info on what is available.

Once this local genomic database is built, you can also add local, [Prokka](https://github.com/tseemann/prokka)-annotated genomes using ```AddLocalGenome.py```.  Using these options will enable your genomic input data to be compatible with downstream scripts that rely on accessing Genbank data for analysis or visualization.

PyParanoid also requires a ```strainlist.txt``` which should be a text file containing the names of the strains to include in the analysis, one per line.  These should match up with the prefixes of the amino-acid FASTA files in the ```pep``` folder.

#### Input vs output

Generally the way I use PyParanoid is to build a pangenome using only high-quality reference genomes.  This is primarily because draft genomes in many contigs frequently have split, truncated, or missing genes due to assembly breakpoints.  Once the pangenome is built, these draft genomes can be added to the database.  Importantly, they can be added month-by-month as new sequences become available.

As an example, I built gene models for ~200 Pseudomonas strains in late 2016 and then annotated ~900 more genomes using those models.  To add additional genomes now, I simply propagate the gene families to the new genomes rather than redoing the entire workflow.

Of course, this is a tradeoff between computational time and detecting novel gene families.  Obviously, it depends on the phylogenetic distribution or the training dataset vs. the test dataset.  If you used a training set of hundreds of genomes that are broadly distributed across a genus or family, you will have better luck propagating those gene families than if you use only a few dozen strains from a single species as the training dataset and then use those gene families on a test dataset containg genomes from a different genus or family. A good example of how to be confident in the phylogenetic scope of the training and test datasets can be found in the PyParanoid manuscript.

 I am planning to address this issue in future development by adding an option to add new gene families as new strains get added to an existing database.

## Example applications

#### Build a Pseudomonas fluorescens database :seedling::ear_of_rice::herb:

Here is an example to carry out a basic analysis using our favorite plant-associated bacterium *Pseudomonas fluorescens*.  This is a full walkthrough of PyParanoid's capabilities on a modest-sized dataset - it should take roughly 1-2 hrs on a 4-8 core system.

Note that in this example, only genomes named *Pseudomonas fluorescens* will be downloaded.  Due to the messiness inherent in bacterial species nomenclature (especially *Pseudomonas spp.*), this will likely be a polyphyletic group.

```bash
python DownloadEnsemblGenomes.py --complete --names fluorescens pfl_genomeDB

## Generate strainlist.txt for the complete strains using your favorite method
## i.e. this command pulls out all strain names just downloaded
cut -f 3 pfl_genomeDB/genome_metadata.txt | grep -v -x "species" > strainlist.txt

## Run PyParanoid to generate homology calls and models
python PyParanoid.py --clean --verbose pfl_genomeDB strainlist.txt pfl_pyp

## Download more Pfl genomes to genomeDB, this time including draft genomes
python DownloadEnsemblGenomes.py --names fluorescens pfl_genomeDB

## Go all out and download every Pfl sequence from RefSeq
python ../PyParanoid/DownloadNcbiGenomes.py --name "Pseudomonas fluorescens" pfl_genomeDB

## generate prop_strainlist.txt for the new files just downloaded

## Propagate groups to new draft genomes
python PropagateGroups.py pfl_genomeDB prop_strainlist.txt pfl_pyp
```

## Build a species tree

###### Pull out orthologs

This command pulls out orthologs from the "pfl_pyp" database generated in the previous section. As this dataset includes many draft genomes with missing or fragmented genes, specifying a threshold is a good idea. ```--threshold 0.9``` will find genes present as a single copy in over 90% of all strains. If no threshold is specified, orthologs will be "strict" (i.e. present as a single copy in every strain).

```bash
python IdentifyOrthologs.py --threshold 0.9 pfl_pyp pfl_ortho
```

FINISH THIS SECTION

## Analysis examples

I've included a few iPython notebooks detailing some relatively simple analyses you can do using only three ingredients: the presence/absence matrices generated by PyParanoid, a species tree, and genbank files for specific strains of interest.

Check them out in the ```analysis_ipython``` folder.  They all depend on my local Pseudomonas PyParanoid database which is too massive to share easily. Just adapt the path arguments to the relevant folders for your PyParanoid project.


ADD PICS AND MORE NOTEBOOKS

## Citing PyParanoid

if you're reading this PyParanoid is still in development so just cite this github page.
