## Load required libraries
```
library(phyloseq)
library(dplyr)
library(stringr)
library(tidyr)
```

## Read input files
```
otu <- read.table(file = "yourfeautretable.tsv", sep = "\t", header = TRUE, row.names = 1, skip = 1, comment.char = "")
taxonomy <- read.table(file = "yourtaxonomy.tsv", sep = "\t", header = TRUE, row.names = 1)
metadata_allsamples <- read.table(file = "YourMappingfile.txt", sep = "\t", header = TRUE, row.names = 1)
```

## Process taxonomy data
```
tax.cur <- taxonomy %>%
  separate(Taxon, into = c("Kingdom", "Phylum", "Class", "Order", "Family", "Genus", "Species"), sep = "; ") %>%
  mutate(across(everything(), ~str_replace(.x, "^[a-z]__", ""))) %>%
  mutate(across(everything(), ~replace(.x, is.na(.x) | .x == "__", "")))
```

## Retain taxa IDs as row names 
```
tax.cur <- tax.cur %>% 
  rownames_to_column(var = "TaxaID") %>%
  rowwise() %>%
  mutate(
    Species = ifelse(Species == "", paste0("Unclassified ", Genus), Species),
    Genus = ifelse(Genus == "", paste("Unclassified", Family), Genus),
    Family = ifelse(Family == "", paste("Unclassified", Order), Family),
    Order = ifelse(Order == "", paste("Unclassified", Class), Order),
    Class = ifelse(Class == "", paste("Unclassified", Phylum), Class),
    Phylum = ifelse(Phylum == "", paste("Unclassified", Kingdom), Phylum),
    Kingdom = ifelse(Kingdom == "", "Unclassified Kingdom", Kingdom)
  ) %>%
  ungroup() %>%
  column_to_rownames(var = "TaxaID")
```

## Handle the case where Species is not empty and should include Genus
```
tax.cur <- tax.cur %>%
  mutate(Species = ifelse(Species != "" & Genus != "", paste(Genus, Species), Species))
```
  
  
## Create your phyloseq object
```
OTU <- otu_table(as.matrix(otu), taxa_are_rows = TRUE)
TAX <- tax_table(as.matrix(tax.cur))
SAMPLE <- sample_data(metadata_allsamples)
TREE <- read_tree("tree.nwk")

ps <- phyloseq(OTU, TAX, SAMPLE, TREE)
ps
```

## Transform to relative abundance (RA)
```
ps_RA <- transform_sample_counts(ps, function(x) x / sum(x) * 100)
ps_RA <- prune_taxa(taxa_sums(ps_RA) > 0, ps_RA)
ps_RA
```

## 1. Subset negative controls and calculate mean relative abundance
```
ps_RA_nCs <- subset_samples(ps_RA, Sample_or_Control %in% "Control")
ps_RA_nCs
```

### Feature table into data frame
To be able to detect contaminant features from the negative controls transform first the feature table from your phyloseq object into a data frame
```
table_nCs <- as.data.frame(as.matrix(ps_RA_nCs@otu_table))
table_nCs <- data.frame(FeatureID = row.names(table_nCs), table_nCs)
```

## Calculate mean from negative controls and filter for relative abundance > 1% 
We will work now with main RA bacterial taxa
```
table_nCs_calc <- table_nCs %>%  
  mutate(mean = rowMeans(select(., youfirstnCsID:yourlastnCsIDinthetable), na.rm = TRUE))

table_nCs_calc_1 <- table_nCs_calc %>% filter(mean > 1)
```

## 2. Subset Samples and calculate mean relative abundance
```
ps_RA_Samples <- subset_samples(ps_RA, Sample_or_Control %in% "Sample")
table_Samples <- as.data.frame(as.matrix(ps_RA_Samples@otu_table))
table_Samples <- data.frame(FeatureID = row.names(table_Samples), table_Samples)

table_Samples_calc <- table_Samples %>%  
  mutate(mean_samples = rowMeans(select(., yourfirstsampleID:yourlastsampleIDinthetable)), na.rm = TRUE)
```

## 3. Match FeaturesIDs by "FeatureID" column and calculate ratio between the mean RA of nCs and mean RA of Samples
```
match_tables <- inner_join(table_Samples_calc, table_nCs_calc_1, by = "FeatureID")
match_tables <- match_tables %>%
  mutate(ratio = mean / mean_samples)
```

## 4. Identify contaminants (ratio > 0.9)
ratio above 0.9 --> contaminant = a especific feature found in negative controls with higher RA compared to its mean RA in Samples 
```
Contaminants <- match_tables %>% filter(ratio > 0.9)
write.table(Contaminants, file = "Contaminants.tsv")
```

## 5. Remove contaminants from Samples 
```
table_filt <- table[!(table$FeatureID %in% Contaminants$FeatureID), ] 
table_filt <- subset(table_filt, select = -FeatureID)
```

## Recreate the phyloseq object with filtered data 
Proceed with further analysis for taxonomy visualization and/or microbiome diversities with now filtered data
```
OTU_filt <- otu_table(as.matrix(table_filt), taxa_are_rows = TRUE)
ps_filt <- phyloseq(OTU_filt, TAX, SAMPLE, TREE)
ps_filt
```
