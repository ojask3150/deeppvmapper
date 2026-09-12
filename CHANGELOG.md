# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Added
- Training scaffold for candidate segmentation backbones on BDAPPV (refs #11):
  `train.py` / `eval.py`, `configs/`, quick-experiment scripts
  (`scripts/run_quick.sh`, `scripts/upload_results.sh`) and `experiments/`
  result summaries. Weights and runs are gitignored, never committed.

## [1.0] - 2026-06-17

First tagged stable release — pipeline validated end-to-end.

### Added
- Asynchronous tile decode/prefetch: a background process pool now decodes
  JP2 tiles while the GPU runs classification, instead of loading tiles
  sequentially before each batch. Closes #2.
- `decode_workers` / `decode_stagger_s` config options to tune the decode
  pool size and stagger initial submissions (avoids lockstep bursty waits).

### Changed
- Cleared the leftover single-tile test filter in `config.yml`'s default
  `tiles_list`.
