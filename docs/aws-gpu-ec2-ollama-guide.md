# AWS GPU EC2 Guide For Ollama

This guide explains which AWS EC2 instance to use for the backend when Ollama
needs GPU acceleration.

Use this guide when the CPU-only backend feels slow, especially on instances
like `c5.xlarge`.

## Goal

Run the production backend stack on EC2:

```txt
EC2 GPU instance
  Docker Compose
    FastAPI backend
    Ollama

Vercel
  Next.js frontend

Managed Postgres
  Neon, Supabase, RDS, or another Postgres database with pgvector
```

The GPU is mainly for Ollama. The FastAPI backend itself does not need a GPU.

## Why Not c5.xlarge

`c5.xlarge` is a CPU-only instance:

```txt
4 vCPU
8 GiB RAM
No GPU
```

Ollama can run on CPU, but local LLM inference will be slow. Moving Ollama from
Docker to native Ubuntu on the same CPU-only instance usually does not create a
large speedup. The big speedup comes from using an NVIDIA GPU.

## Recommended Instance

For this project, start with:

```txt
g4dn.xlarge
```

It gives:

```txt
4 vCPU
16 GiB RAM
NVIDIA T4 GPU with 16 GB VRAM
125 GB local instance storage
```

This is a good starter instance for:

```txt
llama3.2:3b
nomic-embed-text
small or medium quantized models
```

Better but more expensive options:

```txt
g5.xlarge  - NVIDIA A10G, 24 GB VRAM
g6.xlarge  - NVIDIA L4, 24 GB VRAM
```

Prefer `g5.xlarge` or `g6.xlarge` if you want more headroom for larger models.

## Avoid g4ad For Ollama

Do not choose `g4ad.xlarge` for this project.

The names look similar, but they are different:

```txt
g4dn.xlarge = NVIDIA GPU, good for Ollama/CUDA
g4ad.xlarge = AMD GPU, not the simple path for Ollama on AWS
```

For this project, choose NVIDIA-based instances such as `g4dn`, `g5`, or `g6`.

## EBS Storage Size

Use:

```txt
100 GiB gp3 root volume
```

This means the EBS root volume only.

On `g4dn.xlarge`, AWS may show two volumes in the launch summary:

```txt
100 GiB EBS root volume
+ 125 GB local instance store
= about 225 GB total shown
```

That is normal.

The 125 GB local instance store is included with the instance, but it is not the
same as EBS. Do not rely on it for persistent Ollama models. Store the project,
Docker data, and Ollama model volume on the EBS root disk unless you are okay
with re-downloading models later.

For the default project models, the model storage is small:

```txt
llama3.2:3b        about 2 GB
nomic-embed-text   about 300 MB
```

Practical EBS sizes:

```txt
30 GiB   minimum for only the default small models
50 GiB   safer low-cost option
100 GiB  recommended MVP default
150 GiB+ if testing many or larger models
```

Use `gp3` with the default `3000` IOPS.

## Expected Storage Cost

EBS is usually much cheaper than the GPU instance.

Example gp3 storage prices checked on 19 May 2026:

`gp3` stands for General Purpose 3, the third generation of General Purpose SSD volumes on Amazon EBS.

```txt
Singapore ap-southeast-1: 100 GB gp3 is about $9.60/month
US East us-east-1:        100 GB gp3 is about $8.00/month
```

Check the AWS pricing page for the latest value before committing to a region:

```txt
https://aws.amazon.com/ebs/pricing/
```

## GPU Instance Quota Error

If launch fails with this message:

```txt
You have requested more vCPU capacity than your current vCPU limit of 0 allows
for the instance bucket that the specified instance type belongs to.
```

It means the AWS account currently has no quota for that GPU instance family in
the selected region.

For `g4dn.xlarge`, request quota for:

```txt
Running On-Demand G and VT instances
```

Minimum needed:

```txt
4 vCPUs
```

Recommended request:

```txt
8 vCPUs
```

`8` gives enough quota for one `g4dn.xlarge` with room to relaunch or test
another small GPU instance.

## Request The Quota Increase

In the AWS Console:

1. Open `Service Quotas`.
2. Select `AWS services`.
3. Search for `ec2`.
4. Open `Amazon Elastic Compute Cloud (Amazon EC2)`.
5. Search for `G and VT`.
6. Open `Running On-Demand G and VT instances`.
7. Click `Request increase at account-level`.
8. Enter `8`.
9. Submit the request.

Make sure the selected AWS region is the same region where the EC2 instance will be launched.

Example request reason:

```txt
I need to run one g4dn.xlarge GPU instance for testing an Ollama-based AI
backend using NVIDIA GPU acceleration.
```

Quota request status usually moves through these states:

```txt
Pending
Case Opened
Approved
```

`Pending` means AWS received the quota request and is preparing to review it.
`Case Opened` means AWS created a Support Center case for the quota increase.
At this point, the applied quota may still show `0`, so the GPU instance launch
can still fail. Wait until the request is `Approved` and the applied quota value
updates to the requested value, such as `8`.

If AWS cannot approve the request, the status may change to `Denied` or the
support case may ask for more information. Reply to the support case if AWS asks
for details, then wait for approval before launching the GPU instance again.

## After Launch

After the GPU EC2 instance is running, follow:

```txt
docs/ec2-backend-compose-guide.md
```

When NVIDIA drivers and the container runtime are installed, check GPU access:

```bash
nvidia-smi
```

Then start the production backend stack and pull the models:

```bash
docker compose -f docker-compose.prod.yml up -d --build
docker exec company-ai-ollama ollama pull llama3.2:3b
docker exec company-ai-ollama ollama pull nomic-embed-text
```

Check that Ollama has the models:

```bash
docker exec company-ai-ollama ollama list
```

During chat generation, run:

```bash
nvidia-smi
```

If GPU memory or utilization changes while Ollama is answering, Ollama is using
the GPU.

## Quick Recommendation

For the current project:

```txt
Instance:       g4dn.xlarge
Root volume:    100 GiB gp3 EBS
Quota request:  Running On-Demand G and VT instances = 8 vCPUs
Models:         llama3.2:3b + nomic-embed-text
```
