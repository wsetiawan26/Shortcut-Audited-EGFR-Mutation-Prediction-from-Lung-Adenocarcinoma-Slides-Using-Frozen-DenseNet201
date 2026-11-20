# Shortcut-Audited EGFR Mutation Prediction from Lung Adenocarcinoma Slides Using Frozen DenseNet201

Code for the paper *"Shortcut-Audited EGFR Mutation Prediction from Lung Adenocarcinoma Slides Using Frozen DenseNet201"* (Setiawan, Romadhoni, Alfaini, Putri; submitted to the International Journal of Intelligent Engineering and Systems).

The pipeline predicts EGFR mutation status from H&E whole-slide images (WSIs) of TCGA-LUAD. It combines a **frozen ImageNet DenseNet201** patch encoder with an **attention-based multiple instance learning (ABMIL)** aggregator. The model itself is simple on purpose. The point of the repository is the **evaluation**: a leakage-controlled protocol plus a shortcut audit that asks how much of the reported AUROC is real.

## What the audit checks

| Check | Question it answers |
|---|---|
| Patient-level split, locked hold-out scored once | Is the test number free of selection optimism? |
| Selection estimate vs locked test | How large is the gap between the CV value used for model choice and the hold-out value? |
| Shortcut baselines on the same splits (patch count only, tissue source site only, both) | Can tissue amount or site alone match the model? |
| Patch-count matched subset and fixed-K inference | Does the signal survive when tissue amount is controlled? |
| Site-preserved CV (`StratifiedGroupKFold`, group = TSS) | Does performance hold when no site appears in both training and validation? |
| Five seeds (42–46) for the final model | How stable is the hold-out AUROC to initialisation? |
| Repeated 5×5 CV, DeLong / Mann–Whitney / permutation tests, bootstrap CI | How uncertain is each number? |
| Calibration (Brier score) and Youden threshold from OOF predictions | Are the probabilities usable at a fixed threshold? |
| Attention overlays, top-k patch montages, blind pathologist export | What does the model actually look at? |
| UNI pathology foundation model as backbone (same ABMIL configuration) | How much does a domain-specific encoder change the result? |

## Main results

| Setting | AUROC |
|---|---|
| 5-fold CV, fold-wise mean (used for selection) | 0.827 |
| 5-fold CV, pooled out-of-fold | 0.774 |
| Site-preserved CV, pooled out-of-fold | 0.730 |
| Hold-out, seed 42 (primary result, 40 WSIs, 10 Mutant) | 0.713 (95% CI 0.502–0.914) |
| Hold-out, mean of seeds 42–46 | 0.674 (range 0.620–0.713) |
| Site-only baseline, hold-out | 0.717 |
| Patch-count-only baseline, hold-out | 0.677 |
| UNI + ABMIL, hold-out, mean of five seeds | 0.813 |

The site-only baseline matched the model on the hold-out set, and the gap between the selection estimate and the locked test was more than 0.1 AUROC. We recommend that slide-based mutation studies report shortcut baselines, seed spread, and calibration next to the headline AUROC.

## Data

