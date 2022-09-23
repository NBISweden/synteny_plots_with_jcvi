# How to create synteny plots using JCVI / MCscan (Python version)

This is a step-by-step tutorial on how to create synteny plots between two or multiple species using [MCScan](https://github.com/tanghaibao/jcvi/wiki/MCscan-(Python-version)). I'm using as examples the Lepidium project. Many of the procedures can be wrapped into simpler scripts, but this is a detailed explanation on how to run and what to expect, and things that will break.

Most steps run really fast, and I have ran it on my local machine with success. At the Known Issues at the bottom of the page there's a small list of things that appear broken (all asthetics parameters for legend texts, nothing that will change the analyses results).

All figures and examples right now come from a single project, so this is for internal use only. I will eventually create a mock project to use non-user owned figures so we can make this public.

## Requirements

**If you're using [NAC](https://github.com/NBISweden/annotation-cluster/wiki/NAC-general), you don't need to re-create the environment described below, it is already there. Just `conda activate jcvi`.**

- Install texlive and other packages. Currently conda installation is not working properly, so system wide installation:

`sudo apt-get install -y texlive texlive-latex-extra texlive-latex-recommended dvipng cm-super msttcorefonts`

- Create environment, install required packages, then install JCVI using pip. Prefer pip because the conda version struggles to see and use texlive, it's a known bug to the developer. Also installing everything one step at a time might look weird and a waste of time, but it will help with conflicts of versions of required packaged for each tool. Doing it in this order works at [NAC](https://github.com/NBISweden/annotation-cluster/wiki/NAC-general). 

```
conda create -n jcvi python=3.9
conda activate jcvi
conda install pip
conda install -c conda-forge biopython
conda install -c conda-forge matplotlib
conda install -c anaconda scipy
conda install -c bioconda last
pip install jcvi
```

## Preparing the data for usage by JCVI

### Getting the CDS from the genomes using AGAT
If you use the [JCVI](https://github.com/tanghaibao/jcvi/) native tool it will break downstream, but using [AGAT](https://github.com/NBISweden/AGAT) for the file conversions works perfectly fine.

`conda activate agat`
```
agat_sp_extract_sequences.pl -g L_campestre_rc3_functional.gff -f campestre_genome.fa -t cds -o campestre_genome.cds.fa
agat_sp_extract_sequences.pl -g L_heterophyllum_rc3_functional.gff -f heterophyllum_genome.fa -t cds -o heterophyllum_genome.cds.fa
```
### Getting BED files using AGAT
If you use the [JCVI](https://github.com/tanghaibao/jcvi/) native tool it will break downstream, but using AGAT for the file conversions works perfectly fine.
```
agat_convert_sp_gff2bed.pl --gff /projects/sandbox/andre/Lepidium-synteny/data/L_campestre_rc3_functional.gff -o campestre.bed
agat_convert_sp_gff2bed.pl --gff /projects/sandbox/andre/Lepidium-synteny/data/L_heterophyllum_rc3_functional.gff -o heterophyllum.bed
agat_convert_sp_gff2bed.pl --gff /projects/sandbox/andre/Lepidium-synteny/data/hybrid_LcxL_rc3_functional.gff -o hybrid.bed
```
`conda deactivate`


### You need to reformat the FASTA files too
Now using [JCVI](https://github.com/tanghaibao/jcvi/) so the cds file is correctly loaded later on.

`conda activate jcvi`
```
python -m jcvi.formats.fasta format /projects/sandbox/andre/Lepidium-synteny/data/campestre_genome.cds.fa campestre.cds
python -m jcvi.formats.fasta format /projects/sandbox/andre/Lepidium-synteny/data/heterophyllum_genome.cds.fa heterophyllum.cds
python -m jcvi.formats.fasta format /projects/sandbox/andre/Lepidium-synteny/data/hybrid_genome.cds.fa hybrid.cds
```

## Pairwise synteny search
This will generate the input files for the synteny maps for pairs of genomes. It will use the basename of the cds and bed files. It will assume both are identical.
```
python -m jcvi.compara.catalog ortholog campestre heterophyllum --no_strip_names
python -m jcvi.compara.catalog ortholog campestre hybrid --no_strip_names
python -m jcvi.compara.catalog ortholog heterophyllum hybrid --no_strip_names
```

## Pairwise synteny visualization
This will create the synteny maps for pairs of genomes.
```
python -m jcvi.graphics.dotplot campestre.heterophyllum.anchors
python -m jcvi.graphics.dotplot campestre.hybrid.anchors
python -m jcvi.graphics.dotplot heterophyllum.hybrid.anchors
```

![synteny map example](https://github.com/NBISweden/synteny_plots_with_jcvi/blob/main/figures/campestre.heterophyllum.syntenymap.png)

## Plotting the histogram
Histogram of synteny patterns (# of blocks per gene).
```
python -m jcvi.compara.synteny depth --histogram campestre.heterophyllum.anchors
python -m jcvi.compara.synteny depth --histogram campestre.hybrid.anchors
python -m jcvi.compara.synteny depth --histogram heterophyllum.hybrid.anchors
```
![histogram example](https://github.com/NBISweden/synteny_plots_with_jcvi/blob/main/figures/campestre.heterophyllum.depth.png)

## Macro synteny visualization
This will plot figures with "karyotypes" for pairs of species

### First create seqids file
Each line is the name of the chrs for each species. DO NOT HAVE EMPTY LINES AT THE END or everything will break.
```
LG1,LG2,LG3,LG4,LG5,LG6,LG7,LG8
LG1,LG2,LG3,LG4,LG5,LG6,LG7,LG8
```

### Layout file
The parameters for the plot. x-y position on the file, position of labels, etc. DO NOT HAVE EMPTY LINES AT THE END or everything will break.
```
# y, xstart, xend, rotation, color, label, va,  bed
 .6,     .1,    .8,       0,      , L. ampestre, top, campestre.bed
 .4,     .1,    .8,       0,      , L. eterophyllum, bottom, heterophyllum.bed
# edges
e, 0, 1, campestre.heterophyllum.anchors.simple
```

### Create the "simple" file referred to in the *layout* file
It's a simpler version of the anchors file, necessary for plotting
```
python -m jcvi.compara.synteny screen --minspan=30 --simple campestre.heterophyllum.anchors campestre.heterophyllum.anchors.new 
python -m jcvi.compara.synteny screen --minspan=30 --simple campestre.hybrid.anchors campestre.hybrid.anchors.new 
python -m jcvi.compara.synteny screen --minspan=30 --simple hybrid.heterophyllum.anchors hybrid.heterophyllum.anchors.new 
```
### Plot using the *layout* file
Change according to the seqids and layoyt files you're using.
```
python -m jcvi.graphics.karyotype seqids layout
python -m jcvi.graphics.karyotype seqids campestre.hybrid.layout
```

![karyotype style plot example](https://github.com/NBISweden/synteny_plots_with_jcvi/blob/main/figures/campestre.heterophyllum.karyotype.png)


## Macro synteny visualization of three species
This will plot figures with the "karyotypes" style of graph for 3 species at a time

### SEQIDS file example
Just like before, you'll need a seqids file with the chrs you want to plot for the 3 species. Each line for one species, following the same order of the plot. Do not have any empty line in the file.
```
LG1,LG2,LG3,LG4,LG5,LG6,LG7,LG8
LG1,LG2,LG3,LG4,LG5,LG6,LG7,LG8
LG1,LG2,LG3,LG4,LG5,LG6,LG7,LG8
```

### *Layout* file example
Prepare a separate layout file for each trio. The way it works is that the species that will be in the middle must be on the second line. Do not have any empty line in the file.
```
# y, xstart, xend, rotation, color, label,      va,     bed
 .7,     .2,    .95,       0,      , L. campestre,         top,    campestre.bed
 .5,     .2,    .95,       0,      , Hybrid,     bottom,     hybrid.bed
 .3,     .2,    .95,       0,      , L. heterophyllum,     bottom,    heterophyllum.bed
# edges
e, 0, 1, hirtum_atlanticum.heterophyllum.anchors.simple
e, 1, 2, heterophyllum.hirtum_nebrodense.anchors.simple
```

### Plotting with three species now
First make sure you have the pairwise synteny search for the two pairs that will form the three-species plot. Remember that it must be: top species with middle species. Then middle species and bottom species, in this strict order. You might now have these from before in the exact order.
```
python -m jcvi.compara.catalog ortholog campestre hybrid --no_strip_names
python -m jcvi.compara.catalog ortholog hybrid heterophyllum --no_strip_names
```
#### Now you have to create the simplified anchor files, just like before, for the two pairs.
```
python -m jcvi.compara.synteny screen --minspan=10 --minsize=1 --intrabound=500 --simple campestre.hybrid.anchors campestre.hybrid.anchors.new
python -m jcvi.compara.synteny screen --minspan=10 --minsize=1 --intrabound=500 --simple hybrid.heterophyllum.anchors hybrid.heterophyllum.anchors.new
```
#### You can then plot the figure using

`python -m jcvi.graphics.karyotype three.seqids campestre.hybrid.heterophyllum.three.layout`

![three species karyotype style figure](https://github.com/NBISweden/synteny_plots_with_jcvi/blob/main/figures/campestre-hybrid-heterophyllum-version-2-newkaryotype.png)


## Microsynteny visualization
This is used when you want to have plots to highlight the position of a gene (or region, or anything) between two or three species, or just want to plot a limited region of a chromosome, showing off the different genomic features.

### Create the *block.layout* file
Now it has "block" as part of the file name, and it's like before. Do not forget to remove empty lines.

```
# x,   y, rotation,   ha,     va,   color, ratio,            label
0.5, 0.6,        0, left, center,       m,     1,       Campestre LG1
0.5, 0.4,        0, left, center, #fc8d62,     1, Heterophyllum LG1
# edges
e, 0, 1
```

### Create the unified BED file
For the two species you'll be analyzing:
`cat campestre.bed heterophyllum.bed > campestre_heterophyllum.bed`

### Create the *block* file
You want to plot only a region, so you'll need a blocks file with the region of interest. Let's say you want the region around a certain gene:

`grep "LCAMPM00000027240" campestre2.blocks -A 30 -B 30 > gtr1.block`

Now I chose 30 blocks before and after the area I'm interested. You have to play with this to get as long or short region you're interested in.

### Plotting microsynteny figure
`python -m jcvi.graphics.synteny gtr1.block campestre_heterophyllum.bed blocks.layout`

![single gene microsynteny two species](https://github.com/NBISweden/synteny_plots_with_jcvi/blob/main/figures/lg1.micro.two.species.png)

#### Add-ons to plotting
If you want, you can add more flourishes to the plot. You can use "--glyphstyle=arrow" to add arrows for genes/features, also "--genelabels=LCAMPM00000027240" to highlight a particular feature of interest, and control the size of the font with "--genelabelsize=10".

`python -m jcvi.graphics.synteny gtr1block campestre_heterophyllum.bed gtr1.blocks.layout --glyphstyle=arrow --genelabels=LCAMPM00000027240,LHETEM00000033639 --genelabelsize=10`


## Microsynteny for a single region/gene between three species

It follows the same steps as before. Create the block.layout file, the block file and don't forget to create the new bed file.

`cat campestre.bed heterophyllum.bed hybrid.bed > campestre_heterophyllum_hybrid.bed`

### Highlight region of interest
To do that you'll have to manually edit the blocks file. You add a code for a color and an asterisk. For example, to highlight in green a region, just add g* to the beginning of the line. Here's an example:

```
LCAMPM00000027237	LHETEM00000033623	LHYBCAMPHETEM00000027343
LCAMPM00000027238	LHETEM00000033628	LHYBCAMPHETEM00000027350
LCAMPM00000027239	LHETEM00000033632	LHYBCAMPHETEM00000027351
g*LCAMPM00000027240	LHETEM00000033639	LHYBCAMPHETEM00000027352
LCAMPM00000027241	LHETEM00000033642	LHYBCAMPHETEM00000027353
LCAMPM00000027242	LHETEM00000033643	LHYBCAMPHETEM00000027354
LCAMPM00000027243	LHETEM00000033644	LHYBCAMPHETEM00000027354
```
If you want to highlight more than one region you can add "r*" to highlight another block in red, for example.

#### GTR1
One example for GTR1.

`python -m jcvi.graphics.synteny gtr1.blocks campestre_heterophyllum_hybrid.bed LG5.blocks.layout --glyphstyle=arrow`

![single gene mapping 3 species example](https://github.com/NBISweden/synteny_plots_with_jcvi/blob/main/figures/gtr1.png)



## Plotting *Lepidium campestre* vs *Arabidopsis thaliana*
This is the same steps as before, but now I'm creating comparison not within *Lepidium*, but between the different *Lepidium* species and *Arabidopsis thaliana*. These plots are more interesting.

### Generate input files for the *A. thaliana* genome
```
conda activate agat
agat_sp_extract_sequences.pl -g Arabidopsis_thaliana.TAIR10.52_reformatwith_agat_convert_sp_gxf2gxf.gff -f Arabidopsis_thaliana.TAIR10.dna.toplevel.fa -t cds -o Arabidopsis_thaliana.TAIR10.cds.fa

agat_convert_sp_gff2bed.pl --gff /projects/sandbox/andre/Lepidium-synteny/data/Arabidopsis_thaliana.TAIR10.52_reformatwith_agat_convert_sp_gxf2gxf.gff -o arabidopsis.bed

conda activate jcvi
python -m jcvi.formats.fasta format /projects/sandbox/andre/Lepidium-synteny/data/Arabidopsis_thaliana.TAIR10.cds.fa arabidopsis.cds
```
### Now generate all the plots
Please notice in the quote block below that you need to manually create some files, it's not a copy and paste case here.
```
#### Pairwise synteny search - campestre - arabidopsis
python -m jcvi.compara.catalog ortholog campestre arabidopsis --no_strip_names
python -m jcvi.compara.catalog ortholog arabidopsis campestre --no_strip_names

#### Plot histogram  - campestre - arabidopsis
python -m jcvi.compara.synteny depth --histogram campestre.arabidopsis.anchors

#### Plot synteny figure - campestre - arabidopsis

##### chrs have different names between assemblies, so be aware of that. lep.arabid.seqids is the file created with these two lines:
LG1,LG2,LG3,LG4,LG5,LG6,LG7,LG8
1,2,3,4,5

#### Prepare Simple anchors file for plotting
python -m jcvi.compara.synteny screen --minspan=10 --simple campestre.arabidopsis.anchors campestre.arabidopsis.anchors.new 
python -m jcvi.compara.synteny screen --minspan=10 --minsize=1 --intrabound=500 --simple campestre.arabidopsis.anchors campestre.arabidopsis.anchors.new

#### Plotting figure 1x1 - campestre - arabidopsis
python -m jcvi.graphics.karyotype lep.arabid.seqids lep.arabid.layout
```
Two examples of plots created with the commands above:
![arabidopsis synteny example](https://github.com/NBISweden/synteny_plots_with_jcvi/blob/main/figures/campestre.arabidopsis.png)

![arabidopsis karyotype example](https://github.com/NBISweden/synteny_plots_with_jcvi/blob/main/figures/campestre.arabidopsis.karyotype.png)


## Known Issues

- You will get this error below. Things will work, don't worry.
`[16:30:28] INFO     Set text.usetex=False. Font styles may be inconsistent.            base.py:403`

- You will also get this error below. This happens at the cluster. If you install things on your local machine, this won't happen. I haven't found a way for it to see Helvetica, or a way to change the default font yet. I ended up fixing all fonts using Illustrator afterwards. This won't change the plot itself, only the legend text, so it's not an analysis-breaking bug, just something annoying. For now.
`WARNING  findfont: Generic family 'sans-serif' not found because none of the following families were found: Helvetica   font_manager.py:1418`

- Some further customization of the plots generated by MCscan are still a bit obscure, there's no documentation for that. So if you don't want to fix legend placement and other texts, you will need to become really good at matplotlib. :)






