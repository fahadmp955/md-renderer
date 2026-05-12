# AWS Lambda Free Tier And Running Cost Guide

> A practical cost note for this tarot video service.
>
> Verified against AWS pricing pages on **May 12, 2026**. Prices can vary by region, so this document uses **`us-east-1` style assumptions** unless noted otherwise.

---

## The Short Answer

If we move the API layer to AWS Lambda, the **free tier is generous for normal API traffic** but **less generous for video rendering**.

- For a lightweight API Lambda, you can often stay near **1,000,000 requests/month** in the free tier.
- For this tarot video project, the **Remotion render workload** is the real limiter.
- Using a conservative model, I would expect roughly **600-700 video renders/month** to fit inside Lambda free-tier compute.
- If the service stays in its current shape with an **always-on Nest server + worker + Postgres**, a realistic idle baseline is about **$25-$55/month** before OpenAI / ElevenLabs / storage-heavy growth.
- If you go **more serverless**, the infrastructure baseline can be closer to **$0-$5/month at very low traffic**, and then scale mostly with usage.

---

## Official AWS Numbers To Anchor On

From AWS Lambda pricing:

- **1,000,000 free requests / month**
- **400,000 GB-seconds of free compute / month**
- After free tier: **$0.20 per 1 million requests**
- After free tier compute: **$0.0000166667 per GB-second**

From API Gateway HTTP API pricing:

- **1,000,000 free HTTP API calls / month** for eligible free-tier usage
- Then about **$1.00 per million requests** at the first tier shown by AWS

These are the most important numbers for quick planning.

---

## How To Think About Lambda Usage

Lambda cost is mostly:

```text
requests + (memory in GB × execution time in seconds)
```

That second part is the important one.

### Example: simple API endpoint

If one Lambda uses:

- `512 MB`
- `300 ms`

Then each call uses:

```text
0.5 GB × 0.3 s = 0.15 GB-s
```

Free-tier compute capacity:

```text
400,000 / 0.15 = 2,666,666 calls
```

But the request free tier is only **1,000,000 requests**, so in that case the **request limit hits first**.

That means a lightweight API can often get **about 1 million free calls/month**.

---

## My Assumptions For This Tarot Video Service

These assumptions are based on the code in this repo:

- The backend generates a tarot reading, audio, and then a Remotion Lambda render.
- The render duration is tied to audio length in [`src/video-gen/adapters/remotion-lambda-video.adapter.ts`](/Users/fahadchougle/Work/tarot-ai-server/src/video-gen/adapters/remotion-lambda-video.adapter.ts:1).
- The default target frame rate is effectively around **20 FPS** from env fallback logic.
- A typical reading likely becomes a **45-60 second video**.
- Remotion Lambda rendering usually means **multiple Lambda invocations**, not one tiny request.

For planning, I am assuming:

| Item | Assumption | Why |
| --- | --- | --- |
| Region | `us-east-1` style pricing | Common reference region |
| Final video length | `50 seconds` | Matches short tarot reading format |
| API/orchestration compute | `0.5 GB-s per reading` | Small compared to render |
| Remotion render compute | `600 GB-s per video` | Conservative model for parallel render + stitch |
| Output size | `25 MB per video` | Reasonable for short H.264 output |
| No provisioned concurrency | `true` | Cheaper and simpler for early stage |

### Why I picked `600 GB-s` per video

This is not an AWS published number. It is a **planning estimate**.

I chose it because:

- Remotion render jobs are compute-heavy compared to normal API Lambdas.
- The composition includes image sequences, audio, transitions, and final encoding.
- Lambda render pipelines often fan out into multiple workers, so total compute is the **sum of all workers**, not just wall-clock time.

I would treat `600 GB-s/video` as a **safe planning assumption**, not a promise.

---

## Free Tier Capacity For This Project

### 1. Lightweight API usage

If your API Lambda stays small, the free tier can comfortably handle:

- about **1,000,000 API calls/month**

because request count usually hits before compute.

### 2. Tarot video renders

Using the planning assumption:

```text
400,000 GB-s / 600 GB-s per render = 666 renders/month
```

So a useful planning number is:

- **~650 free video renders/month**

That is the number I would use for budgeting conversations.

---

## What It Might Cost Per Month

### Scenario A: Very early hobby usage

Assume:

- `200` video renders / month
- `200` API calls to start those jobs
- `5 GB` of stored outputs kept around

Estimated infra cost:

