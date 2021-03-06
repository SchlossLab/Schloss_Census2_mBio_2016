Getting the archaeal sequence data and metadata
===============================================

Get data out of ARB
-------------------

This is redundant with what we saw for the bacterial data, but for completion...

> To create SSU Ref (ARB file), all sequences below 1,200 bases for Bacteria and Eukarya and below 900 bases for Archaea or an alignment identity below 70 or an alignment quality value below 50 have been removed from SSU Parc. All sequences with a Pintail value &lt; 50 or an alignment quality value &lt; 75 have been assigned to color group 1 in ARB (red). All Living Tree Project or StrainInfo typestrains have been assigned to color group 2 in ARB (light blue). From <http://www.arb-silva.de/documentation/release-119/>

``` bash
wget http://www.arb-silva.de/fileadmin/silva_databases/release_119/ARB_files/SSURef_119_SILVA_14_07_14_opt.arb.tgz
wget http://www.arb-silva.de/fileadmin/silva_databases/release_119/ARB_files/SSURef_119_SILVA_14_07_14_opt.arb.tgz.md5
tar xvzf SSURef_119_SILVA_14_07_14_opt.arb.tgz
```

Within ARB, we will exclude color group 1, chloroplasts, and mitochondria.

To get good sequences...

-   Go to search window
-   Set Search\_Fields to "ARB\_color" and use "1"; click on the equal sign to make it not equal
-   Hit Mark Listed, Unmark Rest button (N=1493493)

The problem with the taxonomies is that the sequences don't all have taxonomies. Need to figure out which taxonomy to base the analysis on. Do the following searches...

-   ARB\_color != 1 & tax\_rdp == "*Archaea*" (N=40288)
-   ARB\_color != 1 & tax\_greengenes == "*Archaea*" (N=19795)
-   ARB\_color != 1 & tax\_slv == "*Archaea*" (N=17641)
-   ARB\_color != 1 & tax\_embl == "*Archaea*" (N=44574)
-   ARB\_color != 1 & (tax\_rdp | tax\_greengenes | tax\_embl | tax\_slv) (N=46387 )

Let's stick with the archaeal sequences that have an RDP taxonomy and we will analyze them in RDP space, but we can always go back to the taxonomy provided by the other systems if we'd like. A quick spot checking suggests that the RDP taxonomy information is actually richer than that of the EBML taxonomy information.

-   Return to Search and Query window
-   Change search field to "ARB\_color", enter "1", and click the equal sign to be not-equal
-   Change the second search field to "tax\_rdp" and set it to "*Archaea*"
-   Hit "Search"
-   Hit Mark Listed Unmark Rest (N=40,288)
-   Click "Write to Fields of Listed", select the "remark" field, and enter "good archaea"
-   Click "Write"
-   In main ARB window go File-&gt;Export-&gt;Export to external format
-   Select Compress -&gt; "Vertical Gaps" and fasta\_mothur.eft as the format
-   Rename `noname.fasta` to `archaea.fasta`

Now we need the taxonomy information.

-   Go Tree -&gt; NDS
-   Click "name", "acc", "tax\_rdp". The "tax\_rdp" field should have 250 characters
-   Unclick everything else
-   Click "Close"
-   Go File-&gt;Export-&gt;Export fields
-   Set the file name to "archaea.taxonomy" and Column output to "TAB separated"
-   Click "SAVE"

Finally, let's save the database by doing File -&gt; Quick Save Changes and then quit out of ARB.

Format taxonomy file
--------------------

The next thing we need to do is to clean up the archaea.taxonomy.nds file to make it into a proper, mothur compatible archaea.taxonomy file. First we want to remove the names of any sub taxa (e.g. suborder). To do this we need to get the RDP list of names and the taxonomic level they belong to...

