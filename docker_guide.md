# Docker Build and Testing Guide

---

# 1. Build Docker Image on Windows

## 1.1 Enter Project Directory

```powershell
cd SpikeFormerUNet-Docker
```

Project structure:

```text
SpikeFormerUNet-Docker/

├── Dockerfile
├── run.sh
├── requirements.txt

├── weights/

├── inference/
├── model/
├── utilities/

├── config.py
├── inference_main.py
├── inference_single_model.py
├── weighted_ensemble.py
```

---

## 1.2 Create Dockerfile

```dockerfile
FROM pytorch/pytorch:2.7.1-cuda12.6-cudnn9-runtime

WORKDIR /app

COPY . /app

RUN pip install --upgrade pip

RUN pip install -r requirements.txt

RUN chmod +x run.sh

CMD ["bash", "run.sh"]
```

---

## 1.3 Create run.sh

```bash
#!/bin/bash

python inference_main.py
```

---

## 1.4 Build Docker Image

Run the following command in the directory containing the Dockerfile:

```powershell
sudo docker build -t docker.synapse.org/syn75259843/spikeformerunet:3.0 .
```

Verify the image was built successfully:

```powershell
docker images
```

Expected output:

```text
REPOSITORY        TAG
spikeformerunet   latest
```

---

## 1.5 Export Docker Image

Save the Docker image as a tar file:

```powershell
docker save -o spikeformerunet.tar spikeformerunet:latest
```

Generated file:

```text
spikeformerunet.tar
```

---

# 2. Upload Image to Nectar

```powershell
scp spikeformerunet.tar ubuntu@<IP>:/home/ubuntu/docker/
```

Example:

```powershell
scp spikeformerunet.tar ubuntu@130.216.xxx.xxx:/home/ubuntu/docker/
```

---

# 3. Install Docker on Nectar

Update system packages:

```bash
sudo apt update
```

Install Docker:

```bash
sudo apt install -y docker.io
```

Start Docker service:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Verify installation:

```bash
sudo docker --version
```

---

# 4. Install NVIDIA Container Toolkit

Add the NVIDIA repository:

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
| sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
```

```bash
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
| sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
| sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

Install the toolkit:

```bash
sudo apt update
sudo apt install -y nvidia-container-toolkit
```

Configure Docker runtime:

```bash
sudo nvidia-ctk runtime configure --runtime=docker
```

Restart Docker:

```bash
sudo systemctl restart docker
```

---

# 5. Verify GPU Access Inside Docker

Run:

```bash
sudo docker run --rm \
--gpus all \
nvidia/cuda:12.6.3-runtime-ubuntu22.04 \
nvidia-smi
```

If the V100 GPU appears, GPU passthrough is working correctly.

---

# 6. Import Docker Image

Go to the upload directory:

```bash
cd ~/docker
```

Load the image:

```bash
sudo docker load -i spikeformerunet.tar
```

Verify:

```bash
sudo docker images
```

Expected output:

```text
REPOSITORY        TAG
spikeformerunet   latest
```

---

# 7. Prepare Test Data

Create directories:

```bash
mkdir -p ~/test/input
mkdir -p ~/test/output
```

Expected input structure:

```text
~/test/input/

├── BraTS-GoAT-00001/
│   ├── BraTS-GoAT-00001-t1c.nii.gz
│   ├── BraTS-GoAT-00001-t1n.nii.gz
│   ├── BraTS-GoAT-00001-t2f.nii.gz
│   └── BraTS-GoAT-00001-t2w.nii.gz
│
├── BraTS-GoAT-00002/
│   ├── ...
│
└── ...
```

---

# 8. Run Docker Exactly as BraTS Evaluation

Official testing command:

```bash
sudo docker run \
  --rm \
  --network none \
  --gpus=all \
  --volume /home/ubuntu/test/input:/input:ro \
  --volume /home/ubuntu/test/output:/output:rw \
  --memory=48G \
  --shm-size=16G \
  spikeformerunet:latest
```

Parameter explanation:

| Parameter | Description |
|------------|------------|
| `--network none` | Disable internet access |
| `--gpus=all` | Use all available GPUs |
| `/input:ro` | Read-only input directory |
| `/output:rw` | Writable output directory |
| `--memory=48G` | BraTS memory limit |
| `--shm-size=16G` | BraTS shared memory limit |

---


# 9. Running Docker Containers in the Background on HPC

This guide explains how to run a Docker container in the background on an HPC or cloud instance, monitor its progress, and stop it when needed.

---

## Method 1: Docker Detached Mode (Recommended)

Start the container in the background using Docker's detached mode:

```bash
sudo docker run -d \
    --name spikeformerunet_test \
    --gpus=all \
    --network none \
    --volume /home/ubuntu/docker/input:/input:ro \
    --volume /home/ubuntu/docker/output:/output:rw \
    --memory=48G \
    --shm-size=16G \
    docker.synapse.org/syn75259843/spikeformerunet:3.0
```

### Parameter Explanation

| Parameter | Description |
|------------|------------|
| `-d` | Run the container in the background (detached mode). |
| `--name spikeformerunet_test` | Assign a custom container name. |
| `--gpus=all` | Make all available GPUs visible inside the container. |
| `--network none` | Disable network access inside the container. |
| `--volume /home/ubuntu/docker/input:/input:ro` | Mount input directory as read-only. |
| `--volume /home/ubuntu/docker/output:/output:rw` | Mount output directory as read-write. |
| `--memory=48G` | Limit container RAM usage to 48 GB. |
| `--shm-size=16G` | Set shared memory size to 16 GB. |

---

## Monitor Container Status

List running containers:

```bash
docker ps
```

Example:

