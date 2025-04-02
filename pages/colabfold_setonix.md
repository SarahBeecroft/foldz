# Running ColabFold on Setonix

## Overview

This guide explains how to run ColabFold batch jobs on Pawsey's Setonix supercomputer. ColabFold is a simplified and accelerated implementation of AlphaFold that can work with pre-computed Multiple Sequence Alignments (MSAs). Please note there is currently a memory limitation that restricts computations to proteins of approximately 3,013 amino acids or less. Attempting to process larger proteins will result in out-of-memory errors.

### Prerequisites

- A Pawsey account with GPU allocation

- Pre-computed MSA file in A3M format

- Basic familiarity with SLURM job submission

## Job Script Template

Below is a template SLURM script for running ColabFold. Save this as run_colabfold.slurm:
```bash
#!/bin/bash 
#SBATCH --job-name=colabbatch
#SBATCH --partition=gpu-highmem
#SBATCH --nodes=1
#SBATCH --gres=gpu:2
#SBATCH --mem=0
#SBATCH --time=01:00:00
#SBATCH --account=${PAWSEY_PROJECT}-gpu

# Load required module
module load pawseyenv/2023.08
module load singularity/3.11.4-nompi

# Set input and output paths
A3M=/path/to/your/msafile.a3m
OUT=$MYSCRATCH/colabfold/${SLURM_JOB_ID}
containerImage=docker://quay.io/pawsey/colabfold:rocm6.0.0

# Environment settings for optimal GPU performance
export TF_FORCE_UNIFIED_MEMORY="1"
export XLA_PYTHON_CLIENT_MEM_FRACTION="4.0"
export XLA_PYTHON_CLIENT_ALLOCATOR="platform"
export TF_FORCE_GPU_ALLOW_GROWTH="true"

# Verify connection to ColabFold API
openssl s_client -connect api.colabfold.com:443 

# Run ColabFold
srun -N 1 -n 1 -c 16 --gres=gpu:2 \
        singularity exec $containerImage \
        colabfold_batch \
        --num-recycle 5 \
        --pair-mode unpaired \
        --model-type alphafold2_multimer_v3 \
        --num-models 3 $A3M $OUT
```

### Key Parameters and Settings

Resource Allocation:

- Uses the gpu partition

- Requests 1 GPU

- Sets unlimited memory (--mem=0)

- Default runtime is 1 hour

- Uses 8 CPU cores

Environment Variables:

```
SINGULARITYENV_XLA_PYTHON_CLIENT_PREALLOCATE=0
SINGULARITYENV_TF_FORCE_UNIFIED_MEMORY="1"
SINGULARITYENV_XLA_PYTHON_CLIENT_MEM_FRACTION="4.0"
SINGULARITYENV_TF_FORCE_GPU_ALLOW_GROWTH="true"
export XLA_PYTHON_CLIENT_ALLOCATOR="platform"
```

These settings optimize GPU memory usage and performance on Setonix's AMD GPUs.

ColabFold Settings:


`--num-recycle 5` Number of prediction refinement cycles

`--pair-mode unpaired` For single chain predictions

`--model-type alphafold2_multimer_v3` Uses the latest multimer model

`--num-models 3` Generates 3 prediction models


### Before Running

- Modify the input path:

  `A3M=/path/to/your/msafile.a3m`

- Optional: Adjust the output directory:

  `OUT=$MYSCRATCH/colabfold/${SLURM_JOB_ID}`

- Replace `${PAWSEY_PROJECT}` with your project code.

### Running Your Job

1. Submit your job:

`sbatch run_colabfold.slurm`

2. Monitor your job:

`squeue -u $USER`

### Output Files

ColabFold will create a directory with the job ID containing:

- Predicted structures in PDB format

- Confidence scores

- Ranking information

- Log files

### Common Issues and Solutions

API Connection: The script checks the connection to the ColabFold API. If this fails, ensure you have internet connectivity from your compute node.

Memory Issues: If you encounter memory errors:

- Try adjusting `XLA_PYTHON_CLIENT_MEM_FRACTION`

- Consider using a shorter runtime with more attempts

- Check if your input MSA is too large

### Further Reading

For more details on running GPU workflows on Setonix, refer to https://pawsey.atlassian.net/wiki/spaces/US/pages/51928618 
