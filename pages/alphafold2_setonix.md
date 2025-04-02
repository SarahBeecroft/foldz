# Running AlphaFold2 on Setonix

## Overview

This guide explains how to run AlphaFold2 (AF2) on Pawsey's Setonix supercomputer GPU nodes. Please note there is currently a GPU memory limitation that restricts computations to proteins of approximately 3,013 amino acids or less. Attempting to process larger proteins will result in out-of-memory errors.

If you do not have a GPU allocation available to you, this software will use CPUs if no GPUs are available. However, it will take longer to complete. 

## Reference Data Location

AlphaFold2 requires several reference databases. On Setonix, these are located at:

`/scratch/references/alphafold_feb2024/databases/`

The following databases are available:

- UniRef90 (uniref90.fasta)

- MGnify (mgy_clusters_2022_05.fa)

- PDB70

- Small BFD

- PDB mmCIF files

## Job Script Template

Below is a template SLURM script for running AlphaFold2. Save this as run_af2.slurm. 
```bash
#!/bin/bash -l
#SBATCH -A ${PAWSEY_PROJECT}-gpu
#SBATCH --nodes=1
#SBATCH --partition=gpu-highmem
#SBATCH --time=10:00:00
#SBATCH --gres=gpu:2

# Load required module
module load pawseyenv/2023.08
module load singularity/3.11.4-nompi

REF_DIR='/scratch/references/alphafold_feb2024/databases'

# Run AlphaFold2
srun -N 1 -n 1 -c 16 --gres=gpu:2 \
  singularity exec docker://quay.io/pawsey/alphafold2-amd-gpu:rocm6.1.1 \
  python /opt/alphafold/run_alphafold.py \
  --fasta_paths=/path/to/your/sequence.fasta \
  --model_preset=monomer \
  --use_gpu_relax=True \
  --benchmark=False \
  --uniref90_database_path=${REF_DIR}/uniref90/uniref90.fasta \
  --mgnify_database_path=${REF_DIR}/mgnify/mgy_clusters_2022_05.fa \
  --pdb70_database_path=${REF_DIR}/pdb70/pdb70 \
  --data_dir=${REF_DIR} \
  --template_mmcif_dir=${REF_DIR}/pdb_mmcif/mmcif_files \
  --obsolete_pdbs_path=${REF_DIR}/pdb_mmcif/obsolete.dat \
  --small_bfd_database_path=${REF_DIR}/small_bfd/bfd-first_non_consensus_sequences.fasta \
  --output_dir=${MYSCRATCH}/alphafold2/output/${SLURM_JOB_ID} \
  --max_template_date=2023-05-14 \
  --db_preset=reduced_dbs \
  --logtostderr \
  --hhsearch_binary_path=/opt/hh-suite/bin/hhsearch \
  --hhblits_binary_path=/opt/hh-suite/bin/hhblits
```

This example script uses two GCDs on the highmem GPU partition to provide enough CPU memory for the Jackhmmr steps.  

### Important Notes and Limitations

Fasta file input: Please note you must change the template script to point to your FASTA file --fasta_paths=/path/to/your/sequence.fasta

Template date: Please note you should change the --max_template_date to suit your analysis.

Output Directory: Remember to modify the --output_dir path to suit your needs if required

Database Preset: The script uses reduced_dbs preset for faster processing. For higher accuracy, you can change to full_dbs but this will increase runtime.

### Running Your Job

1. Submit your job:

`sbatch run_af2.slurm`

2. Monitor your job:

`squeue -u $USER`

### Output Files

AlphaFold2 will create a directory with the job ID under your specified output directory. This will contain:

- Predicted structures in PDB format

- Confidence scores (pLDDT and PAE)

- Log files

- MSA visualization files

### Support

For issues or questions, please contact the Pawsey Help Desk at help@pawsey.org.au

### Further Reading

For more details on running GPU workflows on Setonix, refer to https://pawsey.atlassian.net/wiki/spaces/US/pages/51928618 