```text
CONTAINER ID   IMAGE                 STATUS
f626888cc45c   spikeformerunet:2.5   Up 10 minutes
```

---

## View Container Logs

Follow container logs in real time:

```bash
docker logs -f spikeformerunet_test
```

---

## Check GPU Utilization

Monitor GPU usage:

```bash
watch -n 5 nvidia-smi
```

Example:

```text
GPU Name        Memory-Usage
V100-SXM2-16GB  3500MiB / 16160MiB
```

---

## Stop a Running Container

Stop a specific container:

```bash
docker stop spikeformerunet_test
```

Or stop using the container ID:

```bash
docker stop f626888cc45c
```

---

## Remove a Container

After stopping:

```bash
docker rm spikeformerunet_test
```

---

## Stop All Running Containers

```bash
sudo docker stop $(sudo docker ps -q)
```

---

## Remove All Containers

```bash
sudo docker rm $(sudo docker ps -aq)
```

---

## Method 2: Using nohup

You can also run Docker through `nohup` so it survives terminal disconnection.

```bash
nohup sudo docker run \
    --rm \
    --gpus=all \
    --network none \
    --volume /home/ubuntu/docker/input:/input:ro \
    --volume /home/ubuntu/docker/output:/output:rw \
    --memory=48G \
    --shm-size=16G \
    docker.synapse.org/syn75259843/spikeformerunet:3.0 \
    > docker.log 2>&1 &
```

### Parameter Explanation

| Parameter | Description |
|------------|------------|
| `nohup` | Ignore hangup signals when SSH disconnects. |
| `> docker.log` | Redirect standard output to a log file. |
| `2>&1` | Redirect standard error to the same log file. |
| `&` | Run the process in the background. |
| `--rm` | Automatically remove the container after completion. |

---

### Monitor nohup Logs

```bash
tail -f docker.log
```

---

# Memory Monitoring on HPC

## System Memory

```bash
free -h
```

Example:

```text
               total        used        free
Mem:           251Gi       120Gi        80Gi
```

---

## Real-Time Process Monitoring

```bash
htop
```

or

```bash
top
```

Inside `top`, press:

```text
Shift + M
```

to sort processes by memory usage.

---

## Check Your Own Processes

```bash
ps -u $USER -o pid,%mem,rss,vsz,cmd --sort=-rss
```

RSS is the actual physical memory consumption.

---

## Check GPU Memory

```bash
nvidia-smi
```

Real-time monitoring:

```bash
watch -n 1 nvidia-smi
```

---

# Recommended Workflow

Start container:

```bash
docker run -d \
    --name spikeformerunet_test \
    --gpus=all \
    --network none \
    --volume /home/ubuntu/docker/input:/input:ro \
    --volume /home/ubuntu/docker/output:/output:rw \
    --memory=48G \
    --shm-size=16G \
    spikeformerunet:2.5
```

Monitor logs:

```bash
docker logs -f spikeformerunet_test
```

Monitor GPU:

```bash
watch -n 5 nvidia-smi
```

Check output files:

```bash
ls /home/ubuntu/docker/output
```

Stop container if necessary:

```bash
docker stop spikeformerunet_test
```

---

# 10. Docker Cleanup

After testing, you may want to remove unused containers, images, and build cache to free disk space.

---

## Remove a Container

List all containers:

```bash
sudo docker ps -a
```

Remove a stopped container:

```bash
sudo docker rm spikeformerunet_test
```

Or remove by container ID:

```bash
sudo docker rm <container_id>
```

Force remove a running container:

```bash
sudo docker rm -f spikeformerunet_test
```

---

## Remove an Image

List local images:

```bash
sudo docker images
```

Remove an image by name:

```bash
sudo docker rmi spikeformerunet:latest
```

Or remove by image ID:

```bash
sudo docker rmi <image_id>
```

Force removal if the image is referenced by stopped containers:

```bash
sudo docker rmi -f spikeformerunet:latest
```

---

## Remove All Stopped Containers

```bash
sudo docker container prune
```

Skip the confirmation prompt:

```bash
sudo docker container prune -f
```

---

## Remove Dangling Images

These are untagged images left behind after rebuilding.

```bash
sudo docker image prune
```

Skip confirmation:

```bash
sudo docker image prune -f
```

---

## Remove Build Cache

Remove unused build cache:

```bash
sudo docker builder prune
```

Remove all build cache:

```bash
sudo docker builder prune -a
```

Skip confirmation:

```bash
sudo docker builder prune -af
```

---

## Remove Unused Docker Resources

Remove unused containers, networks, dangling images, and build cache:

```bash
sudo docker system prune
```

Remove everything that is not currently used (including unused images):

```bash
sudo docker system prune -a
```

Skip confirmation:

```bash
sudo docker system prune -af
```

> **Note:** `docker system prune -a` removes all unused images, not only dangling images. Images currently used by running containers will not be removed.

---

## Check Docker Disk Usage

View how much disk space Docker is using:

```bash
sudo docker system df
```

Detailed disk usage:

```bash
sudo docker system df -v
```

Example output:

```text
TYPE            TOTAL     ACTIVE    SIZE
Images          4         1         18.2GB
Containers      2         1         320MB
Local Volumes   1         1         5.6GB
Build Cache     3         0         2.4GB
```

---

## Complete Cleanup (Use with Caution)

If you want to completely reset your local Docker environment:

```bash
sudo docker stop $(sudo docker ps -q)
sudo docker rm -f $(sudo docker ps -aq)
sudo docker rmi -f $(sudo docker images -q)
sudo docker builder prune -af
sudo docker system prune -af
```

> **Warning:** This will remove **all** local containers, images, unused networks, and build cache. Only run this if you no longer need any Docker resources on the machine.
