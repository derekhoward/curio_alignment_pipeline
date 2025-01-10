# Curio Seeker Pipeline Setup 
This guide walks through setting up and running the [Curio Seeker v3.0.0 pipeline for spatial transcriptomics](https://knowledgebase.curiobioscience.com/bioinformatics/seeker-local-installation/) analysis on SciNet's Narval HPC system


- **Important:** Since compute nodes are offline, all containers must be downloaded and transferred beforehand

## 1. Environment Setup
While Nextflow can be installed via conda, Narval already has it as a module along with Apptainer/Singularity:

```bash
# Load required modules
module load StdEnv/2020 # this is required to load the correct nextflow module
module load nextflow/23.04.3 # this version is required according to Curio's documentation
module load apptainer  # or singularity on some other systems
```

## 2. Pipeline and Container Setup

### Download Pipeline
```bash
# Download Curio Seeker pipeline
wget https://curioseekerbioinformatics.s3.us-west-1.amazonaws.com/CurioSeeker_v3.0.0/curioseeker-v3.0.0.tar.gz -O - | tar -xzf -
cd curioseeker-*

# Create singularity directory for all required containers
mkdir -p .singularity

# Download main pipeline container
wget https://curioseekerbioinformatics.s3.us-west-1.amazonaws.com/CurioSeeker_v3.0.0/curio-seeker-singularity-v3.0.0.sif -P .singularity/
```

### Container Management
Since compute nodes are offline, all required containers must be downloaded on a node with internet access and placed in the `.singularity` directory. This is critical as the pipeline typically attempts to download containers on-the-fly, which won't work on offline compute nodes.

1. Download the pre-packaged container bundle:
```bash
wget https://curioseekerbioinformatics.s3.us-west-1.amazonaws.com/containerImages/curioseeker-public-singularity-dec-2024.tar.gz -O - | tar -xzf -
```

2. Move all container files into your `.singularity` directory. Required containers include:
   - Main pipeline container: `curio-seeker-singularity-v3.0.0.sif`
   - STAR container (originally from quay.io/biocontainers/star:2.6.1d--h9ee0642_1)
   - Subread container (originally from quay.io/biocontainers/subread:2.0.1--hed695b0_0)
   - Samtools container

3. Set the Singularity cache directory environment variable:
```bash
export NXF_SINGULARITY_CACHEDIR=/path/to/your/.singularity
```

Important Notes:
- Ensure you use only ONE singularity cache directory for all containers
- Make sure all config files point to the same cache directory
- The main Curio Seeker container should be approximately 1.9 GB in size
- Verify that singularity.enabled = true and params.enable_conda = false in your slurm.config

## 3. Input Data Preparation

### Download Reference Data for Rat Genome
```bash
wget https://curioseekerbioinformatics.s3.us-west-1.amazonaws.com/references/Rattus_norvegicus.tar.gz -O - | tar -xzf -
```

### Prepare Sample Sheet
Create `samplesheet.csv` with the following format:
```csv
sample,experiment_date,barcode_file,fastq_1,fastq_2,genome
sample1,yyyy-mm-dd,/path/to/${Tile_ID}_BeadBarcodes.txt,/path/to/R1.fastq.gz,/path/to/R2.fastq.gz,mRatBN7.2
```
Note about Barcode Files:
The barcode_file (e.g., `${Tile_ID}_BeadBarcodes.txt`) must be obtained from:

- Your experimental metadata provided by the lab team running the Curio spatial transcriptomics experiment
- [Curio Bead Barcode Whitelist Retrieval website](https://knowledgebase.curiobioscience.com/bioinformatics/tile-barcode/)

## 4. SLURM Configuration
We have included a version of `slurm.config` used on Narval.

## 5. Running the Pipeline
Make sure the correct modules are loaded
```
export root_output_dir=/home/UNAME/scratch/outputs # this should be on scratch to ensure you have enough space for intermediate files and final results

nextflow run main.nf \
    --input /home/dhoward/scratch/mommybrain/samplesheet_rat_to_finish.csv \
    --outdir ${root_output_dir}/results/ \
    -work-dir ${root_output_dir}/work/ \
    --igenomes_base /home/UNAME/projects/def-shreejoy/UNAME/reference \
    -profile slurm \
    -config /home/UNAME/projects/def-shreejoy/UNAME/curioseeker-v3.0.0/slurm.config
```

## 6. Monitor Progress
Key files to monitor:
- Summary: `${root_output_dir}/results/pipeline_info/execution_report_${time_stamp}.html`
- Detailed log: `.nextflow.log` in launch directory

## Key Output Files
After successful completion, find results in ${root_output_dir}:
```
results/
└── OUTPUT/${SAMPLE_ID}/
    ├── ${SAMPLE_ID}_anndata.h5ad
    ├── ${SAMPLE_ID}_barcodes.tsv
    ├── ${SAMPLE_ID}_cluster_assignment.txt
    ├── ${SAMPLE_ID}_genes.tsv
    ├── ${SAMPLE_ID}_Report.html
    └── ${SAMPLE_ID}_seurat.rds
```

## Troubleshooting Tips
1. Check `.nextflow.log` for detailed error messages
2. Ensure all containers are properly downloaded
3. Verify input file paths in samplesheet.csv are absolute paths
4. Use `-resume` flag to continue from last successful step after fixing errors