``` bash
wget -N http://sourceforge.net/projects/rdp-classifier/files/RDP_Classifier_TrainingData/RDPClassifier_16S_trainsetNo10_rawtrainingdata.zip
unzip -o RDPClassifier_16S_trainsetNo10_rawtrainingdata.zip
mv RDPClassifier_16S_trainsetNo10_rawtrainingdata/* data/references/
rm -rf RDPClassifier_16S_trainsetNo10_rawtrainingdata*
```

Now we're read to do some R'ing to get the taxonomy file formatted properly...

``` r
tax_data <- read.table(file="data/mothur/archaea.taxonomy.nds", sep="\t")

#get the names to match the fasta file
seq_names <- paste(tax_data$V2, tax_data$V1, sep=".")

taxonomy <- gsub(" ", "_", tax_data$V3)  #convert any spaces to underscores
taxonomy <- gsub("[^;]*_incertae_sedis$", "", taxonomy) #remove the unknowns
taxonomy <- gsub('\"', '', taxonomy) #remove quote marks

#now we need to pull out the taxon names for the sub taxa levels
levels <- read.table(file="data/references/trainset10_db_taxid.txt", sep="*", stringsAsFactors=F)
subs <- levels[grep("sub", levels$V5),] #get the sub taxa names
sub.names <- subs$V2

tax.split <- strsplit(taxonomy, split=";")  #split the taxonomy string by semicolon

#function to see which taxa names in a list are found in the sub taxa list
remove.subs <- function(tax.vector){
    return(tax.vector[which(!tax.vector %in% sub.names)])
}

#apply sub taxa finding function to all sequences
no.subs <- lapply(tax.split, remove.subs)

#merge data together by semicolon for each sequence
no.subs.str <- unlist(lapply(no.subs, paste, collapse=";"))

#Tack a semicolon on to the end of each sequence
no.subs.str <- paste0(no.subs.str, ";")

write.table(cbind(seq_names, no.subs.str), "data/mothur/archaea.tax", row.names=F,
                                                col.names=F, quote=F, sep="\t")
```

Get good sequences
------------------

Now we need to know the start/end position of the sequences so that we can make sure the reads overlap the same alignment space.

``` bash
mothur "#summary.seqs(fasta=data/mothur/archaea.fasta, processors=12)"
```

                Start    End    NBases  Ambigs  Polymer NumSeqs
    Minimum:    1        4058   900     0       4   1
    2.5%-tile:  357      4692   905     0       5   1008
    25%-tile:   365      4783   935     0       5   10073
    Median:     379      7409   1262    0       5   20145
    75%-tile:   456      7729   1349    0       6   30217
    97.5%-tile: 1746     7984   1470    4       7   39281
    Maximum:    2321     8711   2174    46      32  40288
    Mean:       513.659  6698.2 1185.3  0.3     5.44244
    # of Seqs:  40288

Notes... \* Will want to get rid of sequences with large number of Ns in them and the ridiculously long homopolymrs - how large?

``` r
data <- read.table(file="data/mothur/archaea.summary", header=T, row.names=1)
quantile(data$start, probs=seq(0.9,1,0.01))
start <- 2000

quantile(data$end, probs=seq(0,0.1,0.01))
end <- 4058

quantile(data$nbases, probs=seq(0.0,0.1,0.01))
min_length <- 900

quantile(data$ambigs, probs=seq(0.9,1.0,0.01))
max_ambig <- 2

quantile(data$polymer, probs=seq(0.9,1.0,0.01))
max_polymer <- 8

trimmed <- data[data$start <= start & data$end >= end &
                    data$polymer <= max_polymer & data$nbases > min_length &
                    data$ambigs <= max_ambig,]
nrow(trimmed)
summary(trimmed$nbases)
```

Now let's run these parameters using mothur.

``` bash
mothur "#set.current(processors=8);
        screen.seqs(fasta=archaea.fasta, taxonomy=archaea.tax, start=2000, end=4058, maxambig=2, maxhomop=8, minlength=900, inputdir=data/mothur/, outputdir=data/mothur/);"
```

