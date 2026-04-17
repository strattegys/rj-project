# RJ Project - Local LLM Server Specifications
## Hardware for Self-Hosted Coding + SRE Intelligence

**Date:** 2026-04-16

---

## Two Workloads, Potentially One Server

| Workload | When it runs | What it does | Model size needed |
|----------|-------------|--------------|-------------------|
| **Coding Assistant** | During setup/config, on-demand | Write integration code, generate configs, debug connection issues, build adapters | 30-70B (needs to be genuinely good at code) |
| **SRE Runtime Agent** | 24/7 heartbeat loop | Triage alerts, investigate incidents, correlate signals | 8-14B (Tier 1) + 30-70B (Tier 2, on-demand) |

These can share hardware — the coding assistant is bursty (used during setup), while the SRE agent is steady-state. They won't peak at the same time.

---

## The GPU Reality Check

LLM quality scales with parameter count. Here's what's actually usable for **coding tasks** today:

### Models Worth Running Locally for Code

| Model | Params | VRAM Needed (FP16) | VRAM (Q4 quantized) | Coding Quality |
|-------|--------|-------------------|---------------------|----------------|
| Qwen 2.5 Coder 7B | 7B | 14 GB | 5 GB | Decent for simple tasks |
| DeepSeek-Coder-V2-Lite | 16B | 32 GB | 10 GB | Good |
| Codestral 22B (Mistral) | 22B | 44 GB | 14 GB | Very good |
| Qwen 2.5 Coder 32B | 32B | 64 GB | 20 GB | Excellent — best local coding model at this size |
| DeepSeek-V3/R1 (distilled 70B) | 70B | 140 GB | 40 GB | Near-Claude quality for many tasks |
| Llama 3.1 70B | 70B | 140 GB | 40 GB | Strong general + code |
| Qwen 2.5 72B | 72B | 144 GB | 42 GB | Very strong all-around |

**The sweet spot for a self-hosted coding LLM: Qwen 2.5 Coder 32B (quantized)**
- Genuinely good at writing Python, Go, YAML configs, SQL, Terraform
- Runs on a single 48GB GPU or dual 24GB GPUs
- Fast enough for interactive use (~20-40 tokens/sec on good hardware)
- Punches well above its weight on coding benchmarks

**If budget allows: DeepSeek-R1 70B or Qwen 72B (quantized)**
- Approaches frontier model quality for code
- Needs 2x 24GB GPUs minimum, ideally 2x 48GB
- Slower but significantly smarter for complex reasoning

---

## Server Configurations

### Option A: "Get Started" — Single GPU ($3,000-5,000)

Best for: Proving the concept, small infrastructure (<50 services)

```
CPU:      AMD Ryzen 9 7900X or Intel i7-13700K (12-16 cores)
RAM:      64 GB DDR5
GPU:      1x NVIDIA RTX 4090 (24 GB VRAM)
Storage:  1 TB NVMe SSD (model weights + event store)
OS:       Ubuntu 22.04 LTS

What it runs:
  Coding LLM:    Qwen 2.5 Coder 32B (Q4 quantized) — fits in 24GB
  SRE Tier 1:    Phi-4 14B or Qwen 2.5 7B (Q4) — swap with coding model
  SRE Tier 2:    Same as coding model (Qwen 32B)
  Inference:     Ollama or llama.cpp

Limitations:
  - Can only run ONE model at a time (swap between coding + SRE)
  - Tier 2 investigation blocks Tier 1 triage while running
  - No redundancy — if GPU dies, everything stops
  - 70B models won't fit (even quantized)

Performance:
  - Qwen 32B Q4: ~25-35 tokens/sec
  - Good enough for interactive coding and SRE triage
```

### Option B: "Production Capable" — Dual GPU ($8,000-12,000)

Best for: Running both workloads concurrently, medium infrastructure

```
CPU:      AMD Ryzen 9 7950X or Threadripper 7960X (16-24 cores)
RAM:      128 GB DDR5
GPU:      2x NVIDIA RTX 4090 (48 GB VRAM total)
          OR 1x NVIDIA RTX 6000 Ada (48 GB VRAM)
Storage:  2 TB NVMe SSD
OS:       Ubuntu 22.04 LTS

What it runs:
  Coding LLM:    Qwen 2.5 Coder 32B (FP16 or Q8) on GPU 1
  SRE Tier 1:    Phi-4 14B on GPU 2 (always loaded, fast triage)
  SRE Tier 2:    Qwen 32B on GPU 1 (shares with coding model)
  Inference:     vLLM (better multi-model management)
  Vector DB:     Qdrant (for RAG)
  Event Store:   TimescaleDB

Benefits:
  - Tier 1 SRE runs independently on its own GPU (always ready)
  - Coding model and Tier 2 share GPU 1 (acceptable — not concurrent)
  - 128 GB RAM handles the databases comfortably
  - Can run 70B models with Q4 quantization across both GPUs

Performance:
  - Qwen 32B FP16: ~20-30 tokens/sec
  - Phi-4 14B: ~60-80 tokens/sec (very fast triage)
  - Can serve the heartbeat loop without interruption
```

### Option C: "Enterprise / 70B Models" — Multi-GPU ($15,000-25,000)

Best for: Running the best open models, large infrastructure, multiple users

