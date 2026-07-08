# NeSI GPU Quick Start Guide

## 1. Log in to the Cluster

```bash
ssh mahuika
```

---

## 2. Request an Interactive GPU Session for 15 minutes

### L4 GPU

```bash
srun \
  --account=uoa04825 \
  --partition=milan \
  --gres=gpu:L4:1 \
  --cpus-per-task=4 \
  --mem=16G \
  --time=00:15:00 \
  --pty bash
```

### A100 GPU

```bash
srun \
  --account=uoa04825 \
  --partition=milan \
  --gres=gpu:A100:1 \
  --cpus-per-task=8 \
  --mem=32G \
  --time=00:15:00 \
  --pty bash
```

After entering the compute node:

```bash
hostname
```

Check available GPUs:

```bash
nvidia-smi
```

---

## 3. Load CUDA

List available CUDA versions:

```bash
module spider CUDA
```

Load a CUDA module:

```bash
module load CUDA/12.6.0
```

Verify CUDA:

```bash
nvcc --version
```

Check GPU status:

```bash
nvidia-smi
```

---

## 4. Activate a Conda Environment

Load Miniconda:

```bash
module load Miniconda3
```

List available environments:

```bash
conda env list
```

Activate an environment:

```bash
conda activate jelly126
```

Verify Python:

```bash
which python
python --version
```

Verify PyTorch:

```bash
python -c "import torch; print(torch.__version__)"
```

Check CUDA availability:

```bash
python -c "import torch; print(torch.cuda.is_available())"
```

Check the number of GPUs:

```bash
python -c "import torch; print(torch.cuda.device_count())"
```

Check GPU name:

```bash
python -c "import torch; print(torch.cuda.get_device_name(0))"
```

---

## 5. Test Code

Run a Python script:

```bash
python train.py
```

If the script runs successfully, you are ready to submit a batch job.

---

## 6. Create a Slurm Job Script

Example: `train.slurm`

```bash
#!/bin/bash
#SBATCH --job-name=swin_train
#SBATCH --account=uoa04825
#SBATCH --partition=milan
#SBATCH --gres=gpu:L4:1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --time=24:00:00

#SBATCH --output=logs/%x_%j.out
#SBATCH --error=logs/%x_%j.err

module purge
module load Miniconda3

conda activate jelly126

cd $SLURM_SUBMIT_DIR

python main_nospike.py
```

Create a log directory:

```bash
mkdir -p logs
```

---

## 7. Submit a Job

Submit the job:

```bash
sbatch train.slurm
```

Example output:

```text
Submitted batch job 7539011
```

The number returned is the Job ID.

---

## 8. Monitor Job Status

View all jobs:

```bash
squeue -u $USER
```

View a specific job:

```bash
squeue -j 7539011
```

View detailed job information:

```bash
scontrol show job 7539011
```

Check the estimated start time:

```bash
squeue --start -j 7539011
```

---

## 9. View Job Logs

Monitor output in real time:

```bash
tail -f logs/swin_train_7539011.out
```

View the complete output log:

```bash
less logs/swin_train_7539011.out
```

View the error log:

```bash
less logs/swin_train_7539011.err
```

---

## 10. Check Resource Usage

Check GPU utilization:

```bash
nvidia-smi
```

Monitor CPU and memory usage:

```bash
htop
```

After the job finishes:

```bash
sacct -j 7539011
```

View efficiency statistics:

```bash
seff 7539011
```

---

## 11. Cancel a Job

Cancel a specific job:

```bash
scancel 7539011
```

Cancel all your jobs:

```bash
scancel -u $USER
```

---

## 12. Common Issues

### GPU Type Not Specified

Error:

```text
Our job filter requires that you specify a GPU type
```

Incorrect:

```bash
#SBATCH --gres=gpu:1
```

Correct:

```bash
#SBATCH --gres=gpu:A100:1
```

---


### Check Why a Job Is Pending

Inspect the job:

```bash
scontrol show job <jobid>
```



```text
JobState=PENDING
Reason=Priority
```

Common reasons:

| Reason | Meaning |
|----------|----------|
| Priority | Waiting for scheduler priority |
| Resources | Requested resources are currently unavailable |
| Dependency | Waiting for another job to finish |
| QOSMaxJobsPerUser | Maximum number of jobs for the user has been reached |
| PartitionTimeLimit | Requested runtime exceeds partition limits |

---

## 13. Useful Commands

| Task | Command |
|--------|--------|
| Request interactive GPU session | `srun --account=uoa04825 --partition=milan --gres=gpu:L4:1 --pty bash` |
| Submit a batch job | `sbatch train.slurm` |
| View all jobs | `squeue -u $USER` |
| View a specific job | `squeue -j JOBID` |
| View job details | `scontrol show job JOBID` |
| View job history | `sacct -j JOBID` |
| Check job efficiency | `seff JOBID` |
| Cancel a job | `scancel JOBID` |
| Check GPUs | `nvidia-smi` |
| Check CUDA | `nvcc --version` |
| Check Python path | `which python` |
| Check PyTorch | `python -c "import torch"` |