- **Images:** 200 diagnostic WSIs from TCGA-LUAD (one WSI per patient), [GDC Data Portal](https://portal.gdc.cancer.gov/), project TCGA-LUAD.
- **Labels:** EGFR mutation status from cBioPortal, study `luad_tcga_pan_can_atlas_2018` (50 Mutant, 150 Wild-Type).
- **Split:** 160 development cases (40 Mutant) and 40 locked hold-out cases (10 Mutant), split at patient level.
- **Patches:** 224 × 224 px at 40×, Otsu tissue mask, tissue and blur filters, Macenko stain normalisation with a reference taken from development slides only. All kept patches are used (no cap per slide): 11,579,207 patches in total, 2,115–165,616 per WSI.

The WSIs and labels are not redistributed here. Patch extraction and Macenko normalisation are done by a separate preprocessing step. The notebook expects its output: one ZIP of normalised patches per WSI plus `macenko_summary.csv`.

## Repository contents

```
FINALL_DENSENET201ABMIL.ipynb   main Colab notebook (cells 1–35)
README.md
```

The notebook comments and cell descriptions are in Indonesian.

## Notebook workflow

| Cells | Stage | Hardware |
|---|---|---|
| 1–4 | Mount Drive, configuration (`A`), quick data check, imports, ZIP reader, catalogue and split | CPU |
| 5 | Stage 1: scan ZIPs and list every patch per WSI | CPU |
| 6–8 | Stage 2: DenseNet201 feature extraction (1,920-d per patch, frozen, one `.pt` file per WSI) | T4 |
| 9–11 | ABMIL model (`MILNet`) and experiment engine (`Exp`); Stage 3: 5-fold CV for loss (BCE, weighted BCE, focal), 24-configuration tuning, and pooling ablation (attention vs mean vs max) | T4 |
| 12–13 | Stage 4: retrain on 160 cases and score the 40 hold-out cases **once** (threshold 0.5); bootstrap and Clopper–Pearson CIs; efficiency | T4 |
| 14–28 | Revision 1 analyses: Wilcoxon tests, site summary, DeLong tests, calibration, site-preserved CV, Youden threshold, five seeds, FLOPs and inference time, attention heatmaps, repeated 5×5 CV, UNI backbone | A100 recommended |
| 29–35 | Revision 2 analyses: shortcut baselines, UNI with five seeds, blind top-10 patch export for pathologist review, attention overlays on H&E thumbnails, EGFR prevalence from cBioPortal, paired DeLong UNI vs DenseNet201, patch-count control | CPU / GPU |

The hold-out stage is locked behind `KONFIRMASI_HOLDOUT = False` in cell 2. Set it to `True` only after cross-validation and tuning are finished. All analyses from cell 14 onward are secondary. They read saved predictions or retrain with the frozen final configuration and never change the primary hold-out result.

### Final configuration

Class-weighted BCE, attention pooling, hidden 256, attention dimension 128, dropout 0.25, learning rate 1e-4, weight decay 1e-5, 21 epochs (median best epoch across folds), seed 42, threshold 0.5. DenseNet201 has 18.09 M frozen parameters; the ABMIL head has 525,058 trainable parameters.

## How to run

1. Open the notebook in Google Colab and connect a GPU runtime (T4 for the main pipeline, A100 or ≥ 40 GB for the revision cells and UNI).
2. Put the Macenko-normalised patch ZIPs and `macenko_summary.csv` on Google Drive, and set the paths in cell 2.
3. Run cells 1–11 in order, then unlock and run cell 12 once.
4. Run the secondary analyses as needed. Each cell's header lists its prerequisites and output files.

Results are written to Google Drive (`04_hasil…` for the main pipeline and `05_analisis_tambahan_revisi1/` for the secondary analyses).

### Main dependencies

Python 3, PyTorch, torchvision, scikit-learn, SciPy, NumPy, pandas, matplotlib. The UNI cells also need `timm`, `huggingface_hub`, and approved access to [`MahmoodLab/UNI`](https://huggingface.co/MahmoodLab/UNI). UNI weights are distributed by their authors under CC-BY-NC-ND 4.0 and are not included here.

## Citation

The paper is under review. Please cite the repository for now:

```bibtex
@misc{setiawan2026egfrshortcut,
  author = {Setiawan, Wahyudi and Romadhoni, Natasya Haidar and Alfaini, Muhammad Fajri and Putri, Deka Miftalia Anugrah},
  title  = {Shortcut-Audited {EGFR} Mutation Prediction from Lung Adenocarcinoma Slides Using Frozen {DenseNet201}},
  year   = {2026},
  howpublished = {GitHub repository},
  note   = {Code for a manuscript submitted to the International Journal of Intelligent Engineering and Systems}
}
```

## Contact

Wahyudi Setiawan, Department of Information Systems, Faculty of Engineering, Universitas Trunojoyo Madura, Bangkalan, Indonesia — wsetiawan@trunojoyo.ac.id

## License

[Choose a license before publishing, for example MIT for the code.]

This code is for research use only. It is not a medical device and must not be used for clinical decisions.
