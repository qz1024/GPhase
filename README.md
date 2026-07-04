# GPhase: A Phasing Assembly Tool Leveraging an Assembly Graph and Hi-C/Pore-C/Omni-C Data

## GPhase leverages an assembly graph and Hi-C/Pore-C data to facilitate genome assembly phasing, automatically resolves and assigns collapsed sequences, and fills assembly gaps based on the graph structure.

# Table of Contents

- [Installation](#installation)
- [Step1: Mapping Hi-C/Pore-C data to assembly](#step1-mapping-hi-cpore-c-data-to-assembly)
- [Step2: Estimating of the number of contig collapses based on HiFi data and popCNV](#step2-estimating-of-the-number-of-contig-collapses-based-on-hifi-data-and-popcnv)
- [Step3: Running the GPhase scaffolding pipeline](#step3-running-the-gphase-scaffolding-pipeline)
- [Output file](#output-file)
- [Final assembly result](#final-assembly-result)
- [Generate a Hi-C heatmap](#generate-a-hi-c-heatmap)
- [Tips](#tips)
- [Test dataset](#test-dataset)
- [Contact](#contact)



# Installation

To install GPhase, follow these steps:

```
# conda
git clone --depth 1 https://github.com/panlab-bioinfo/GPhase.git
cd GPhase
conda env create -f gphase_environment.yml
conda activate gphase
./gphase -h

# docker
docker pull --platform linux/amd64 tanging1024/gphase:latest
docker run --platform linux/amd64 -v /your/data/path:/your/data/path -w /your/data/path tanging1024/gphase:latest gphase -h

# singularity
singularity pull gphase.sif docker://tanging1024/gphase:latest
singularity exec --bind /your/data/path:/your/data/path gphase.sif gphase

```

> [!WARNING]
>
> GPhase requires the raw, unprocessed hifiasm unitig FASTA (`*.p_utg.fa`) as the assembly input. This FASTA must correspond exactly to the hifiasm primary unitig GFA (`*.p_utg.gfa`), with matching sequence IDs and graph records. Do not rename, reorder, filter, polish, purge, scaffold, or otherwise modify the primary unitigs before running GPhase. The target genome should also have sufficient heterozygosity; if the genome is nearly homozygous, haplotype phasing will have little biological meaning.



# Step1: Mapping Hi-C/Pore-C data to assembly

GPhase supports multiple data types, including Hi-C, Pore-C and Omni-C. It also supports their pairs(pa5) and bam(BAM) format mapping files.

1. Hi-C reads you can using Chromap or other mapping tools, such as BWA, can be used. When using Chromap, if the default MAPQ parameters do not produce satisfactory results, the `--MAPQ-threshold` value can be lowered to include more Hi-C mapping information. When using other mapping software, the BAM files need to be sorted.

```
chromap -i -r asm.fa -o index
chromap --preset hic -x index -r asm.fa -q 0 \
    -1 HiC_1.fq.gz -2 HiC_2.fq.gz \
    --remove-pcr-duplicates -t 64 --SAM -o map.chromap.sam
samtools view -@ 64 -bh map.chromap.sam -o map.chromap.bam
```

1. For contact-pair long reads, including Pore-C and CiFi, we recommend using `contact_pair_pipeline.sh` as the default workflow. This script always performs read mapping internally with minimap2, then converts the alignments to a Hi-C-like BAM file `map.concatemer2pe.bam` with `concatemer2pe.py`. That BAM can be passed directly to `gphase pipeline -m`. You can run it directly or through `gphase contact-pair`. Use `-x map-ont` for Pore-C/ONT reads and `-x map-hifi` for CiFi/HiFi-based contact-pair reads. The `-o` option specifies the output directory prefix, and the final BAM path is `<prefix>/map.concatemer2pe.bam`. For all `contact-pair` parameters, see [doc/README.md#gphase-contact-pair](doc/README.md#gphase-contact-pair).

```
/path/to/GPhase/gphase contact-pair \
    asm.fa \
    reads1.fq.gz reads2.fq.gz \
    -x map-ont \
    -o contact_pair \
    -t 32
```

The previous Pore-C workflow based on [PPL Toolbox](https://github.com/versarchey/PPL-Toolbox) is still available as a backup workflow. If you need to reproduce the results reported in the paper, you can use this PPL-based process to generate the final pairs file `map.PPL.pairs` and then input it into GPhase. For all `ppl` parameters, see [doc/README.md#gphase-ppl](doc/README.md#gphase-ppl).

```
/path/to/GPhase/gphase ppl \
    -g asm.fa \
    -f reads.fq.gz \
    -o PPL
```



# Step2: Estimating of the number of contig collapses based on HiFi data and popCNV

The popCNV_pipeline.sh script estimates the copy number of collapsed contigs collapse based on HiFi data using the popCNV software. The file used by popCNV for GPhase input is `collapse_num.txt` : popcnv/06.genes.round.cn. For details, see [popCNV](https://github.com/sc-zhang/popCNV). For all `popcnv` parameters, see [doc/README.md#gphase-popcnv](doc/README.md#gphase-popcnv).

```
/path/to/GPhase/gphase popcnv \
    -f asm.fa \
    -p output_prefix \
    -t 32 \
    -r reads.fq.gz
```



# Step3: Running the GPhase scaffolding pipeline

1. `asm.fa` :  Genome assembly file in FASTA format (unitigs).
2. `p_utg.gfa` : Assembly graph file in GFA format.
3. `collapse_num.txt` : File with contig collapse information (from popCNV: popcnv/06.genes.round.cn).
4. `map.chromap.bam` : pairs/bam file with mapped Hi-C/Pore-C/CiFi reads. For contact-pair data, the recommended input is `contact_pair/map.concatemer2pe.bam`, where `contact_pair` is the `-o` output directory/prefix used by `gphase contact-pair`; for paper reproduction, `PPL/map.PPL.pairs` can also be used.
5. `n_chr` : Number of chromosomes.
6. `n_hap` : Number of haplotypes.
7. `p` : Prefix for output files.

```
/path/to/GPhase/gphase pipeline\
    -f asm.fa \
    -g genome.bp.p_utg.gfa \
    -c collapse_num.txt \
    -m map.chromap.bam \
    --n_chr 12 \
    --n_hap 4 \
    -p output_prefix \
    --rescue \
    --min_len 50
```

For more parameters, please refer to `gphase pipeline -h` or [doc/README.md#gphase-pipeline](doc/README.md#gphase-pipeline).

Required parameters:

- `-f` : Genome assembly file in FASTA format (unitigs).
- `-g` : Assembly graph file in GFA format.
- `-c` : File with contig collapse information (from popCNV: `popcnv/06.genes.round.cn`).
- `-m` : Hi-C/Pore-C/Omni-C/CiFi mapping file in `.bam` or `.pairs` format.
- `--n_chr` : Number of chromosomes.
- `--n_hap` : Number of haplotypes.
- `-p` : Prefix for output files. Only the character `.`, numbers, and uppercase/lowercase letters are allowed (`[a-zA-Z0-9.]`).

Below are some of the more important optional parameters:

- `--cluster_q` : Hi-C/Pore-C/Omni-C/CiFi mapping quality score threshold (MAPQ) used during clustering. The default is `1`. This applies when the input is a BAM file.
- `--scaffold_q` : Hi-C/Pore-C/Omni-C/CiFi mapping quality score threshold (MAPQ) used during scaffolding. The default is `0`. This applies when the input is a BAM file.
- `--hap_pm` : The threshold for the intensity parameter of homologous sequence identification. The default is `0.7`. If the heterozygosity of the assembled species is high, `0.6` can be used; if the heterozygosity of the species is low, `0.8` can be used.
- `--chr_pm` : Similarity threshold for chromosome-level partig clustering. The default is `0.95`.
- `--nor_hic` : Normalization mode for 3C link connections. Choices are `no`, `ratio`, and `length`; default is `ratio`.
- `--min_len` : Minimum scaffold length in kb in HapHiC sorting. The default is `50`.
- `--thread` : Number of parallel processes used in scaffolding. The default is `12`.



# Output file

GPhase will output a folder named gphase_output, which will generate the following four folders in sequence.

- `preprocessing` : Data preprocessing
- `cluster_chr` : Results of chromosome clustering
- `cluster_hap` : Haplotype clustering results within each chromosome
- `scaffold_hap` : Scaffolding results for each haplotype within each chromosome



# Final assembly result

The final assembly result file is located in the scaffold_hap folder and mainly contains the following:

- `gphase_final.agp` : unitig level assembly result agp file
- `gphase_final.fasta` : unitig level assembly result fasta file
- `gphase_final_rescue.agp` :  unitig level assembly result agp file after rescue
- `gphase_final_ctg2utg.txt` : correspondence between unitig and contig
- `gphase_final_contig.fasta` : Contig-level fasta sequence
- `gphase_final_contig.agp` : contig level assembly result agp file
- `gphase_final_contig_scaffold.fasta` : contig level assembly result fasta file



# Generate a Hi-C heatmap

GPhase provides two workflows for generating Hi-C heatmaps and preparing assemblies for manual correction in Juicebox.

### 1. Method1: Generate using the original unitig-level FASTA

Because collapsed sequences appear multiple times in the assembly, duplicated unitigs must be distinguished in the AGP, FASTA, and Hi-C mapping files before heatmap generation. This workflow first assigns a fixed suffix to duplicated collapsed unitigs (in both AGP and FASTA), remaps Hi-C reads with `mapQ:0` (retaining multi-mappings), and then generates a Hi-C heatmap with Juicer.

> **Note:** The `asm.fa` input to `rename_collapse_agp_pairs_fasta.py` must be the **original unitig-level FASTA**, not the GPhase assembly output.

```
# Rename AGP and unitigs
python /Path/to/GPhase/scaffold_hap/rename_collapse_agp_pairs_fasta.py \
    gphase_final.agp asm.fa rename --no-hic 

# Remap Hi-C reads
chromap -i -r rename.fa -o reindex
chromap --preset hic -x reindex -r rename.fa -q 0 \
    -1 HiC_1.fq.gz -2 HiC_2.fq.gz \
    --remove-pcr-duplicates -t 64 -o remap.chromap.pairs

# Generate Hi-C heatmap
bash /Path/to/GPhase/scaffold_hap/juicebox.sh \
-f rename.fa \
-a rename.agp \
-p remap.chromap.pairs \
-o final_hic -g /Path/to/GPhase
```

### 2. Method2: Generate using GPhase contig-level assembly (`gphase_final_contig.fasta`)

This workflow uses the contig-level assembly produced by GPhase `gphase_final_contig.fasta` and `gphase_final_contig.agp` in the `scaffold_hap` folder). Because GPhase has already resolved collapsed sequences into separate contigs, no additional renaming step is required. Simply remap Hi-C reads to the contig-level reference and generate the heatmap with `juicebox.sh`. 

```
# Remap Hi-C reads
chromap -i -r gphase_final_contig.fasta -o reindex
chromap --preset hic -x reindex -r gphase_final_contig.fasta -q 0 \
    -1 HiC_1.fq.gz -2 HiC_2.fq.gz \
    --remove-pcr-duplicates -t 64 -o remap.chromap.pairs

# Generate Hi-C heatmap
bash /Path/to/GPhase/scaffold_hap/juicebox.sh \
-f gphase_final_contig.fasta \
-a gphase_final_contig.agp \
-p remap.chromap.pairs \
-o final_hic -g /Path/to/GPhase
```



### **Output files in the Hi-C heatmap pipeline**

The commands above produce the following files for Hi-C heatmap generation, Juicebox visualization, and downstream export:

`rename_collapse_agp_pairs_fasta.py` *(Method 1 only)*

- `rename.fa` — FASTA with original unitig names (e.g. `utg000404l`). Duplicated collapsed unitigs are suffixed with `_dup1`, `_dup2`, etc.
- `rename.agp` — AGP consistent with `rename.fa`.

`chromap` *(both methods)*

- `remap.chromap.pairs` — Hi-C pairs remapped to the reference.

`juicebox.sh`

- `final_hic.hic` — Hi-C contact map for **[Juicebox](https://github.com/aidenlab/Juicebox)**.
- `final_hic.assembly` — Juicebox assembly file with sequential ctg-style IDs (e.g. `ctg00000001.1`).
- `final_hic.liftover.agp` — Maps each Juicebox ctg ID (column 1) to the underlying reference sequence (column 6) and coordinates (columns 7–8).

> **Important:** Juicebox uses `ctg********.1`-style names (`.assembly` / `.hic`), while the reference FASTA uses `utg******l` (Method 1) or `ctg******l` (Method 2) names. `final_hic.liftover.agp` records the correspondence between these two naming systems.

Example lines from final_hic.liftover.agp:
```
# Method 1 (unitig-level rename.fa):
ctg00000007.1   1       37974   1       W       utg007901l_dup1 1       37974   +
ctg00000008.1   1       27515   1       W       utg007594l      1       27515   +

# Method 2 (contig-level gphase_final_contig.fasta):
ctg00000001.1    1      42223   1       W       ctg000001l      1       42223    +
```

## Manual operation in Juicebox

1. Open [Juicebox Assembly Tools (JBAT)]([https://github.com/aidenlab/Juicebox](https://github.com/aidenlab/Juicebox)).
2. Load `final_hic.hic` and `final_hic.assembly`.
3. Manually correct mis-joins, inversions, and ordering as needed.
4. Export the modified results as `final_hic.review.assembly` (or any `*.review.assembly` name).

## Export corrected assembly

After manual correction in Juicebox, export the corrected assembly with `juicer post`. This applies `final_hic.review.assembly` onto the reference FASTA using `final_hic.liftover.agp` as the coordinate map. Use the same reference FASTA that was supplied to `juicebox.sh`:

**Method 1 (unitig-level):**

```bash
/path/to/GPhase/src/HapHiC/utils/juicer post \
    -o corrected \
    final_hic.review.assembly \
    final_hic.liftover.agp \
    rename.fa
```

**Method 2 (contig-level):**

```bash
/path/to/GPhase/src/HapHiC/utils/juicer post \
    -o corrected \
    final_hic.review.assembly \
    final_hic.liftover.agp \
    gphase_final_contig.fasta
```

**Outputs:**

- `corrected.FINAL.fa` — scaffold-level corrected sequences
- `corrected.FINAL.agp` — corresponding AGP



# Tips
1. The `cluster_q` and `scaffold_q` parameters are only enabled when the input mapping file format is BAM. If using pairs, the `mapQ` parameter of the mapping software (e.g., Chromap) can be adjusted, but it is not recommended to set `mapQ` to 0, as this will affect the accuracy of the phasing due to multiple-mapping.
2. When assembling `polyploids`, it is recommended to use `unitig-level` assembly `sequences` and `graph` for phasing assembly. Generally, unitig results in fewer errors compared to contig. Furthermore, using unitig allows for the utilization of more assembly graph information, leading to better assembly results.
3. GPhase can largely solve the problem of sequence collapse during assembly, but it cannot solve the problem of `large fragments collapsing` in haplotypes.

# Test dataset

To help you quickly verify the installation and use of the software, we provide a small test dataset. This dataset contains input data that demonstrates the core functionality of the software. You can download it from this link [https://drive.google.com/drive/folders/1M_ZlSHBTDwtCHGrUI6uMCVutfIweECaY?usp=sharing](https://drive.google.com/drive/folders/1M_ZlSHBTDwtCHGrUI6uMCVutfIweECaY?usp=sharing)

Use the following command to run the test dataset

```
tar -zxvf test_dataset.tar.gz
export PATH=$PATH:/path/to/GPhase
bash run_gphase.sh
```



# Contact

This software is developed by Professor Wei-Hua Pan's team at the Shenzhen Institute of Genome Research, Chinese Academy of Agricultural Sciences. 

If you have any questions or concerns while using the software, please submit an issue in the repository or contact us through the following methods:

### Email:



#### Prof. Pan: [panweihua@caas.cn](mailto:panweihua@caas.cn)



#### Du Wenjie: [duwenjie1024@163.com](mailto:duwenjie1024@163.com)

