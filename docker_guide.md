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
docker build -t spikeformerunet:latest .
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

# 9. Check Outputs

View outputs:

```bash
ls -lh ~/test/output
```

Expected output:

```text
BraTS-GoAT-00001.nii.gz
BraTS-GoAT-00002.nii.gz
BraTS-GoAT-00003.nii.gz
...
```

Correct output structure:

```text
/output/
├── BraTS-GoAT-00001.nii.gz
├── BraTS-GoAT-00002.nii.gz
└── ...
```

Incorrect output structure:

```text
/output/
└── predictions/
    ├── BraTS-GoAT-00001.nii.gz
```

BraTS requires a flat output directory.

---

# 10. Final Submission Checklist

Before submission, confirm:

- [ ] All inputs are read from `/input`
- [ ] All final outputs are written to `/output`
- [ ] No `/hpc/...` paths remain
- [ ] No hard-coded dataset locations remain
- [ ] No `CUDA_VISIBLE_DEVICES`
- [ ] No `os.chdir(...)`
- [ ] Temporary files are stored in `/tmp`
- [ ] Docker runs successfully with `--network none`
- [ ] Output directory is flat
- [ ] One `.nii.gz` file is generated for each case

If all items are checked, the Docker container is ready for BraTS submission.