```
CPU:      AMD Threadripper PRO 7975WX (32 cores)
          OR dual Intel Xeon w5-3435X
RAM:      256 GB DDR5 ECC
GPU:      2x NVIDIA A6000 (96 GB VRAM total)
          OR 4x NVIDIA RTX 4090 (96 GB VRAM total)
          OR 2x NVIDIA L40S (96 GB VRAM total)
Storage:  4 TB NVMe SSD (RAID 1)
Network:  10 GbE (for pulling data from monitoring tools)
OS:       Ubuntu 22.04 LTS
PSU:      1600W+ (4090s are power hungry)

What it runs:
  Coding LLM:    DeepSeek-R1 70B or Qwen 72B (Q8 quantized) — best quality
  SRE Tier 1:    Phi-4 14B (dedicated GPU, always hot)
  SRE Tier 2:    Same 70B model (shared with coding)
  SRE Tier 3:    Fine-tuned 7B specialist (optional, on Tier 1's GPU)
  Inference:     vLLM with model sharding across GPUs
  Vector DB:     Qdrant
  Event Store:   TimescaleDB
  State Store:   Redis

Benefits:
  - 70B models are genuinely close to Claude/GPT-4 for code
  - Enough VRAM to keep multiple models loaded
  - ECC RAM for reliability (this runs 24/7)
  - Can serve multiple concurrent users for the coding assistant

Performance:
  - 70B Q8 across 2x A6000: ~15-25 tokens/sec
  - Fast enough for both interactive use and automated investigation
  - Tier 1 on dedicated GPU: sub-second triage decisions
```

### Option D: "Cloud Private" — GPU Cloud Instance ($/hr)

For orgs that want local LLM but don't want to buy hardware.
Deploy in a private VPC — data stays in your cloud, never hits a third-party AI API.

```
Provider options:
  AWS:        g5.12xlarge (4x A10G, 96 GB VRAM)     ~$5.67/hr
              p4d.24xlarge (8x A100, 320 GB VRAM)    ~$32/hr (overkill)
  GCP:        g2-standard-48 (4x L4, 96 GB VRAM)    ~$5.20/hr
  Azure:      NC48ads_A100 (2x A100, 160 GB VRAM)   ~$8.50/hr
  Lambda:     1x A100 80GB                            ~$1.10/hr
  RunPod:     1x A100 80GB                            ~$1.64/hr

Recommended:
  Lambda or RunPod 2x A100 80GB: ~$2.50-3.50/hr
  - Enough for any open model at full precision
  - Scale to zero when not needed (coding assistant hours only)
  - Keep a smaller always-on instance for SRE runtime

Monthly cost estimate:
  SRE runtime (always-on, 1x A10G):      ~$550/mo
  Coding assistant (on-demand, 8hrs/day): ~$600/mo
  Total:                                   ~$1,150/mo
```

---

## Recommended Stack for Each Option

All options run the same software stack:

```
┌─────────────────────────────────────────────┐
│                SOFTWARE STACK                │
├─────────────────────────────────────────────┤
│                                             │
│  Model Serving:   vLLM or Ollama            │
│                   (OpenAI-compatible API)    │
│                                             │
│  Coding UI:       Open WebUI                │
│                   (ChatGPT-like interface)   │
│                   OR                         │
│                   Continue.dev / Tabby       │
│                   (IDE integration)          │
│                                             │
│  SRE Agent:       Custom Python/Go service  │
│                   (the heartbeat engine)     │
│                                             │
│  RAG:             Qdrant + nomic-embed-text  │
│  State:           Redis + TimescaleDB       │
│  Deployment:      Docker Compose or K8s     │
│                                             │
└─────────────────────────────────────────────┘
```

### Coding assistant integration options:

1. **Open WebUI** — Browser-based ChatGPT clone, connects to Ollama/vLLM, supports RAG, file uploads, conversation history. Free, self-hosted. Best for general coding Q&A and config generation.

2. **Continue.dev** — VS Code / JetBrains extension that points to your local model. Tab completion + chat. Best for developers writing integration code in their IDE.

3. **Tabby** — Self-hosted GitHub Copilot alternative. Tab completion + code review. Good for the autocomplete experience.

4. **Aider** — Terminal-based coding assistant. Points to local model via OpenAI-compatible API. Good for CLI-first workflows. Can directly edit files.

All of these speak the OpenAI API format, and both Ollama and vLLM expose that API — so they're interchangeable.

---

## My Recommendation

**Start with Option A** (single RTX 4090) to prove the concept:
- Buy or repurpose one workstation
- Install Ollama + Qwen 2.5 Coder 32B
- Install Open WebUI for the coding interface
- Build the MVP heartbeat agent against this
- Total investment: ~$3,000-5,000

**Graduate to Option B** (dual GPU) when you need concurrent SRE + coding:
- Add a second 4090 or upgrade to an RTX 6000 Ada
- Move to vLLM for better multi-model management
- This is the "real product" configuration for demos and early customers

**Offer Option D** (cloud private) for customers who don't want hardware:
- Same software stack, deployed in their VPC
- They get the "no data leaves your network" guarantee
- Lower upfront cost, predictable monthly spend

---

## Key Specs Summary

| | Option A | Option B | Option C | Option D (cloud) |
|---|---------|----------|----------|-------------------|
| **GPU VRAM** | 24 GB | 48 GB | 96 GB | 80-160 GB |
| **Best coding model** | Qwen 32B Q4 | Qwen 32B FP16 | 70B Q8 | 70B FP16 |
| **Concurrent models** | 1 | 2 | 3+ | 2-4 |
| **Upfront cost** | $3-5K | $8-12K | $15-25K | $0 |
| **Monthly cost** | ~$50 (power) | ~$100 (power) | ~$200 (power) | $1,000-1,500 |
| **Coding quality** | Good | Very good | Excellent | Excellent |
| **SRE + coding concurrent** | No | Yes | Yes | Yes |
