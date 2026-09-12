# Experiments — candidate segmentation backbones (refs #11)

Quick, reproducible architecture comparison for the new-backbones issue:
SegFormer, DeepLabV3+ (ConvNeXt-Tiny) and U-Net (ConvNeXt-Tiny /
EfficientNet-B0) trained on BDAPPV positives, evaluated with the same
protocol so the numbers are directly comparable.

## Setup

```bash
pip install -r requirements-train.txt
```

The inference pipeline is not needed — `train.py` / `eval.py` only depend on
`requirements-train.txt`.

## Data (one-time download)

Full dataset (~8 GB), or IGN-only (~2.5 GB) since the default provider is `ign`:

```bash
# full
hf download gabrielkasmi/bdappv --repo-type dataset --local-dir data/bdappv

# ign images/masks + manifests only (google/ skipped)
hf download gabrielkasmi/bdappv --repo-type dataset \
    --include "ign/*" "*.csv" --local-dir data/bdappv
```

(Older hub versions use `huggingface-cli download` with the same arguments.)

Expected layout (`data_dir` in `configs/train.yaml`):

```
data/bdappv/
├── ign/            # or google/  (data_provider in configs/train.yaml)
│   ├── img/*.png   # 400x400 aerial images
│   └── mask/*.png  # binary PV masks (positives only)
├── annotations.csv # canonical split manifest
└── metadata.csv
```

## Environment variables

Only needed for `scripts/upload_results.sh` (HF Hub uploads) — copy
`.env.example` to `.env` and:

```bash
set -a; . ./.env; set +a
```

`HUGGINGFACE_HUB_TOKEN` is never required for training; anonymous downloads
cover the encoder pretraining weights and the dataset.

## Quick runs (1% subset, 3 epochs)

```bash
bash scripts/run_quick.sh segformer
bash scripts/run_quick.sh deeplab
bash scripts/run_quick.sh unet
# or all three:
bash scripts/run_quick.sh all
```

Each invocation:

1. trains `runs/<model>-mini/` (checkpoints, metrics.json, config dump,
   val_preds previews) — **not committed**, gitignored
2. evaluates the best checkpoint on the same validation subset and writes
   `experiments/results/<model>-mini.json` + preview PNGs under
   `experiments/results/previews/<model>-mini/` — **these are committed**

Equivalent direct commands:

```bash
python train.py --config configs/train.yaml --model segformer \
    --subset 0.01 --run-name segformer-mini
python eval.py --checkpoint runs/segformer-mini/checkpoints/best.pth \
    --out experiments/results/segformer-mini.json \
    --previews-dir experiments/results/previews/segformer-mini
```

## Full runs

Edit `configs/train.yaml` (or pass CLI overrides): `subset: 1.0`, more epochs,
possibly a larger `batch_size`. Use a distinct `--run-name`:

```bash
python train.py --model segformer --subset 1.0 --epochs 50 --run-name segformer-full
```

## Protocol notes

- **Split**: canonical department-based train/val/test split from
  `annotations.csv` (dataset seed 42). Do not re-split — published BDAPPV
  results depend on it. `split_mode: random` exists only as a fallback and
  prints a warning.
- **Provider**: default `ign` — matches the pipeline's IGN 20 cm imagery and
  the Etalab 2.0 license. Switch to `google` (BDAPPV paper baselines) via
  `data_provider`. Google imagery carries Google Earth Engine ToS
  redistribution restrictions — relevant when publishing weights later.
- **Image size**: BDAPPV is 400×400; default `image_size: 512` upsamples.
  Keep multiples of 32 (SegFormer requirement).
- **Quick-run hyperparams** (held constant across models for comparability):
  3 epochs, batch 8, AdamW lr 1e-4, seed 42, basic augmentations
  (hflip/vflip/rot90). Same seed + `--subset` fraction ⇒ identical data
  subsets across models.

## Metrics

| key | definition |
|---|---|
| `mIoU` | mean of PV-class and background IoU |
| `pixelF1` | foreground (PV) pixel F1 |
| `instanceF1` | connected components matched one-to-one at IoU ≥ 0.5 (a proxy for per-installation detection) |
| `inference_fps` | single-image forward passes/s at `image_size` (eval.py) |

## Commit policy

- **Commit**: `experiments/results/*.json` and small preview PNGs under
  `experiments/results/previews/`
- **Never commit**: checkpoints (`*.pth`, `*.ckpt`), `runs/`, `.env`, tokens —
  all gitignored
- **Large artifacts**: `scripts/upload_results.sh` (private HF repo by default)

## Smoke test without the dataset

```bash
python train.py --dry-run
```

Synthetic data, 1 epoch, CPU, no downloads — verifies the whole
train → checkpoint → metrics → previews plumbing.
