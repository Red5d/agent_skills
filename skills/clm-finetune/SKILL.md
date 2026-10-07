---
name: clm-finetune
description: "Use when fine-tuning CLM decision-model heads on typed tasks (choice/noul/score)."
version: 1.1.0
---

# CLM Fine-Tuning (typed decision heads)

A verified end-to-end pipeline for fine-tuning CLM base checkpoints into
per-domain typed decision heads (pilot run: test accuracy 0.425 → 0.879 in 2
epochs; full-scale run: val 0.974 / test 0.966 in ~3 min on CPU).

Paths below use placeholders. Pick your own working locations and export them
once:

```bash
export CLM_REPO=~/CLM            # clone of github.com/Contrastive-LM/CLM
export CLM_FT=~/clm-ft           # your working dir (datasets, caches, runs)
export CLM_CKPT=~/.cache/clm/CLM_v0.1-8B.pt   # base checkpoint (via clm-download)
```

Set up a virtualenv with the training deps; `pyarrow` is required for the
dataset tooling and is not in the default dependency set — install it into
the venv explicitly.

## Embedding server

Embeddings come from an OpenAI-compatible embedding endpoint (llama-server
with a GGUF embedding model works; any `/v1/embeddings` server with the same
tokenizer works). Run it on a machine with enough RAM for the model — not
inside a small container:

```bash
llama-server -m <path-to-Qwen3-8B-Q4_K_M.gguf> \
  --embedding --pooling last -ngl 99 -c 4096 --host 0.0.0.0 --port 8090
```

Notes:
- `--embed-model` in the trainer is the HF repo id (tokenizer source);
  `--served-model-name` is what llama-server reports. Mixing these up fails
  with "is not a valid model identifier".
- llama-server accepts both token-id arrays and text in `/v1/embeddings` with
  base64 `encoding_format` — CLM's ServerBackend plugs in unmodified.

## Dataset format

Parquet rows: `{id, state (str or dict), questions (System One wire format:
type/instructions/criteria), gold: {qid: {"label": key}}, workflow}`. Choice
labels are zero-based index strings; noul labels are true/false.

Directory layout REQUIRED: `root/<workflow>/train.parquet` + `test.parquet`
(the test split is used as-is for eval). If you mirror an existing decision
model's prompts/question text, reuse that exact text for comparability.

The dataset builder constructs labels by design: known-malicious payloads
(e.g. SecLists — note that paths/filenames change over time; browse
`api.github.com/repos/danielmiessler/SecLists/contents/<dir>` to find the
current ones) plus benign logs.

## Embedding cache (required before full-scale training)

At 37k+ unique texts the in-process ServerBackend accumulates ~600MB+ of
fp32 vectors and will OOM small environments. Pre-embed with the chunked
driver instead — run once per split; it appends to a shared cache:

```bash
<venv>/bin/python $CLM_FT/preembed.py <parquet> --embed-cache $CLM_FT
```

The cache filename is DERIVED from the HF id: the slug keeps hyphens
(e.g. `choice_Qwen_Qwen3-8B_2048.npz` under `--embed-cache`). Match exactly
or the trainer silently re-embeds and OOMs.

## Training (validated configuration)

If any embedding is missing, train on the largest-RAM machine available.
With a complete cache, `--embed-url` can be omitted entirely (the embedding
server need not run). Use `--loss infonce`: the `softce` loss builds a dense
q+c float32 matrix (~2.4GB at 28.8k questions) and OOMs even with a warm
cache on small machines.

```bash
<venv>/bin/python $CLM_REPO/train/finetune.py --task choice \
  --data $CLM_FT/logrisk_ds --workflow logrisk \
  --embed-model Qwen/Qwen3-8B --embed-cache $CLM_FT \
  --init-ckpt $CLM_CKPT \
  --loss infonce --targets hard --epochs 20 --batch 512 \
  --out-dir $CLM_FT/<run>
```

Embedding cache per run dir (sha1→vec npz) means reruns skip re-embedding.

## Evaluation

Held-out REAL data from the target domain, not the generator's test split —
a "test" split produced by the same generator is optimistic. Evaluate the
trained `best_head.pt` against held-out real logs plus any benchmark suites
you have; keep a small eval script per domain.

## Per-domain heads are the deployment shape

The fine-tuned head regresses on OTHER suites (it drifts toward the
fine-tuned domain). Deploy per-domain: the fine-tuned head for its domain,
the original base checkpoint for everything else.

## Pitfalls

- `--data` must be `root/<workflow>/` with `train*/test*` parquet basenames —
  `load_typed_rows` filters by basename prefix when given a dir, and treats a
  stray top-level filename as an HF repo id (fails with HFValidationError).
- `pq.write_table` does NOT create parent dirs — makedirs first.
- Files are passed directly to `pq.read_table` — JSONL will NOT load; the
  dataset must be parquet.
- Container/RAM caps: never run vLLM or encoder loads inside a small
  container; heads-only training on CPU is fine.