Single cell genomics data
-------------------------

Using 16S rRNA gene sequences from the single cell genomics projects we'd like to see whether that type of data has had a meaningful impact on the trajectory of the microbial census. The LSU team scraped these 251 sequences from a database and have provided them to me as a fasta file. Some of them are quite short and none of them are aligned. Let's go ahead and align them to the SILVA alignment and then screen them to keep sequences that overlap with the SILVA sequences.

``` bash
mothur "#set.dir(input=data/mothur, output=data/mothur);
        align.seqs(fasta=single_cell.archaea.fasta, reference=archaea.good.fasta);
        screen.seqs(fasta=current, start=2000, end=4058, maxambig=2, maxhomop=8, minlength=900);
        classify.seqs(fasta=current, reference=data/references/trainset10_082014.pds.fasta, taxonomy=data/references/trainset10_082014.pds.tax, cutoff=80, processors=8)"
```

Pooling the data
----------------

Now we want to merge the SILVA and single cell genomics 16S rRNA gene sequence collections. We'll also filter the sequences to overlap in the same alignment space, unique them, and precluster them...

``` bash
cat data/mothur/archaea.good.fasta data/mothur/single_cell.archaea.good.align > data/mothur/all_archaea.align
cat data/mothur/archaea.good.tax data/mothur/single_cell.archaea.good.pds.wang.taxonomy > data/mothur/all_archaea.taxonomy

mothur "#set.dir(input=data/mothur, output=data/mothur);
        filter.seqs(fasta=all_archaea.align, vertical=T, trump=.);
        unique.seqs();
        pre.cluster(fasta=current, name=current, diffs=9);"

#need to make the taxonomy file match the fasta and names files...
cut -f 1 data/mothur/all_archaea.filter.unique.precluster.names > data/mothur/precluster.accnos
mothur "#set.dir(input=data/mothur/, output=data/mothur/);
        get.seqs(taxonomy=all_archaea.taxonomy, accnos=precluster.accnos)"
```

Now we're ready to cluster the sequences. We'll do it by the classic approach without any cutoffs and see what we get. Let's start by splitting things at the phylum level and cluster from there...

``` bash
mothur "#set.dir(input=data/mothur/, output=data/mothur/);
        cluster.split(fasta=all_archaea.filter.unique.precluster.fasta, name=all_archaea.filter.unique.precluster.names, taxonomy=all_archaea.pick.taxonomy, taxlevel=2, classic=T, processors=8)"
```

Get metadata
------------

We'd like to use some metadata from the database to characterize the changes in the representation of sequences over time, by environment, methods, etc. The fields housed within SILVA are available at their [website](http://www.arb-silva.de/fileadmin/arb_web_db/release_115/Fields_description/SILVA_description_of_fields_16_06_2013.htm) and there is an [FAQ](http://www.arb-silva.de/documentation/faqs/) on their site as well. There were a number of fields that I didn't think were relevant and instead focused on 30 fields that I thought could help the cause. I marked these fields in the NDS feature and extracted them to `archaea_metadata.nds`. We need to tweak this file slightly in R to get it into a format that we can use.