| Component | Estimated cost |
| --- | ---: |
| Lambda compute | `$0.00` |
| Lambda requests | `$0.00` |
| API Gateway | `$0.00` or near-zero |
| S3 storage + requests | `~$0.10-$0.50` |
| Logs / misc | `~$0-$2` |
| **Total infra** | **~$0-$3/month** |

This is why Lambda feels magical for small projects.

### Scenario B: Early traction

Assume:

- `700` video renders / month
- `700` API calls
- `20 GB` stored in S3

Lambda compute estimate:

```text
700 × 600 GB-s = 420,000 GB-s
Billable after free tier = 20,000 GB-s
20,000 × $0.0000166667 = $0.33
```

Estimated infra cost:

| Component | Estimated cost |
| --- | ---: |
| Lambda compute | `~$0.33` |
| Lambda requests | `~$0.00` |
| API Gateway | `~$0.00` to tiny |
| S3 storage + requests | `~$0.50-$1.50` |
| Logs / misc | `~$1-$3` |
| **Total infra** | **~$2-$5/month** |

Even here, Lambda itself is still cheap.

### Scenario C: Small public launch

Assume:

- `2,000` video renders / month
- `2,000` API calls
- `50 GB` stored in S3

Lambda compute estimate:

```text
2,000 × 600 GB-s = 1,200,000 GB-s
Billable after free tier = 800,000 GB-s
800,000 × $0.0000166667 = $13.33
```

Estimated infra cost:

| Component | Estimated cost |
| --- | ---: |
| Lambda compute | `~$13.33` |
| Lambda requests | `~$0.00-$0.05` |
| API Gateway | `~$0.00-$1` |
| S3 storage + requests | `~$1-$3` |
| Logs / misc | `~$2-$5` |
| **Total infra** | **~$16-$22/month** |

This is still very reasonable for video infrastructure.

---

## The Bigger Truth: Lambda May Not Be Your Main Cost

For this repo, the likely top costs are often **not Lambda**.

The more expensive pieces may be:

- OpenAI usage for script generation
- ElevenLabs or other TTS usage
- Database hosting if you keep a dedicated Postgres instance running 24/7
- Long-term S3 storage if you keep every generated video forever
- CDN / bandwidth if lots of users watch videos repeatedly

In other words:

> Lambda is probably **not** the part that will hurt first.

For an AI video product, **model and media costs usually dominate before raw Lambda pricing does**.

---

## Cost To Keep The Service Running

There are really **two different architectures** you could mean.

### Option 1: Keep the current app shape

This repo currently looks like an app that expects:

- a NestJS API
- a background worker loop
- a Postgres database
- storage for media

If you keep that shape with small always-on infrastructure, I would budget:

| Baseline item | Planning estimate |
| --- | ---: |
| Small app server / container | `$10-$20/month` |
| Small Postgres instance | `$15-$30/month` |
| Storage, logs, tiny extras | `$1-$5/month` |
| **Always-on baseline** | **~$25-$55/month** |

This is **before** OpenAI, TTS, and larger media usage.

### Option 2: Go more serverless

If you move the API and worker logic toward:

- API Gateway
- Lambda
- S3
- managed queue/events
- a low-cost or serverless database

then the idle cost can be much lower:

| Baseline item | Planning estimate |
| --- | ---: |
| Lambda + API Gateway idle | `$0-$1/month` |
| Small S3 footprint | `$0-$1/month` |
| Logs / misc | `$0-$3/month` |
| Database | varies the most |
| **Serverless-style baseline** | **~$0-$5/month plus DB** |

This is the setup where Lambda gives you the most leverage.

---

## My Recommendation

If your goal is to launch lean:

1. Keep the render path on **Remotion Lambda**.
2. Treat **~650 renders/month** as a conservative free-tier planning number.
3. Expect **Lambda cost to stay small** even after you outgrow free tier.
4. Focus cost control on AI model usage, TTS minutes, video retention, and whether your API/worker/database stay always on.

If you want one simple budgeting sentence:

> For this project, Lambda can probably cover your first few hundred renders very cheaply, and your biggest recurring cost is more likely to come from AI services and always-on infrastructure than from Lambda itself.

---

## Sources

- AWS Lambda pricing: https://aws.amazon.com/lambda/pricing/
- Amazon API Gateway pricing: https://aws.amazon.com/api-gateway/pricing/
- Amazon S3 pricing: https://aws.amazon.com/s3/pricing/

## Notes

- This file intentionally mixes **official AWS numbers** with **my planning assumptions**.
- The official numbers are sourced above.
- The render-per-video estimate is **my model for this repo**, not an AWS guarantee.
