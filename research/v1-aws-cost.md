# Projected monthly AWS cost for V1 and default storage quota

Research for issue #4 (map: #1). Researched 2026-11-26. Region: **us-east-1**. All prices are on-demand list prices in USD, before tax, credits, Savings Plans or Reserved Instances.

## Sources (primary, date-stamped)

| Short name | Source | Retrieved |
|---|---|---|
| **PL** | AWS Price List bulk API, `https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/<service>/current/us-east-1/index.json` for AmazonEKS, AmazonEC2, AmazonRDS, AmazonS3, AWSELB, AmazonVPC, AmazonCognito, AWSQueueService, AmazonCloudWatch, AmazonECR, AWSSecretsManager, awskms, AmazonRoute53. CloudFront uses the global `AmazonCloudFront/current/index.json`, and the flat-rate plans use `CloudFrontPlans/current/index.json` (version `20260911124622`). Offer index `publicationDate` 2026-11-26T17:52:02Z | 2026-11-26 |
| **EC2-OD** | The EC2 pricing page's on-demand JSON: `b0.p.awsstatic.com/pricing/2.0/meteredUnitMaps/ec2/USD/current/ec2-ondemand-without-sec-sel/US East (N. Virginia)/Linux/index.json`, published 2026-11-24T23:14:17Z | 2026-11-26 |
| **SPOT** | The EC2 Spot pricing page feed, `website.spot.ec2.aws.a2z.com/spot.js`, us-east-1 Linux. This is a **point-in-time** price, and spot prices change | 2026-11-26 |
| **CF-FR** | CloudFront Developer Guide, "CloudFront flat-rate pricing plans" (`docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/flat-rate-pricing-plan.html`) | 2026-11-26 |
| **CF-P** | `aws.amazon.com/cloudfront/pricing/` | 2026-11-26 |
| **COG** | `aws.amazon.com/cognito/pricing/` | 2026-11-26 |
| **reCBZ** | reCBZ 0.7.0 README on PyPI, real-world size of Chainsaw Man vol. 1 | 2026-11-26 |
| **WebP** | Google, "An image format for the Web" (`developers.google.com/speed/webp`): "WebP lossless images are 26% smaller in size compared to PNGs" | 2026-11-26 |

## TL;DR

| | ~100 Readers | ~10k Readers |
|---|---|---|
| **Total, as specced (S3 Standard, CloudFront pay-as-you-go)** | **≈ $220/mo** | **≈ $2,040/mo** |
| **Total with cheap mitigations** (S3 Intelligent-Tiering; CloudFront Business flat-rate at 10k) | ≈ $215/mo | **≈ $1,090/mo** |
| Per Reader | ≈ $2.20 | ≈ $0.11–0.20 |
| Dominant drivers | **Fixed platform costs**: EKS control plane $73, nodes $52, NAT gateway $33, RDS $26, ALB $25. Usage costs are under $10 | **S3 storage** ($920 Standard / ≈$400 IT), **CloudFront egress** ($630 PAYG / $200 flat), RDS Multi-AZ $140, nodes $137 |

- **The hidden trap is NAT.** If Ingest's S3 reads and writes go through a NAT gateway, they cost **$0.045/GB**. At 10k Readers uploading ~1 GB/month each, that is **≈ $920/month**, as much as all of storage. An **S3 gateway VPC endpoint is free** and removes it. Make it mandatory in the VPC module.
- **CloudFront flat-rate plans** (new since late 2025) change the egress picture. Business ($200/mo) includes 50 TB and 125M requests, with **no overage charges**. At 10k Readers (≈8.4 TB, ≈13M requests) that replaces ≈$630 of pay-as-you-go egress. At 100 Readers, pay-as-you-go is **$0** inside CloudFront's always-free 1 TB and 10M requests.
- **Recommended default quota: 10 GB per Reader** (≈40 Volumes at the blended 0.25 GB per Volume: ≈65 JPG or ≈20 PNG Volumes). A Reader at the full quota costs **≈ $0.23/month** in S3 Standard (**≈ $0.10** with Intelligent-Tiering). A worst-case heavy Reader at the full quota costs **≈ $0.60/month** all-in on pay-as-you-go CloudFront, or ≈ $0.25 on a flat-rate plan.