``` r
metadata_fields <- c(
    "name", #internal ARB database ID, do not change!
    "acc", #accession number
    "bio_material", #identifier for the biological material from which the nucleic acid sequenced was obtained
    "clone", #clone from which the sequence was obtained
    "clone_lib", #clone library from which the sequence was obtained
    "collected_by", #name of the person who collected the specimen
    "collection_date", #date that the sample/specimen was collected
    "country", #geographical origin of sequenced sample
    "culture_collection", #institution code and identifier for the culture from which the nucleic acid sequenced was obtained, with optional collection code
    "date", #entry creation and update date separated by ;
    "description", #description
    "embl_class", #describes the data class in EMBL, e.g. CON: Constructed, WGS: Whole Genome Shotgun, see www.ebi.ac.uk/ena/about/embl_bank_format
    "env_sample", #identifies sequences derived by direct molecular isolation from a bulk environmental DNA sample (by PCR with or without subsequent cloning of the product, DGGE, or other anonymous methods) with no reliable identification of the source organism. Indicated by ‘yes’ in the ARB files
    "full_name", #organism species
    "host", #natural (as opposed to laboratory) host to the organism from which sequenced molecule was obtained.
    "identified_by", #name of the taxonomist who identified the specimen
    "insdc", #the International Nucleotide Sequence Database Collaboration (INSDC) Project Identifier that has been assigned to the entry
    "isolate", #individual isolate from which the sequence was obtained
    "isolation_source", #describes the physical, environmental and/or local geographical source of the biological sample from which the sequence was derived
    "journal", #reference location
    "lat_lon", #geographical coordinates of the location where the specimen was collected
    "publication_doi", #cross-reference DOI number
    "pubmed_id", #cross-reference Pubmed ID
    "strain", #strain from which the sequence was obtained. (t) or [T]: typestrains, [C]: cultivated, [G]: genomes
    "sub_species", #name of sub-species of organism from which sequence was obtained
    "submit_author", #submission authors from reference location
    "submit_date", #submission date from reference location
    "title", #reference title
    "depth_slv", #depth is the vertical distance below surface, e.g. for sediment or soil samples depth is measured from sediment or soil surface, respectively. Depth can be reported as an interval for subsurface samples.
    "habitat_slv" #habitat description according to EnvO-Lite
)


metadata <- read.table(file="data/mothur/archaea_metadata.nds", sep="\t",
                        stringsAsFactors=FALSE, col.names=metadata_fields,
                        fill=TRUE, na.strings="", quote="")
rownames(metadata) <- paste(metadata$acc, metadata$name, sep=".")
metadata <- metadata[,-c(1,2)]
```

Now we want to read in `archaea.bad.accnos` so that we can figure out which sequences to cull from the metadata table.

``` r
if(!"openxlsx" %in% rownames(installed.packages())){
    install.packages("openxlsx")
}
library("openxlsx")

bad_accnos_data <- read.table(file="data/mothur/archaea.bad.accnos",
                    col.names=c("name.accnos", "reason"), stringsAsFactors=F)
bad_accnos <- bad_accnos_data$name.accnos
metadata_good <- metadata[-which(rownames(metadata) %in% bad_accnos),]

categories_sheet <- read.xlsx("data/raw/FinalCategories.xlsx", sheet=1, startRow=1, colNames=TRUE)
categories <- categories_sheet[,1]
names(categories) <- categories_sheet[,2]

categories_sheet_missing <- read.xlsx("data/raw/ExtraCategoriesArchaea.xlsx", sheet=1, startRow=1, colNames=FALSE)
categories_missing <- categories_sheet_missing[,"X1"]
names(categories_missing) <- categories_sheet_missing[,"X2"]

categories <- c(categories, categories_missing)


metadata_good$category <- toupper(categories[metadata_good$isolation_source])

extra <- table(metadata_good$isolation_source[is.na(metadata_good$category)])

if(length(extra) == 0){
    write.table(extra, file="missing.txt", row.names=F, col.names=F, quote=FALSE,  sep="\t")
}

write.table(metadata_good, file="data/mothur/archaea.good.metadata", quote=TRUE, sep="\t")
```

We'd also like to get the metadata from the single cell data. We have an `xlsx` that the LSU group pulled together that we'll read in and concatenate to the end of `archaea.good.metadata`.

