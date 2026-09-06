# Portfolio assistant

A FastAPI service on Cloud Run that answers questions about you, using
`knowledge.md` as its only source. The site's chat widget streams from it.

```
browser (GitHub Pages)  ──POST /chat──▶  Cloud Run (FastAPI)  ──▶  Claude Haiku 4.5
        ▲                                        │
        └────────── SSE token stream ────────────┘
```

## Why there's no vector database

Your whole corpus is ~4 KB. That fits in the system prompt with room to spare,
so there is nothing to retrieve *from* — embeddings and a vector store would add
moving parts and latency to solve a problem you don't have. The dossier is
marked as a cache breakpoint, so after the first call those tokens are billed at
about a tenth of the normal rate.

Revisit this only if the knowledge base grows past roughly 50 KB.

## One-time setup

**1. Get an API key** at console.anthropic.com → API Keys.

**2. Set a spend cap** in Billing → Limits. This is the real ceiling on cost —
do it before you deploy, not after. $5/month is plenty.

**3. Store the key in Secret Manager** (never in the image, never in the repo):

```bash
gcloud services enable run.googleapis.com secretmanager.googleapis.com \
  cloudbuild.googleapis.com

printf 'sk-ant-YOUR-KEY' | gcloud secrets create anthropic-key --data-file=-
```

**4. Deploy:**

```bash
cd assistant
gcloud run deploy portfolio-assistant \
  --source . \
  --region us-west1 \
  --allow-unauthenticated \
  --min-instances 0 \
  --max-instances 3 \
  --memory 512Mi \
  --set-secrets ANTHROPIC_API_KEY=anthropic-key:latest \
  --set-env-vars ALLOWED_ORIGINS=https://christopherpearce1.github.io
```

`--max-instances 3` caps how much can run at once, which caps the blast radius
if someone hammers it. `--min-instances 0` means you pay nothing while idle, at
the cost of a ~2s cold start on the first question after a quiet spell.

**5. Wire up the site.** The deploy prints a service URL. Paste it into
`ASSISTANT_ENDPOINT` near the bottom of `index.html`:

```js
const ASSISTANT_ENDPOINT = "https://portfolio-assistant-xxxxx-uw.a.run.app";
```

Empty string = widget stays hidden and the ChatGPT/Claude/Perplexity links act
as the fallback. Non-empty = chat panel appears. Nothing else to toggle.

**6. Verify:**

```bash
curl https://YOUR-SERVICE-URL/health
# {"ok":true,"model":"claude-haiku-4-5"}
```

## Keeping it current

`knowledge.md` is a copy of the site's `llms.txt`. When you update one, update
the other and redeploy, or the assistant will answer from stale facts:

```bash
cp ../llms.txt knowledge.md && gcloud run deploy portfolio-assistant --source .
```

## What it costs

Haiku 4.5 is $1 per million input tokens and $5 per million output, with cached
input reads at $0.10 per million.

| per exchange | tokens | cost |
|---|---|---|
| dossier + instructions (cached) | ~1,400 | $0.00014 |
| question + history | ~300 | $0.0003 |
| answer | ~250 | $0.00125 |
| **total** | | **~$0.0017** |

So roughly **$1.70 per 1,000 questions**. Cloud Run's free tier (2M requests and
180k vCPU-seconds a month) covers a portfolio site's traffic outright, so the
hosting line is $0. Realistically you're looking at **under $2/month**, and the
spend cap from step 2 guarantees it can't surprise you.

## Abuse controls

The endpoint is public and unauthenticated, which is the right call for a
portfolio — a login would kill the feature. So the limits are layered:

| control | where | value |
|---|---|---|
| requests per IP | `main.py` | 15/hour |
| message length | `main.py` | 600 chars |
| conversation turns sent | `main.py` | last 12 |
| output tokens | `main.py` | 500 |
| allowed origins | CORS | your domain only |
| concurrent instances | Cloud Run | 3 |
| monthly spend | Anthropic console | your cap |

The per-IP limit is in-memory, so it's per container instance. With
`--max-instances 3` the worst case is 3× the nominal limit, which is fine at
this scale. If you ever want it exact, move the counter to Firestore.

CORS keeps a *browser* on another site from calling this, but it does not stop
`curl`. The spend cap and instance cap are what actually bound your exposure —
treat CORS as tidiness, not security.

## Guardrails

The system prompt in `main.py` pins the assistant to the dossier, tells it to
say "not published" rather than guess, and to ignore instructions embedded in
visitor messages. Worth testing once it's live — ask it something you know isn't
in `knowledge.md` and confirm it declines instead of inventing an answer.

## Local development

```bash
pip install -r requirements.txt
export ANTHROPIC_API_KEY=sk-ant-...
export ALLOWED_ORIGINS=http://localhost:8000
uvicorn main:app --reload --port 8080
```

Then serve the site (`python3 -m http.server 8000` from the parent folder) and
set `ASSISTANT_ENDPOINT` to `http://localhost:8080`.
