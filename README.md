# ctg-sc-cellplex-10x 
## Nextflow pipeline for processing of 10x multiplex data. GEX + CellPlex on same flowcell

- Run one project at a time only
- Demux -> cellranger -> QC -> delivery
- Supports CellPlex and RNA libraries sequenced on same flowcell.
- CMO IDs used in `cellranger multi` must be specified in samplesheet. See `2. CMO specification` and `1. Samplesheet` sections below.
- Handles delivery via lfs, and sends delivery email

1. Edit your samplesheet to match the example samplesheet. See section `SampleSheet` below
2. Edit the nextflow.config file to fit your project and system. 
3. Run pipeline 
```
nohup nextflow run pipe-sc-cellplex-10x.nf > log.pipe-sc-cellplex-10x.txt &
```
## Driver
- Driver for pipeline located in `.../shared/ctg-pipelines/ctg-sc-cellplex-10x/sc-cellplex-10x-driver`

## Input files

The following files must be in the runfolder to start pipeline successfully.

1. Samplesheet (Preferably `CTG_SampleSheet.sc-cellplex-10x.csv`. (see `SampleSheet` section below)

### 1. Samplesheet (CTG_SampleSheet.sc-cellplex-10x.csv):

- Can have other names than CTG_SampleSheet.**sc-cellplex-10x**.csv - then specify which sheet when starting driver: e.g. `sc-cellplex-10x-driver -s CTG_SampleSheet.2022_102.csv`

#### Example sheet
```
[Header]
metaid,2022_222
email,per.a@med.lu.se
autodeliver,y
[Data]
Lane,Sample_ID,index,Sample_Project,Sample_Species,Sample_Lib,Sample_Pair,Sample,CMO
,CellPlex1,SI-TT-B4,2021_test_Julia_Cellplex_run220221_test,human,gex,1,CellP1,CMO301|CMO302
,CellPlex1_CP,SI-NN-A1,2021_test_Julia_Cellplex_run220221_test,human,cp,1,CellP1
,CellPlex3,SI-TT-B6,2021_test_Julia_Cellplex_run220221_test,human,gex,3,CellP3,CMO303|CMO304
,CellPlex3_CP,SI-NN-D1,2021_test_Julia_Cellplex_run220221_test,human,cp,3,CellP3
```
#### [Header] section
Optional entries. They can be skipped, but recommended to use:
- `metaid` : optional. set to create working folders with this name. otherwise, it will be created based on runfolder name/date.
##### NOT YET SUPPORTED:
- `email` : Email to customer (or ctg staff) that should retrieve email with qc and deliver info upon completion of pipeline. Note: only lu emails works (e.g. @med.lu.se or @lth.se.
- `autodeliver` : set to `y` if email should be sent automatically upon completion. Otherwise, set to `n`.

#### [Data] section

 | Lane | Sample_ID | index | Sample_Species | Sample_Project | Sample_Lib | Sample_Pair | Sample | CMO | cmotype |
 | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | 
 | | Sr1 | SI-GA-D9 | human | 2022_022 | gex | 1 | S1 | CMO301\|CMO302 | multi |
 | | Sr1_CP | SI-GA-H9 | human | 2022_022 | cp | 1 | S1 | | |
 | | Sr2 | SI-GA-C9 | human | 2022_022 | gex | 2 | S2 | CMO303\|CMO304 | single |
 | | Sr2_CP | SI-GA-C9 | human | 2022_022 | gex | 2 | S2 | |

- The nf-pipeline takes the following Columns from samplesheet to use in channels:

- `Sample_ID` : ID of sample. Sample_ID can only contain a-z, A-Z and "_".  E.g space and hyphen ("-") are not allowed! If 'Sample_Name' is present, it will be ignored. IMPORTANT: CellPlex (CP) Sample_ID MUST have corresponding name to GEX Sample_ID, with `_CP` suffix! So if GEX Sample_ID is "Sample_A", CP sample MUST be "Sample_A_CP". If GEX Sample_ID is "Cascvf", CP Sample_ID MUST be "Cascvf_CP".
- `index` : Must use index ID (10x ID) if dual index. For single index, the index sequence works too.
- `Sample_Project` : Project ID. E.g. 2021_033, 2021_192.
- `Sample_Species` : Only 'human'/'mouse'/'custom' are accepted. If species is not human or mouse, set 'custom'. This custom reference genome has to be specified in the nextflow config file. See below how to edit the config file.
- `Sample_Lib` : 'gex'/'cp'. Specify whether sample is RNA (gex) or CellPlex (cp) library. 
- `Sample_Pair` : To match the GEX sample with the corresponding CP sample. e.g. in the example above, sample 'Sr1' is the GEX library, that should be matched with 'Sr1_CP' which is the CP library of the sample.
- `Sample` : A common name for GEX+CellPlex sample. Must be the same for the corresponing GEX and Cellplex samples (such as in example above). So all samples with "Sample_Pair" = 1, must have same "Sample" etc.
- `CMO` : CMO IDs used for the given sample. NOTE: This is specified in the `GEX` sample only in the sheet. MUST be specified for the GEX, otherwise it will crash. Leave the CP sample empty here.
- `cmotype` : `multi` or `single`. 
  - `multi`: If there are multiple CMOs pr sample. The same biological sample were split and a CMO was added to each new vial, and then pooled to one "sample_ID" in samplesheet. This would cause the CMO-"declaration" to be 
  ```
  [samples]
  sample_id,cmo_ids
  sample1,CMO301|CMO302
  sample2,CMO303|CMO304
  ```
  - `single`: Each CMO represent one sample. Typically, different samples were treated with different CMOs, and then pooled. So one declaration will be:
  ```
  [samples]
  sample_id,cmo_ids
  sample1,CMO301
  sample2,CMO302
  sample3,CMO303
  sample4,CMO304
  ```
 Note that for cmotype=`single`, the pipeline will generate the following nomenclature on this "CMO-declaration" in the library.csv file:
```
[samples]
  sample_id,cmo_ids
  ${sid}-301,CMO301
```

In the samplesheet template **below** it will generate these two csv:
- For CellPlex1: Take the `Sample` column: `CellP1`, and it is cmotype=multi:
 ```
  [samples]
  sample_id,cmo_ids
  CellP1,CMO301|CMO302
  ```
- For CellPlex3: Take the `Sample` column: `CellP3`, and it is cmotype=single: (one CMO pr sample)
 ```
  [samples]
  sample_id,cmo_ids
  CellP3-303,CMO303
  CellP3-304,CMO304
  ```

### Samplesheet template

#### Samplesheet name : `CTG_SampleSheet.sc-cellplex-10x.csv`
```
[Header]
metaid,CellPlex_dev_211005_run220221,
autodeliver,y
email,per.a@med.lu.se
[Data]
Lane,Sample_ID,index,Sample_Project,Sample_Species,Sample_Lib,Sample_Pair,Sample,CMO,cmotype
,CellPlex1,SI-TT-B4,2021_test_Julia_Cellplex_run220221_test,human,gex,1,CellP1,CMO301|CMO302,multi
,CellPlex1_CP,SI-NN-A1,2021_test_Julia_Cellplex_run220221_test,human,cp,1,CellP1,
,CellPlex3,SI-TT-B6,2021_test_Julia_Cellplex_run220221_test,human,gex,3,CellP3,CMO303|CMO304,single
,CellPlex3_CP,SI-NN-D1,2021_test_Julia_Cellplex_run220221_test,human,cp,3,CellP3
```

### 2. CMO specification

#### NB: 
- Use only CMO IDs from reference (below) 
- IDs must be identical to the `id` entry in the reference table below.

###### Reference: 
```
id,name,read,pattern,sequence,feature_type
CMO301,CMO301,R2,5P(BC),ATGAGGAATTCCTGC,Multiplexing Capture
CMO302,CMO302,R2,5P(BC),CATGCCAATAGAGCG,Multiplexing Capture
CMO303,CMO303,R2,5P(BC),CCGTCGTCCAAGCAT,Multiplexing Capture
CMO304,CMO304,R2,5P(BC),AACGTTAATCACTCA,Multiplexing Capture
CMO305,CMO305,R2,5P(BC),CGCGATATGGTCGGA,Multiplexing Capture
CMO306,CMO306,R2,5P(BC),AAGATGAGGTCTGTG,Multiplexing Capture
CMO307,CMO307,R2,5P(BC),AAGCTCGTTGGAAGA,Multiplexing Capture
CMO308,CMO308,R2,5P(BC),CGGATTCCACATCAT,Multiplexing Capture
CMO309,CMO309,R2,5P(BC),GTTGATCTATAACAG,Multiplexing Capture
CMO310,CMO310,R2,5P(BC),GCAGGAGGTATCAAT,Multiplexing Capture
CMO311,CMO311,R2,5P(BC),GAATCGTGATTCTTC,Multiplexing Capture
CMO312,CMO312,R2,5P(BC),ACATGGTCAACGCTG,Multiplexing Capture
```


## Pipeline steps:

Cellranger version: cellranger v6.0 

* `delivery_info`: Generates the delivery info csv file needed to create delivery email (see deliverAuto below)
* `parse samplesheets`: Parse samplesheet to fit cellranger mkfastq requirements.
* `generate library csv`: Creates library.csv file based on input samplesheet. This file is used by the "count" step later
* `Demultiplexing` (cellranger mkfastq): Converts raw basecalls to fastq, and demultiplex samples based on index (https://support.10xgenomics.com/single-cell-gene-expression/software/pipelines/6.0/using/multi#cellranger-mkfastq). 
* `FastQC`: FastQC calculates quality metrics on raw sequencing reads (https://www.bioinformatics.babraham.ac.uk/projects/fastqc/). MultiQC summarizes FastQC reports into one document (https://multiqc.info/).
* `Align` + `Counts` + `CellPlexing` (cellranger multi): Aligns fastq files to reference genome, counts genes for each cell/barcode, and extracts CMOs per barcode - Then performs secondary analysis such as clustering and generates the cloupe files (https://support.10xgenomics.com/single-cell-gene-expression/software/pipelines/6.0/using/multi#cellranger-multi).
* `Cellranger count metrics` (bin/ctg-sc-cellplex-GEX-metrics-concat.py and ctg-sc-cellplex-Multiplex-metrics-concat.py): Collects main count metrics (#cells and #reads/cell etc.) from each sample and collect in table. One table for GEX metrics, and several tables for Multiplex metrics.
* `multiQC`: Compile fastQC and cellranger multi metrics in multiqc report
* `md5sum`: md5sum of all generated files
* `deliverAuto`: executes delivery script in bin/ that edits template html delivery email, creates user, pass and executes mutt command to send email
* `pipe_done`: marks pipeline as done (puts ctg.sc-cite-seq-10x.$projid.done in runfolder and logs)


## Container
Currently using the sc-cite-seq-10x container. Contain cellranger v6.
- `sc-cite-seq-10x`: For 10x sc-cite-seq. 
https://github.com/perllb/ctg-sc-cite-seq-10x/tree/master/container

Build container:
NOTE: Environment.yml file has to be in current working directory
```
sudo -E singularity build sc-cite-seq-10x.sif sc-cite-seq-10x-builder 
```

Add path to .sif in nextflow.config

## Output:
* ctg-PROJ_ID-output
    * `qc`: Quality control output. 
        * cellranger metrics: Main metrics summarising the count / cell output 
        * fastqc output (https://www.bioinformatics.babraham.ac.uk/projects/fastqc/)
        * multiqc output: Summarizing FastQC output and demultiplexing (https://multiqc.info/)
    * `fastq`: Contains raw fastq files from cellranger mkfastq.
    * `count-cr`: Cellranger count output. Here you find gene/cell count matrices, feature quantification, secondary analysis output, and more. See (https://support.10xgenomics.com/single-cell-gene-expression/software/pipelines/latest/using/feature-bc-analysis) for more information on the output files.
    * `ctg-md5.PROJ_ID.txt`: text file with md5sum recursively from output dir root    