``` r
spread_sheet <- read.xlsx("data/raw/FormattedArchaealMetadata.xlsx", sheet=1, startRow=1, colNames=TRUE)
rownames(spread_sheet) <- spread_sheet$name
spread_sheet <- spread_sheet[,-c(1,2)]

metadata_genome_id <- gsub(".* (P?SCGC \\S*).*", "\\1", spread_sheet$full_name)
metadata_genome_id[metadata_genome_id=="SCGC AAA011-J2"] <- "SCGC AAA011-J02"

#need to fix mapping between genome and gene sequences
fasta <- scan("data/mothur/single_cell.archaea.fasta", what="", quiet=TRUE, sep="\n")
headers <- fasta[grepl(">", fasta)]
seq_numbers <- gsub(">(\\d*) .*", "\\1", headers)
parse_header <- gsub(".* (P?SCGC \\S*) .*", "\\1", headers)
parse_header <- gsub(".*SAG-(\\S*) .*", "SCGC \\1", parse_header)

names(seq_numbers) <- parse_header

#recreate a metadata table using sequence names and spread_sheet
full_spread_sheet <- spread_sheet[names(seq_numbers) %in% metadata_genome_id,]
rownames(full_spread_sheet) <- seq_numbers

bad_accnos_data <- read.table(file="data/mothur/single_cell.archaea.bad.accnos", col.names=c("name.accnos", "reason"),  stringsAsFactors=FALSE)
bad_accnos <- as.character(bad_accnos_data$name.accnos)

spread_sheet_good <- full_spread_sheet[-which(rownames(full_spread_sheet) %in% bad_accnos),]
colnames(spread_sheet_good) <- gsub("\\.", "_", colnames(spread_sheet_good))

write.table(spread_sheet_good[,colnames(metadata_good)], file="data/mothur/archaea.good.metadata", quote=TRUE, sep="\t", col.names=F, append=T)
```

Let's now take the big metadata file that we have and concatenate on the OTU assignment and taxonomy information:

``` r
#here we'll read in the list file and extract the OTU assignments for the 0.03
#cutoff
list_text <- scan(file="data/mothur/all_archaea.filter.unique.precluster.an.list", sep="\n", what="")[5]

#let's remove the otu label and the number of OTUs at this cutoff
list_otus <- unlist(strsplit(list_text, "\t"))[-c(1,2)]
n_otus <- length(list_otus)

#this function will take the sequence names in an OTU, split them up and then
#assign them the number of that OTU
split_otu <- function(counter){
    seq_vector <- unlist(strsplit(list_otus[counter], ","))
    n_seqs <- length(seq_vector)
    otu_number <- rep(counter, n_seqs)
    names(otu_number) <- seq_vector
    otu_number
}

#now we get the OTU number for each sequence in the dataset
otu_assignment <- unlist(lapply(1:n_otus, split_otu))


#let's get the sequence classification data and remove the confidence score data
taxonomy_file <- read.table(file="data/mothur/all_archaea.taxonomy", header=F, row.names=1, stringsAsFactors=T)
taxonomy_data <- gsub("\\(\\d*\\)", "", taxonomy_file$V2)
names(taxonomy_data) <- rownames(taxonomy_file)


#let's get the metadata file and paste in the OTU and taxonomy data
metadata <- read.table(file="data/mothur/archaea.good.metadata", header=T, stringsAsFactors=FALSE)
metadata$otu <- otu_assignment[as.character(rownames(metadata))]
metadata$taxonomy <- taxonomy_data[as.character(rownames(metadata))]


#let's make sure all of the category names are upper cased
metadata$category <- toupper(metadata$category)

#correct typos:
metadata$category[metadata$category == 'AQD'] <- 'AQO'
metadata$category[metadata$category == 'AQHS'] <- 'AQH'
metadata$category[metadata$category == 'AQS'] <- 'AQMS'
metadata$category[metadata$category == 'VD'] <- 'BD'
metadata$category[metadata$category == 'BM'] <- 'BI'
metadata$category[metadata$category == 'AH'] <- 'AQH'


#... and we output the metadata file...
write.table(metadata, file="data/mothur/archaea.final.metadata", sep="\t")

#... and we output a group file to use for splitting up the list file into the individual categories...
write.table(file="data/mothur/archaea.all_category.groups", x=cbind(rownames(metadata), metadata$category), quote=FALSE, row.names=FALSE, col.names=FALSE, sep="\t")

#... and we output a group file to use for splitting up the list file into the pooled categories...
#   For archaea: AQH, AQM, AQMS, and ZV into one. the rest into the other
#   per Rene 6/24/2015...
pool1 <- c("AQH", "AQM", "AQMS", "ZV")
pool_categories <- rep(NA, nrow(metadata))
pool_categories[metadata$category %in% pool1] <- "pool1"
pool_categories[!metadata$category %in% pool1] <- "pool2"
pool_categories[is.na(metadata$category)] <- "NA"
write.table(file="data/mothur/archaea.pool_category.groups", x=cbind(rownames(metadata), pool_categories), quote=FALSE, row.names=FALSE, col.names=FALSE, sep="\t")


#... and we pool the categories by environment
env_categories <- rep(NA, nrow(metadata))
env_categories[metadata$category == "AE"] <- "aerosol"
env_categories[metadata$category == "OT"] <- "other"
env_categories[metadata$category %in% c("AQB", "AQBS", "AQF", "AQFS", "AQH", "AQI", "AQM", "AQMS", "AQO", "AQS")] <- "aquatic"
env_categories[metadata$category %in% c("BD", "BF", "BI", "BO", "BP")] <- "built"
env_categories[metadata$category %in% c("PO", "PR", "PS")] <- "plant"
env_categories[metadata$category %in% c("SA", "SD", "SO", "SP")] <- "soil"
env_categories[metadata$category %in% c("ZA", "ZN", "ZO", "ZV")] <- "zoonotic"
write.table(file="data/mothur/archaea.env_category.groups", x=cbind(rownames(metadata), env_categories), quote=FALSE, row.names=FALSE, col.names=FALSE, sep="\t")
```

