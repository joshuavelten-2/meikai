# OCR throughput and cost per Page: CPU vs GPU

Research for [#2](https://github.com/joshuavelten-2/meikai/issues/2), feeding the OCR compute decision (#8) under the map [#1](https://github.com/joshuavelten-2/meikai/issues/1). Researched 2026-11-26.

**Labels used throughout:**
- **[measured]** means a primary source (or our own run) timed it on named hardware.
- **[estimate]** means we derived it from measured numbers. The method is stated next to it.
- **[priced]** means it comes from AWS's own pricing data on the stated date.

## TL;DR

- Either way, compute per Page is nearly free. On an optimised pipeline, GPU spot is about **$0.01-0.02 per 1000 Pages** and CPU spot is about **$0.09-0.13 per 1000 Pages** [estimate]. On stock mokuro both roughly double to quadruple. A 200-Page Volume costs a fraction of a cent on GPU and about 2-5 cents on CPU.
- Per Page, GPU is about **5-10x cheaper and 5-10x faster**. The catch is cold start: a GPU node needs about 2-5 minutes [estimate] to come up from zero, mostly for the CUDA image pull. Model load itself is only about 2-5 s [measured].
- At low traffic the bill is set by **idle nodes and cold starts, not per-Page compute**. Scale-to-zero matters more than the CPU vs GPU choice.
- **Recommendation:** run the Python OCR worker on **Karpenter GPU spot nodes (g4dn.xlarge first, then g6.xlarge / g5.xlarge)** scaled from zero by KEDA on queue depth. Use comic-text-detector plus manga-ocr with batched inference, and keep the worker device-agnostic so a CPU spot NodePool (c7i/c7a.4xlarge) can be the fallback. Ship V1 with stock manga-ocr. Treat a stronger Apache-2.0 recogniser (Baberu OCR) as a cheap accuracy upgrade.

## 1. The realistic pipeline

mokuro is the reference implementation of "manga page → text boxes with coordinates":

- **Detection:** [comic-text-detector](https://github.com/dmMaze/comic-text-detector), a YOLOv5-based block detector with a UNet/DBNet text mask. It is trained on about 13k anime/comic images (Manga109-s, DCM, synthetic). License **GPL-3.0**.
- **Recognition:** [manga-ocr](https://github.com/kha-white/manga-ocr), a Transformers VisionEncoderDecoder (ViT encoder plus BERT-style decoder). It handles **vertical and horizontal** text and multi-line text in one pass. License **Apache-2.0**. Latest version v0.1.16 (2026-07-19).
- **Glue:** [mokuro](https://github.com/kha-white/mokuro) v0.2.5 (2026-07-20), license GPL-3.0. In [`mokuro/manga_page_ocr.py`](https://github.com/kha-white/mokuro/blob/master/mokuro/manga_page_ocr.py) it runs the detector at `input_size=1024`. It then cuts each detected line into crops 64 px high (long lines are split at a max aspect ratio of 16 for vertical text and 8 for horizontal), rotates vertical crops 90°, and calls manga-ocr **once per crop, unbatched**. The output per block is `box`, `vertical`, `font_size`, `lines_coords` and `lines`, which maps directly onto our Text Box (text plus coordinates).

The license matters for a hosted service. GPL-3.0 has no network clause, unlike AGPL, so running it server-side without distributing it creates no source-release duty. Still, keep GPL code isolated in the OCR worker.

### Model size [measured]

| Artifact | Size | Source |
|---|---|---|
| manga-ocr-base `pytorch_model.bin` (fp32, ~111M params) | 444 MB | [HF API](https://huggingface.co/kha-white/manga-ocr-base) |
| comic-text-detector `comictextdetector.pt` / `.onnx` | 80 MB / 95 MB | [manga-image-translator release beta-0.3](https://github.com/zyddnys/manga-image-translator/releases/tag/beta-0.3) (where mokuro downloads it) |
| CPU-only Python env (torch 2.15 CPU + manga-ocr + transformers 5.24) | 1.5 GB installed | our venv, 2026-11-26 |
| CUDA torch wheels (torch 2.9.1 + cudnn, cublas, nccl, cusparse(lt), cufft, cusolver) | ~3.6 GB **compressed** | PyPI JSON sizes |

A CUDA worker image will be roughly **6-10 GB** uncompressed and a CPU image roughly **2 GB** [estimate, from the wheel sizes above].

## 2. Throughput: Pages/sec CPU vs GPU

### 2a. Measured end-to-end (mokuro, whole Page incl. detection)

These numbers come from [kha-white/mokuro#174](https://github.com/kha-white/mokuro/pull/174) (opened 2026-09-09, still open; also [#173](https://github.com/kha-white/mokuro/pull/173)). The authors timed a 177-Page volume (209 for the M4 Pro row) with a cold OCR cache, taking the best of two runs, in fp32. "Optimised" is their three-PR stack: worker-process pipeline, batched OCR, custom beam search, CPU sharding and lazy mask refinement. Its output is byte-identical to upstream in fp32.

| Machine | Device | Upstream 0.2.5 s/Page | Optimised s/Page | Optimised Pages/s | fp16 s/Page |
|---|---|---|---|---|---|
| RTX 4090 + Threadripper 9960X | CUDA | 0.416 | 0.039 | 25.6 | 0.024 |
| RX 6900 XT + Ryzen 7 5800X | ROCm | 0.718 | 0.118 | 8.5 | 0.091 |
| RX 9070 XT + Ryzen 9 7950X | ROCm | 0.973 | 0.241 | 4.1 | 0.049 |
| Threadripper 9960X (24C) | CPU | 1.241 | 0.332 | 3.0 | - |
| Ryzen 9 7950X (16C) | CPU | 2.164 | 0.667 | 1.5 | - |
| Ryzen 7 5800X (8C) | CPU | 2.625 | 1.267 | 0.8 | - |
| M2 Pro | CPU | 2.351 | 0.609 | 1.6 | - |

Caveats:
- These are the PR authors' numbers on consumer hardware, not cloud instances.
- fp16 is not exact: 0.19% of characters change, on 2.6% of Pages.
- Two upstream-tracker issues show that **stock mokuro is CPU/Python-bound even with a GPU**: [#62](https://github.com/kha-white/mokuro/issues/62) reports about 6% GPU use with 80%+ CPU on an RTX 3080, and [#112](https://github.com/kha-white/mokuro/issues/112) reports multithreaded CPU "comparable to cuda mode". So a GPU only pays off once inference is batched and CPU work is pipelined. Upstream has partial fixes for this; the PRs above go further.

### 2b. Measured recognition only (JMangaBench_Read v1.3.3)

[JMangaBench_Read](https://github.com/KUR-creative/JMangaBench_Read) v1.3.3 (2026-10-27) uses 7,789 boxes from 1,847 Pages (about 4.2 boxes per Page). It was timed on an RTX 2080 Ti in fp16:

| Model | Params | CER % | Per-Page p50 ms (batch 1) | Max boxes/s (batched) | License |
|---|---|---|---|---|---|
| manga-ocr (stock) | 111M | 4.81 | 242 | 42 (b56) | Apache-2.0 |
| Baberu OCR v1.1 | 113M | 3.62 | 278 | 105 (b256) | Apache-2.0 |
| PaddleOCR-VL-For-Manga | 959M | 1.99 | 1024 | 6.6 (b10) | Apache-2.0 |
| manga-ocr-base-CPT-flex | 111M | 1.50 | 43 | 260 (b32) | **CC BY-NC-SA 4.0** |
| manga-ocr-nar-small | 54M | 1.71 | 35.5 | 299 (b48) | **CC BY-NC-SA 4.0** |
| manga-ocr-charspot-nano | 1.7M | 2.14 | 28.3 | 633 (b8) | **CC BY-NC-SA 4.0** |
| Frontier LLM APIs (reference only, they produced the labels) | - | 0.24-0.59 | - | - | API |

Caveats:
- The benchmark authors made the CC BY-NC-SA models and flag that conflict of interest themselves.
- Labels are mostly LLM-agreed: only 1,266 of 7,789 are human-confirmed.
- Sound effects are excluded.

### 2c. Our own CPU run [measured]

We ran stock manga-ocr 0.1.16 (torch 2.15 CPU, transformers 5.24) on a Ryzen 7 6800H laptop under WSL2. The input was the 12 example crops from the manga-ocr repo (whole speech bubbles, larger than mokuro's per-line crops), timed after warm-up.

| Threads | Per crop | Peak RSS |
|---|---|---|
| 1 | 933 ms | 831 MB |
| 4 | 461 ms | 833 MB |
| 8 | 360 ms | 834 MB |

At about 4 boxes per Page this is roughly 1.5-3.7 s of recognition per Page. That is in line with the upstream CPU rows in 2a.

### 2d. Cloud instances [estimate]

| Instance | Accelerator / vCPU | Upstream s/Page | Optimised s/Page | Basis |
|---|---|---|---|---|
| g4dn.xlarge | T4 16 GB, 4 vCPU | 0.6-1.2 | 0.15-0.35 | Upstream is CPU-bound, so the 4090 rig's 0.416 gets worse on 4 older vCPUs. Optimised scales the 4090's 0.039 by the fp32 TFLOPS gap (T4 8.1 vs 4090 82.6) with partial overlap. fp16/tensor cores would help further. |
| g6.xlarge / g5.xlarge | L4 / A10G 24 GB, 4 vCPU | 0.5-0.9 | 0.07-0.15 | About 30 fp32 TFLOPS, roughly 2.7x below a 4090 |
| c7i.4xlarge / c7a.4xlarge | 16 vCPU (8 physical cores on c7i) | 2.2-3.0 | 1.0-1.5 | Close to the measured 5800X (8C, 2.625 → 1.267) and 7950X rows |

So GPU throughput is about **3-15 Pages/s per node** (optimised) and CPU about **0.7-1 Page/s per 16-vCPU node** [estimate]. **A real benchmark on g4dn.xlarge and c7i.4xlarge with a sample Volume should replace this table before #8 is finalised.** It takes an hour or two, using the mokuro#174 fork stack and stock mokuro.

## 3. Memory / VRAM

- **RAM, recogniser only:** about 830 MB RSS on CPU [measured, 2c].
- **RAM, detector plus recogniser plus page buffers:** about 1.5-2.5 GB per worker process [estimate]. The optimised CPU mode runs one model process per L3 domain, so multiply by the shard count.
- **VRAM:** both models are about 525 MB fp32 on disk. With batch 32-128 crops at 224 px plus the detector at 1024 px, expect about 2-4 GB VRAM [estimate; no primary source publishes it]. That fits easily in a T4's 16 GB, so one GPU could serve 2-4 worker pods if GPU sharing (time-slicing) is set up. One pod per GPU is the simple default.

## 4. Cold start / model load

| Phase | Time | Label |
|---|---|---|
| `import manga_ocr` (torch plus transformers import) | 2.6-3.0 s | measured (2c) |
| Load manga-ocr from local HF cache | 1.7-2.3 s | measured (2c) |
| Same, first run including 444 MB download from HF | 10.6 s | measured (2c, home broadband) |
| Detector plus OCR init on CUDA ("Initializing text detector" → "OCR ready") | 4.3-5.6 s | measured, from logs in [mokuro#62](https://github.com/kha-white/mokuro/issues/62) |
| Karpenter launch of a new EC2 node to Ready | ~1-2 min | estimate |
| Pull of a 6-10 GB CUDA image onto a fresh node | ~1-3 min | estimate; mitigate by baking the image into the AMI/EBS snapshot, or with SOCI lazy loading |
| Pull of a ~2 GB CPU image | ~20-40 s | estimate |

Bake both model files into the image (or an EBS snapshot) so they are never downloaded from HF at runtime. Scale-to-zero then costs **about 2-5 min of first-Volume latency on GPU** and **about 1.5-2.5 min on CPU** [estimate]. Model load is a rounding error next to node provisioning.

## 5. Accuracy on vertical Japanese text

- manga-ocr was built for manga and lists "both vertical and horizontal text" among its target cases. mokuro marks each block `vertical` and rotates the crops before recognising them.
- On JMangaBench_Read (mostly vertical manga dialogue), stock manga-ocr scores **4.81% CER** and **78.5% exact match per box**. Baberu OCR v1.1 scores 3.62%; its v1.2 model card claims 3.04% ([HF card](https://huggingface.co/genshiai-daichi/baberu-ocr), self-reported). Baberu OCR is Apache-2.0, has 115M params, and ships ONNX builds of 122-243 MB, so it is a **drop-in upgrade candidate**. On Manga109-s held-out sound effects its card reports lCER 0.117 against manga-ocr's 0.631. SFX are a weak spot for manga-ocr.
- The best-scoring small models (CPT-flex 1.5%, nar-small 1.7%) are **CC BY-NC-SA 4.0**. They are usable only if meikai stays non-commercial, so check with the owner before adopting them.
- General VLMs are a poor fit here. [Sasagawa et al., arXiv:2511.15059 (2025)](https://arxiv.org/abs/2511.15059) find that existing multimodal LLMs "perform worse on vertically written Japanese text than on horizontally written". PaddleOCR-VL-For-Manga reaches 1.99% CER but is about 9x larger and about 6x slower than manga-ocr even batched (6.6 boxes/s), which would push GPU cost per Page up roughly 6-40x. Frontier LLM APIs are the most accurate (0.24-0.59% CER) but cost per call; they are a post-V1 option for re-OCR, not the V1 path.

## 6. Price per 1000 Pages

### Hourly prices, us-east-1, Linux

- **On-demand [priced]:** AWS price feed `ec2-ondemand-without-sec-sel`, published 2026-11-24.
- **Spot [estimate]:** on-demand × (1 − savings%) from the [AWS Spot Instance Advisor data](https://aws.amazon.com/ec2/spot/instance-advisor/) (`spot-advisor-data.json`, last modified 2026-03-20). The public spot price page feed (`spot.js`) has not been updated since 2024-05 and was **not** used.
- Check live spot prices in the console or with `aws ec2 describe-spot-price-history` before committing.

| Instance | On-demand $/h | Spot savings | Spot $/h (est) | Interruption freq (advisor) |
|---|---|---|---|---|
| g4dn.xlarge (T4) | 0.526 | 60% | ~0.21 | 10-15% |
| g6.xlarge (L4) | 0.805 | 49% | ~0.41 | >20% |
| g5.xlarge (A10G) | 1.006 | 56% | ~0.44 | >20% |
| c7i.4xlarge (16 vCPU) | 0.714 | 57% | ~0.31 | >20% |
| c7a.4xlarge (16 vCPU) | 0.821 | 55% | ~0.37 | 5-10% |
| c7g.4xlarge (Graviton, 16 vCPU) | 0.580 | 54% | ~0.27 | <5% (ARM PyTorch speed not measured) |

us-east-2 is often cheaper on spot: g4dn.xlarge saves 67% and g6.xlarge 67%.

### $/1000 Pages [estimate]

The formula is $/h × s/Page ÷ 3.6, using the s/Page ranges from 2d. Compute only: no idle time, cold start, storage or egress.

| Instance | Stock mokuro, spot | Optimised, spot | Optimised, on-demand |
|---|---|---|---|
| g4dn.xlarge | $0.035-0.07 | **$0.009-0.02** | $0.02-0.05 |
| g6.xlarge | $0.06-0.10 | **$0.008-0.017** | $0.016-0.034 |
| g5.xlarge | $0.06-0.11 | $0.009-0.018 | $0.02-0.04 |
| c7i.4xlarge | $0.19-0.26 | **$0.09-0.13** | $0.20-0.30 |

What this means in practice:
- **One 200-Page Volume:** about $0.002-0.004 on GPU spot (optimised) against about $0.02-0.05 on CPU spot.
- **Latency per Volume on one pod:** about 0.5-1 min (GPU, optimised) against about 3-5 min (CPU, optimised) or 7-10 min (CPU, stock). CPU latency can be cut by fanning Pages out across pods.
- **200k Pages/month** (about 1000 Volumes): about **$2-4/month** on GPU spot against **$18-26/month** on CPU spot.
- **Idle cost is the real trap.** One always-on node costs about $150/month (g4dn.xlarge spot) or about $225/month (c7i.4xlarge spot). Scale-to-zero (KEDA plus Karpenter consolidation) is mandatory at this scale.

## 7. Recommendation (input to #8)

1. **GPU spot, scale-to-zero.**
   - Karpenter NodePool, spot first: g4dn.xlarge, g6.xlarge, g5.xlarge (several types for spot availability). It falls back to on-demand g4dn.
   - KEDA ScaledObject on queue depth (SQS), `minReplicaCount: 0`. Karpenter `consolidateAfter` is short, so the node drains about 5-10 min after the queue empties.
   - Use the EKS GPU AMI (AL2023 NVIDIA or Bottlerocket NVIDIA) and the NVIDIA device plugin.
2. **Keep the worker device-agnostic.** Use the same container logic on `cuda` or `cpu`, with a second CPU spot NodePool (c7a/c7i.4xlarge) at lower weight, so a GPU spot shortage degrades latency instead of stopping OCR. At these prices CPU is a perfectly acceptable fallback.
3. **Batch inference.** Do not copy stock mokuro's one-crop-at-a-time loop. Batch crops across Pages (as in manga-ocr#108 / mokuro#174) and run CPU pre/post-processing in worker processes. That is where the 4-10x comes from. Process a whole Volume, or a chunk of Pages, per job so model load and the GPU are amortised. Keep fp32 by default.
4. **Make jobs idempotent per Page** (results written per Page to S3/DB). Spot interruptions (10-20%+) then only cost a retry of in-flight Pages.
5. **Bake model weights into the image or the node's EBS snapshot.** Consider SOCI/image prefetch to cut the 2-5 min GPU cold start.
6. **Accuracy path:**
   - V1 ships comic-text-detector plus stock manga-ocr: Apache-2.0 recogniser, about 4.8% CER on vertical manga dialogue.
   - Evaluate Baberu OCR (Apache-2.0, about 3-3.6% CER, same size class) as a swap behind the same recogniser interface.
   - The CC BY-NC-SA models need an owner decision on commercial use.
   - Keep the Text Box schema engine-agnostic (store the engine plus version per Page) so Volumes can be re-OCR'd later.
7. **Before finalising #8,** run the one-hour benchmark (stock plus optimised stacks, on g4dn.xlarge and c7i.4xlarge, using one real Volume) to replace the [estimate] rows in 2d and §6.

## Sources

- mokuro: https://github.com/kha-white/mokuro, `mokuro/manga_page_ocr.py`, `pyproject.toml`, releases v0.2.5; issues #62, #97, #112; PRs #173, #174
- manga-ocr: https://github.com/kha-white/manga-ocr, release v0.1.16; PR #108; HF model https://huggingface.co/kha-white/manga-ocr-base
- comic-text-detector: https://github.com/dmMaze/comic-text-detector; weights in https://github.com/zyddnys/manga-image-translator/releases/tag/beta-0.3
- JMangaBench_Read v1.3.3: https://github.com/KUR-creative/JMangaBench_Read
- Baberu OCR: https://huggingface.co/genshiai-daichi/baberu-ocr
- Licenses of the muscgab models (CC BY-NC-SA 4.0) and PaddleOCR-VL-For-Manga (Apache-2.0): Hugging Face model API tags
- Sasagawa, Kurita, Kawahara, "Evaluating Multimodal Large Language Models on Vertically Written Japanese Text", arXiv:2511.15059
- AWS on-demand prices: `https://b0.p.awsstatic.com/pricing/2.0/meteredUnitMaps/ec2/USD/current/ec2-ondemand-without-sec-sel/US East (N. Virginia)/Linux/index.json` (published 2026-11-24)
- AWS Spot Instance Advisor data: https://spot-bid-advisor.s3.amazonaws.com/spot-advisor-data.json (last modified 2026-03-20)
- PyTorch / NVIDIA wheel sizes: PyPI JSON API (`torch` 2.9.1, `nvidia-*-cu12`)
- Our own CPU measurement: manga-ocr 0.1.16, torch 2.15.0+cpu, transformers 5.24.1, Ryzen 7 6800H (8C/16T), WSL2, 2026-11-26
