Refs #11 — WIP draft PR

## What this PR contains (WIP)

- **Training scaffold**: `train.py`, `eval.py`, `configs/{train,model}.yaml`,
  `scripts/run_quick.sh`, `scripts/upload_results.sh`,
  `requirements-train.txt` (kept separate from the pipeline's
  `requirements.txt`)
- **Quick-experiment protocol** for the candidate backbones — 1% subset,
  3 epochs, identical seed/subset logic across models so runs are directly
  comparable:
  - SegFormer-B0 (`segformer-mini`)
  - DeepLabV3+ w/ ConvNeXt-Tiny (`deeplab-mini`)
  - U-Net w/ ConvNeXt-Tiny (`unet-mini`)
  - (also available: SegFormer-B1, U-Net w/ EfficientNet-B0)
- **Results summaries** (`experiments/results/*.json`) and preview images
  (`experiments/results/previews/`) — filled in as quick runs complete
- **No weights committed**: `.gitignore` covers `runs/`, `models/`, `*.pth`,
  `*.ckpt`, `logs/`; `.env.example` documents the HF token placeholder

## Design notes

- **Dataset**: BDAPPV via HF snapshot (`gabrielkasmi/bdappv`). Default
  provider `ign` — matches the pipeline's IGN 20 cm imagery and the Etalab 2.0
  license (Google imagery carries redistribution restrictions that matter when
  we later publish weights). Switchable to `google` via `configs/train.yaml`.
- **Split**: the canonical department-based train/val/test split shipped with
  the dataset (`annotations.csv`, dataset seed 42). The dataset card asks not
  to re-split; a seeded random fallback exists but warns loudly.
- **Metrics**: mIoU, pixel F1, and instance-level F1 (connected components
  matched one-to-one at IoU ≥ 0.5). The issue draft mentions
  `val_buildingF1`; BDAPPV masks are PV-only (no building channel), so this PR
  reports `val_instanceF1` as the object-level metric — happy to rename if
  maintainers prefer.
- **Checkpoints** are state-dict + model-spec dicts (no full-model pickles),
  ready for the loader in #7 to consume via `pretrained_hf_id` (placeholder in
  `configs/model.yaml`).
- **CI**: not included yet — happy to add a minimal workflow (flake8 +
  `train.py --dry-run` on synthetic data, no dataset download) once the
  approach is confirmed.

## Initial results (quick small-scale runs)

> ⏳ Pending — quick runs (1% subset, 3 epochs) are executing; numbers below
> and in `experiments/results/` will be updated as they complete.

| Model | Params | val_mIoU | val_pixelF1 | val_instanceF1 | Inference FPS (512×512) |
|---|---|---|---|---|---|
| SegFormer-B0 | — | — | — | — | — |
| DeepLabV3+ (ConvNeXt-Tiny) | — | — | — | — | — |
| U-Net (ConvNeXt-Tiny) | — | — | — | — | — |

(Full metrics in `experiments/results/*.json`; input/GT/prediction previews in
`experiments/results/previews/`.)

## What I ran

Quick small-scale experiments on a 1% subset / 3 epochs to compare
architectures quickly (identical data, seed and hyperparams across models):

```bash
bash scripts/run_quick.sh segformer
bash scripts/run_quick.sh deeplab
bash scripts/run_quick.sh unet
```

Each runs `python train.py --config configs/train.yaml --model <m> --subset 0.01
--run-name <m>-mini` and evaluates `best.pth` into
`experiments/results/<m>-mini.json` + previews.

## Notes & next steps

If the maintainers are happy with this approach, I'll:

- run full-scale training for the most promising backbone(s) on the canonical
  (spatial, department-based) split,
- produce final benchmarks vs the Inception/DeepLab baselines on BDAPPV,
- upload final weights and a model card (limitations, intended use, privacy
  notice) to Hugging Face, pending license confirmation.

## Questions for maintainers

1. Any preference among SegFormer / DeepLab / U-Net families to prioritize?
2. The canonical BDAPPV split (department-based, seed 42) is used for
   training/validation — should final benchmarks be reported on `val`, `test`,
   or both? Is the cross-provider shift protocol (train google → test ign) in
   scope?
3. Preferred license for final weights/code — CC-BY, MIT, or both?
4. Is `ign` as the default provider config acceptable (vs `google` used by the
   paper baselines)?
