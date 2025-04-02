# Using ProteinMPNN on AMD GPUs at Pawsey

## Installation

Clone the ProteinMPNN repository into your software directory:
```
cd $MYSOFTWARE
git clone https://github.com/dauparas/ProteinMPNN
```

### Dependencies

ProteinMPNN requires PyTorch, which is available as an optimized system-wide module on Pawsey. To use it:

Check available PyTorch versions:

`module avail pytorch`

Load the appropriate PyTorch module:

`module load pytorch/2.2.0-rocm5.7.3`

The pytorch module version will change over time, so it's important to check the available modules.

Note: No additional conda environment setup is required as all dependencies are handled by the system module.

## Running ProteinMPNN on Pawsey GPUs

SLURM Job Script Template

Create a job script (e.g., run_proteinmpnn.sh) with the following configuration. In particular, note the loading of the pytorch module and adding the correct srun parameters before each python task. This script assumes you’re in the ProteinMPNN directory that you have cloned from https://github.com/dauparas/ProteinMPNN. 
```
#!/bin/bash -l
#SBATCH --partition=gpu
#SBATCH --nodes=1
#SBATCH --gres=gpu:1
#SBATCH --account=${PAWSEY_PROJECT}-gpu
#SBATCH --time=1:00:00
#SBATCH --output=example_1.out

# Load required module
module load pytorch/2.2.0-rocm5.7.3

# Define directories
folder_with_pdbs="${MYSOFTWARE}/ProteinMPNN/inputs/PDB_monomers/pdbs/"
output_dir="${MYSCRATCH}/ProteinMPNN/outputs/example_1_outputs"

# Create output directory if it doesn't exist
if [ ! -d $output_dir ]; then
    mkdir -p $output_dir
fi

# Define path for intermediate files
path_for_parsed_chains=$output_dir"/parsed_pdbs.jsonl"

# Parse PDB chains
srun -N 1 -n 1 -c 8 --gres=gpu:1 --gpus-per-task=1 \
    python helper_scripts/parse_multiple_chains.py \
    --input_path=$folder_with_pdbs \
    --output_path=$path_for_parsed_chains

# Run protein design
srun -N 1 -n 1 -c 8 --gres=gpu:1 --gpus-per-task=1 \
    python protein_mpnn_run.py \
    --jsonl_path $path_for_parsed_chains \
    --out_folder $output_dir \
    --num_seq_per_target 2 \
    --sampling_temp "0.1" \
    --seed 37 \
    --batch_size 1
```

### Important Parameters

`--partition=gpu` Specifies the GPU partition

`--nodes=1` Number of nodes to use

`--gres=gpu:1` Requests one GPU

`--account=${PAWSEY_PROJECT}-gpu` Your project's GPU account

`--time=1:00:00`: Job time limit (adjust as needed)

### Running Tasks

Each Python task in the script uses `srun` with specific GPU parameters:

`-N 1` One node

`-n 1` One task

`-c 8` 8 CPU cores per task

`--gres=gpu:1` One GPU per task

`--gpus-per-task=1` One GPU per task

`--gpu-bind=closest` Optimal GPU-CPU binding

### Submit the Job

1. Submit your job to the SLURM scheduler:

`sbatch run_proteinmpnn.sh`

2. Check job status:

`squeue -u $USER`

## Further Reading

For more details on running GPU workflows on Setonix, refer to https://pawsey.atlassian.net/wiki/spaces/US/pages/51928618 