## 1. Unit prices (us-east-1, 2026-11-26)

| Item | Price | Source |
|---|---|---|
| EKS cluster, standard support | $0.10/h ($73.00/mo). Extended support is $0.60/h total, so **stay on supported Kubernetes versions** | PL AmazonEKS `USE1-AmazonEKS-Hours:perCluster`, `:extendedSupport` |
| EC2 t4g.small / t4g.medium / t4g.large | $0.0168 / $0.0336 / $0.0672 per h | EC2-OD |
| EC2 m7g.large / c7g.large / m8g.large | $0.0816 / $0.0725 / $0.08976 per h | EC2-OD |
| EC2 **spot** t4g.medium / m7g.large / c7g.large / m6g.large | $0.0144 / $0.0367 / $0.0341 / $0.0331 per h (point in time) | SPOT |
| EBS gp3 | $0.08/GB-mo | PL AmazonEC2 |
| NAT gateway | **$0.045/h + $0.045/GB processed** | PL AmazonEC2 |
| VPC interface endpoint | $0.01/h per AZ + $0.01/GB. S3/DynamoDB **gateway** endpoints have no charge (they don't appear in the price list) | PL AmazonVPC `USE1-VpcEndpoint-Hours`, `-Bytes` |
| Public IPv4 address (in use or idle) | $0.005/h ($3.65/mo) | PL AmazonVPC `USE1-PublicIPv4:InUseAddress` |
| ALB | $0.0225/h + $0.008/LCU-h | PL AWSELB |
| RDS PostgreSQL db.t4g.micro / small / medium, Single-AZ | $0.016 / $0.032 / $0.065 per h | PL AmazonRDS |
| RDS PostgreSQL db.t4g.small / medium, Multi-AZ | $0.065 / $0.129 per h | PL AmazonRDS |
| RDS PostgreSQL db.m7g.large Single / Multi-AZ | $0.168 / $0.337 per h | PL AmazonRDS |
| RDS gp3 storage Single / Multi-AZ | $0.115 / $0.23 per GB-mo | PL AmazonRDS `RDS:GP3-Storage`, `RDS:Multi-AZ-GP3-Storage` |
| RDS backup storage beyond the free allocation | $0.095/GB-mo | PL AmazonRDS `RDS:ChargedBackupUsage` |
| S3 Standard | $0.023/GB-mo for the first 50 TB, then $0.022 | PL AmazonS3 `TimedStorage-ByteHrs` |
| S3 Intelligent-Tiering, Frequent / Infrequent / Archive Instant Access | $0.023 / $0.0125 / $0.004 per GB-mo, + $0.0025 per 1,000 monitored objects per month | PL AmazonS3 `TimedStorage-INT-*`, `Monitoring-Automation-INT` |
| S3 PUT/COPY/POST/LIST · GET | $0.005 per 1,000 · $0.0004 per 1,000 | PL AmazonS3 `Requests-Tier1`, `Requests-Tier2` |
| CloudFront pay-as-you-go, US egress | First 1 TB/mo **free**, then $0.085/GB for the first 10 TB, $0.080 for the next 40 TB | PL AmazonCloudFront `Global-DataTransfer-Out-Bytes`, `US-DataTransfer-Out-Bytes` |
| CloudFront pay-as-you-go, requests | First 10M/mo **free**, then $0.0100 per 10k HTTPS (US) | PL AmazonCloudFront `Global-Requests-Tier1`, `US-Requests-Tier2-HTTPS` |
| CloudFront origin fetch from S3 / ALB | No charge for AWS-origin → CloudFront transfer | CF-FR ("continues to be free") |
| CloudFront **flat-rate plans** (per distribution) | Free $0 (1M req, 100 GB) · **Pro $15** (10M req, 50 TB) · **Business $200** (125M req, 50 TB) · Premium $1,000+ (500M req, 50 TB, configurable up to 6B/600 TB). No overage charges | PL CloudFrontPlans `terms.FlatRate`, CF-P, CF-FR |
| Cognito Essentials (default tier) / Lite | **10,000 MAU/mo free on both, indefinitely**. Above that, Essentials $0.015/MAU and Lite $0.0055/MAU (first 90k). Plus has no free tier ($0.02) | PL AmazonCognito, COG |
| SQS standard | First 1M requests/mo free, then $0.40/M | PL AWSQueueService |
| CloudWatch Logs | $0.50/GB ingested (Standard), $0.25/GB (Infrequent Access), $0.03/GB-mo stored. Metrics $0.30/metric-mo, alarms $0.10 | PL AmazonCloudWatch |
| Secrets Manager / KMS CMK / ECR | $0.40/secret-mo / $1/key-mo + $0.03 per 10k req / $0.10/GB-mo | PL |

## 2. Usage assumptions

Every number here is an assumption. The model in §3 is linear in most of them, so it's easy to re-scale.

| Assumption | Value | Why |
|---|---|---|
| Pages per Volume | 200 | Given in the ticket. A typical tankōbon is 180–220 pages |
| Bytes per Page, **JPG** | **0.75 MB** | reCBZ reports that Chainsaw Man vol. 1, a retail digital volume, is **180 MB**. At ~190–200 pages that's ≈0.9 MB/page. Scanned raws used with mokuro are often lower resolution (≈1200–1600 px tall, q85–90), at ≈0.3–0.8 MB. Use 0.75 MB as a middle estimate |
| Bytes per Page, **PNG** | **2.5 MB** | Lossless greyscale with screentone compresses poorly. Informal reports put a ≈1600×2400 PNG page at 2–4 MB, several times its JPG. No primary measurement was found, so **measure real uploads during the beta** and replace this number |
| Mix | 70% JPG / 30% PNG → **1.275 MB/Page, ≈ 0.25 GB per Volume** | Most circulating raws are JPG. PNG shows up for some digital rips |
| Stored per Reader (average) | **4 GB** (≈16 Volumes) | Assumed. Most Readers keep what they're currently reading plus a backlog. The quota (§5) caps the tail |
| New uploads per Reader per month | 1 GB (≈4 Volumes) | Assumed: a Reader roughly matches what they read |
| Pages read per Reader per month | **600** (≈3 Volumes) | A learner reading ~20 Pages a day. Some Readers read far more, and many registered Readers are idle in a given month, which balances out |
| Egress multiplier | **1.1×** | Images are immutable and cached for one year (#7), so a re-read on the same browser costs nothing. The +10% covers prefetch overshoot (the reader prefetches 3 Pages), cache eviction and a second device. Libraries are private, so **there is no cross-Reader CloudFront cache hit**: every first view of a Page is an edge miss and an S3 GET |
| Requests per Reader per month | ≈1,300 = 660 image requests + 240 API calls (Text Box fetch per Volume, debounced progress saves, status polls) + 400 for SPA loads | Assumed |
| SPA and API egress per Reader | 20 MB/mo | Text Box JSON is about 100 KB per Volume, and the SPA bundle is cached |
| Text Box storage in RDS | ≈5 KB per Page (≈10 Text Boxes × ~500 B per row including JSONB lines, row overhead and indexes) | Based on the #5 schema |
| Cognito MAU | ≤ 10,000 at both scales | All 10k Readers active in the same month is the worst case, and it still fits the free tier |
| Logs | 3 GB/mo (small), 40 GB/mo (medium) | Structured app logs only. EKS control-plane audit logs **off** (they can be gigabytes a day), and no Container Insights |
| Ingest CPU | ≈20 s per Volume (decode for size checks, cover thumbnail, PUTs) | 40k Volumes/mo × 20 s ≈ 220 vCPU-hours, ≈0.3 vCPU on average. Negligible |

## 3. Monthly cost model

Generated by a short script on the unit prices in §1 and the assumptions in §2. 730 h/month.

| Line item | ~100 Readers | ~10k Readers | Notes |
|---|---:|---:|---|
| EKS control plane | 73.00 | 73.00 | Fixed |
| EC2 nodes + 20 GB gp3 root each | 52.26 | 137.43 | Small: managed node group, 2 × t4g.medium on-demand (holds Karpenter, CoreDNS, the LB controller, Library and Ingest, across 2 AZs). Medium: the same 2 plus 3 × m7g.large **spot** via Karpenter |
| RDS PostgreSQL | 25.66 | 140.17 | Small: db.t4g.small Single-AZ + 20 GB. Medium: db.t4g.medium **Multi-AZ** + 200 GB (≈31M Pages × 5 KB ≈ 155 GB of Text Boxes). Backups within the free allocation |
| S3 storage, **Standard** | 9.20 | 920.00 | 0.4 TB / 40 TB |
| *S3 storage, Intelligent-Tiering (estimate)* | *4.04* | *404.00* | *Steady-state blend: 15% Frequent, 15% IA, 70% Archive Instant Access + monitoring (≈790 Pages per GB). See §4* |
| S3 requests | 0.49 | 49.22 | Mostly Ingest PUTs: 1 PUT per Page, ≈8M/mo at 10k. GETs are cheap |
| CloudFront, pay-as-you-go | **0.00** | **631.47** | 84 GB and 0.13M requests (free tier) / 8.4 TB and 13M requests |
| *CloudFront, Business flat-rate* | — | *200.00* | *Covers 8.4 TB and 13M requests (allowance 50 TB / 125M). Includes WAF, Route 53 DNS and an S3 credit (§4)* |
| ALB (hour + LCU + 2 public IPv4) | 24.60 | 29.57 | ≈0.15 / 1 LCU on average |
| NAT gateway (1, with an **S3 gateway endpoint**) | 32.94 | 34.20 | Only ECR pulls, SQS, STS, Cognito JWKS and CloudWatch traffic |
| *NAT data **without** an S3 gateway endpoint* | *+9* | ***+922*** | *Ingest reads ≈1 GB of archive and writes ≈1 GB of Pages per Reader-month, all at $0.045/GB* |
| Cognito (Essentials) | 0.00 | 0.00 | ≤ 10k MAU free. Every MAU above that is $0.015 (Lite $0.0055) |
| SQS | 0.00 | 0.16 | Dominated by empty 20 s long polls (≈130k per poller per month) |
| CloudWatch | 1.09 | 19.70 | Logs + 10 alarms |
| Misc (Route 53 zone, 3 secrets, ECR, 1 KMS key) | 2.90 | 2.90 | |
| **Total as specced** (Standard + PAYG) | **≈ 222** | **≈ 2,038** | |
| **Total with mitigations** (IT, + Business flat-rate at 10k) | **≈ 217** | **≈ 1,090** | |

### What dominates

- **~100 Readers: the fixed floor is ≈ 95% of the bill.** EKS ($73) + nodes ($52) + NAT ($33) + RDS ($26) + ALB ($25) ≈ $209. Storage, egress and requests together are under $10. Usage barely moves the bill: 100 Readers or 500 cost about the same.
- **~10k Readers: data dominates.** S3 storage (45% as specced) and CloudFront egress (31%) together make up about three quarters of the bill. Compute and DB are ≈ $350 combined, which is fine for 10k users.
- **Per Reader**, a stored GB costs $0.023/mo and a read GB costs $0.085 (PAYG, above the free 1 TB). Reads are bounded by human reading speed, while storage grows monotonically unless it's capped. That is why the quota matters for storage more than for egress.

## 4. Cheap mitigations, by impact

1. **S3 gateway VPC endpoint (free, mandatory).** It saves up to ≈ $920/mo at 10k Readers and keeps Ingest's S3 traffic off the NAT. Also route `uploads` and `content` traffic through it. With S3 off the NAT, the remaining NAT traffic is small. **Don't add interface endpoints** for ECR (api + dkr), SQS, STS and Logs at small scale: 5 endpoints × 2 AZs × $7.30 ≈ $73/mo, more than the NAT they'd replace. Revisit if NAT data exceeds ≈ 1.5 TB/mo.
2. **CloudFront flat-rate plan once egress passes ≈ 1 TB/mo.** Below ~1,200 Readers, pay-as-you-go is free. **Pro ($15)** covers 50 TB but only 10M requests, which is ≈ 7–8k Readers at ~1,300 requests each. **Business ($200)** covers 125M requests. The allowances are "not hard limits": AWS doesn't bill overages, but it may degrade delivery after sustained excess (CF-FR). Caveats to check before switching (CF-FR):
   - Each plan covers **one distribution** with one apex domain. meikai already has exactly one.
   - A WAF web ACL is **required** (it's included in the plan).
   - **Signed URLs** are listed as supported on all tiers. **Signed cookies** use the same trusted-key-group mechanism but aren't named, so **verify in a test distribution**.
   - **Custom cache and origin-request policies are Business+ only.** On Pro, the `/api/*` behaviour must use AWS-managed policies (e.g. `CachingDisabled` + `AllViewerExceptHostHeader`).
   - **VPC origins** (a private ALB with no public IPs) are Business+ only.
   - Accounts on the AWS Free Tier plan are ineligible.
   - The plan includes S3 storage credits (Pro 50 GB, Business 1 TB), which offset $1.15 / $23 of S3 Standard.
3. **Write Pages as S3 Intelligent-Tiering** (Ingest sets `StorageClass=INTELLIGENT_TIERING` on PUT, so there's no lifecycle transition fee). Manga libraries are read once and then sit, which is the ideal pattern: Pages untouched for 30 days move to IA ($0.0125) and after 90 days to **Archive Instant Access ($0.004)**, with millisecond retrieval and no retrieval fees. At ~1.3 MB per Page, monitoring costs ≈ $0.002/GB-mo against up to $0.019/GB-mo saved. This **halves steady-state storage** (≈ $920 → ≈ $400 at 10k). Covers and manifests are under 128 KB, so they are never auto-tiered or monitored, which is fine.
4. **Lossless WebP for PNG Pages** (the option #7 left open). Google reports lossless WebP is ≈ 26% smaller than PNG (WebP). With PNG at ≈ 59% of stored bytes under the assumed mix, that's ≈ 15% off storage and egress. It costs Ingest CPU and a C/cgo or Go WebP encoder, so do it post-V1 if PNG share turns out high. Lossy re-encoding stays out, per #7.
5. **Trim the fixed floor at small scale** (≈ $50–60/mo total):
   - Replace the managed NAT gateway with a NAT instance on t4g.nano (≈ $3–4 + EIP $3.65), or run nodes in public subnets with locked-down security groups and no NAT (each node then pays $3.65 for a public IPv4).
   - Use a db.t4g.micro ($11.68) until the DB is real.
   - Put Karpenter-managed capacity on **spot** Graviton (≈ 55% below on-demand at today's prices).
   - Once the shape is stable, a 1-year Compute Savings Plan and an RDS Reserved Instance cut ≈ 30–40% off the steady compute and DB lines.
   - The EKS $73 can't be removed while EKS is the platform (owner preference). It's the single largest line at small scale.
6. **Keep the observability baseline lean.** Leave the EKS control-plane log types off (or only `authenticator`), use the Infrequent Access log class for app logs ($0.25/GB), and set log retention to 14–30 days. Container Insights and Managed Prometheus are where a "small" CloudWatch bill becomes $50+.
7. **Watch the RDS growth curve.** Text Box rows (~5 KB per Page) grow with stored Pages. At 10k Readers that's ≈ 155 GB and ~300M rows, which is fine on gp3. A t4g.medium (4 GiB) won't hold the working set in memory, though, so expect db.m7g.large Multi-AZ (+ ≈ $150/mo) around this scale. Intelligent-Tiering doesn't apply here, so this is another reason for a quota.

## 5. Default per-Reader storage quota

**Recommendation: 10 GB default** (stored bytes = Page images + cover + manifest, as counted by `used_bytes` in #5). Keep it configurable per Reader, as #5 allows.

| Quota | ≈ Volumes (blended 0.25 GB) | JPG-only (0.15 GB) | PNG-only (0.5 GB) | S3 Standard at full quota | IT steady state |
|---|---:|---:|---:|---:|---:|
| 5 GB | 20 | 33 | 10 | $0.115 | ≈ $0.05 |
| **10 GB** | **40** | **≈ 65** | **20** | **$0.23** | **≈ $0.10** |
| 25 GB | 100 | 165 | 50 | $0.575 | ≈ $0.25 |
| 50 GB | 200 | 330 | 100 | $1.15 | ≈ $0.51 |

Why 10 GB:

- **It holds a real reading backlog** (a couple of long Series) without becoming an archive service. 5 GB is only 10 PNG Volumes, which is too tight for someone who uploads a whole Series at once.
- **It bounds the worst case.** If every one of 10k Readers filled the quota, storage would be 100 TB ≈ $2,250/mo in Standard or ≈ $1,000 with Intelligent-Tiering. That's high, but it's capped and predictable. At the assumed 4 GB average, the expected bill stays at the §3 numbers.
- **Marginal cost per Reader at the full 10 GB:**
  - S3: **$0.23/mo** (Standard), ≈ $0.10 (IT steady state)
  - RDS Text Boxes: ≈ 8,000 Pages × 5 KB = 40 MB → ≈ $0.01/mo Multi-AZ
  - Ingest PUTs: ≈ $0.04 **once** to fill the quota
  - Egress, which the quota bounds only loosely because reading speed bounds it: a heavy Reader at 3,000 Pages/mo uses ≈ 4.1 GB ≈ **$0.35/mo** on PAYG (above the free 1 TB), and ≈ $0 marginal under a flat-rate plan
  - Cognito: $0 under 10k MAU, $0.015 above
  - **Worst-case heavy Reader at the full quota ≈ $0.60/mo (PAYG CloudFront, S3 Standard), ≈ $0.25/mo with S3 Standard and a flat-rate plan, or ≈ $0.11/mo with Intelligent-Tiering and a flat-rate plan.**
- **The quota doesn't cap churn.** Repeatedly deleting and re-uploading costs Ingest PUTs (≈ $0.04 per 10 GB) and Ingest CPU, but not storage. Leave that to the abuse and rate-limiting ticket (map: "Not yet specified").
- **Revisit with real data.** After the beta, replace the 1.275 MB per Page and the 4 GB average with measured values from `used_bytes`, which is already tracked. If the PNG share is high, lossless WebP (§4.4) effectively raises the quota by ~25% for those Readers at no extra cost.

## 6. Open points and uncertainties

- **Bytes per Page is the weakest input.** The JPG figure rests on one real-world data point (reCBZ, 180 MB per Volume). The PNG figure is a range from informal reports. Storage, egress and the quota scale linearly with it.
- **Spot prices** are a 2026-11-26 snapshot.
- **CloudFront flat-rate + signed cookies**: verify on a test distribution before relying on it (§4.2).
- **Cross-AZ traffic** (pods ↔ RDS, ALB ↔ pods, at $0.01/GB each way) is left out. At these volumes it's a few dollars.
- **Not modelled:** AWS Backup or S3 versioning/replication (the backup/DR ticket isn't specified yet), WAF on pay-as-you-go ($5/web ACL + $1/rule + $0.60/M requests, included in flat-rate plans), support plans and tax.