One thing we'd like to look at are the rarefaction curves for each set of group files and the entire dataset. To do this we'll run the mothur command like so..

``` bash
mothur "#set.dir(input=data/mothur, output=data/mothur);
        make.shared(list=all_archaea.filter.unique.precluster.an.list, group=archaea.all_category.groups, label=unique-0.03-0.05-0.10-0.20);
        system(mv data/mothur/all_archaea.filter.unique.precluster.an.shared data/mothur/all_archaea.all_categories.shared);
        rarefaction.single(shared=all_archaea.all_categories.shared);
        summary.single(shared=all_archaea.all_categories.shared, calc=nseqs-sobs-coverage)"

mothur "#set.dir(input=data/mothur, output=data/mothur);
        make.shared(list=all_archaea.filter.unique.precluster.an.list, group=archaea.pool_category.groups, label=unique-0.03-0.05-0.10-0.20);
        system(mv data/mothur/all_archaea.filter.unique.precluster.an.shared data/mothur/all_archaea.pool_category.shared);
        rarefaction.single(shared=all_archaea.pool_category.shared);
        summary.single(shared=all_archaea.pool_category.shared, calc=nseqs-sobs-coverage)"

mothur "#set.dir(input=data/mothur, output=data/mothur);
        make.shared(list=all_archaea.filter.unique.precluster.an.list, group=archaea.env_category.groups, label=unique-0.03-0.05-0.10-0.20);
        system(mv data/mothur/all_archaea.filter.unique.precluster.an.shared data/mothur/all_archaea.env_category.shared);
        rarefaction.single(shared=all_archaea.env_category.shared);
        summary.single(shared=all_archaea.env_category.shared, calc=nseqs-sobs-coverage)"

mothur "#rarefaction.single(list=data/mothur/all_archaea.filter.unique.precluster.an.list, label=unique-0.03-0.05-0.10-0.20);"
```
