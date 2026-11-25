# Monocular Depth Estimation for Pothole Analysis

Comparative fine-tuning and evaluation of four pretrained monocular depth-estimation models on the **Pothole-440** road-surface dataset.

The project compares MiDaS v3.1, Depth Anything V1 Small, Depth Anything V2 Small, and YOLO26-Depth Nano using a common set of normalized scanner-derived reference targets. The goal is to study their agreement with the prepared targets and their computational requirements for pothole/road-surface analysis.

> **Important target limitation:** The Pothole-440 scanner exports used in this project were available as grayscale depth maps without a verified physical encoding or scale. The targets were globally normalized to \([0,1]\); they are **not verified metric depth in metres or centimetres**. Therefore, the reported MAE and RMSE measure agreement with the prepared normalized scanner-intensity targets, not physical pothole-depth accuracy.

## Results at a glance

All four models were evaluated on the same **64 held-out test images**, using a common \(256 \times 256\) evaluation grid and **1,183,198 valid pixels**.

| Model | Pixel-weighted MAE ↓ | Pixel-weighted RMSE ↓ | Mean inference latency / image ↓ | Parameters | Checkpoint size |
|---|---:|---:|---:|---:|---:|
| MiDaS v3.1 | 0.1181 | 0.1557 | 42.84 ms | 40.93 M | 156.27 MB |
| **Depth Anything V1 Small** | **0.0983** | **0.1413** | 86.61 ms | 24.79 M | 94.66 MB |
| Depth Anything V2 Small | 0.1006 | 0.1432 | 87.15 ms | 24.79 M | 94.66 MB |
| YOLO26-Depth Nano | 0.1103 | 0.1525 | **22.01 ms** | **5.17 M** | **10.11 MB** |

- Depth Anything V1 achieved the lowest pixel-weighted MAE and RMSE against the prepared targets.
- YOLO26-Depth Nano had the lowest measured latency and the smallest checkpoint.
- Latency is an approximate end-to-end inference-pipeline measurement. Model input sizes and preprocessing differ, so it is not a perfectly controlled comparison of isolated forward-pass speed.

See [`results/`](results/) for the summary CSV, per-image metrics, experiment notes, and figures (if included in this repository).

## Repository structure

The following is the recommended repository layout. Adjust filenames if your uploaded notebooks retain different names.

```text
.
├── README.md
├── requirements.txt
├── notebooks/
│   ├── 01_Pothole440_Dataset_Preparation.ipynb
│   ├── 02_MiDaS_v3.1_Finetuning.ipynb
│   ├── 03_Depth_Anything_V1_Finetuning.ipynb
│   ├── 04_Depth_Anything_V2_Finetuning.ipynb
│   ├── 05_YOLO26_Depth_Finetuning.ipynb
│   └── 06_Compare_All_Four_Models.ipynb
└── results/
    ├── model_comparison.csv
    ├── all_models_per_image_metrics.csv
    ├── experiment_notes.txt
    └── figures/
        ├── test_set_model_comparison_grid.png
        ├── accuracy_speed_size_comparison.png
        └── per_image_MAE_boxplot.png
```

The `results/` files are derived from the final comparison notebook. Do not commit large datasets, model checkpoints, cached predictions, or temporary training outputs unless there is a specific reason and the repository's storage limits permit them.

## Dataset

The experiments use **Pothole-440**, which contains road RGB images, pothole annotations, and scanner-derived depth maps.

- Original samples: 440
- Samples retained after excluding five samples with unsuitable or incomplete depth information: **435**
- Training: **305 images**
- Validation: **66 images**
- Test: **64 images**

Related image variations were grouped using filename-based relationships before splitting, to reduce the chance of related variants appearing in different subsets. The corrected preparation manifest is the source of truth for split membership; the comparison notebook selects test rows using its `split` column rather than relying on filename prefixes.

Dataset source: [Pothole440 dataset repository](https://github.com/Youngjincha/Pothole440).

The dataset itself is not included in this repository. Obtain it from its source and follow the preparation notebook's instructions.

### Target preparation

The scanner's exact physical encoding and scale were not available. The grayscale values are therefore treated as a scanner-intensity proxy rather than metric distance.

The target values were globally normalized using the 1st and 99th percentiles of the **training split** (16 and 173):

\[
D_{\mathrm{norm}} =
\operatorname{clip}\left(\frac{D-16}{173-16},0,1\right).
\]

Validity masks exclude scanner regions considered unreliable. Some masks were corrected using an automated procedure; they were not independently verified as exact geometric ground truth. These limitations apply to training targets and evaluation.

## Models

| Model | Pretrained model / configuration | Fine-tuning approach |
|---|---|---|
| MiDaS v3.1 | `Intel/dpt-swinv2-tiny-256` (DPT-SwinV2-Tiny-256) | Affine calibration and masked Smooth L1 fine-tuning; initial training followed by corrected-mask fine-tuning |
| Depth Anything V1 Small | `LiheYoung/depth-anything-small-hf` | Training-only affine calibration initialization, then trainable calibration and masked Smooth L1 fine-tuning |
| Depth Anything V2 Small | `depth-anything/Depth-Anything-V2-Small-hf` | Training-only affine calibration initialization, then trainable calibration and masked Smooth L1 fine-tuning |
| YOLO26-Depth Nano | Pretrained Ultralytics YOLO26 Nano depth model | Fine-tuned with the Ultralytics depth-training pipeline |

For the transformer-based models, predictions are mapped to the normalized target range using a learnable affine calibration layer. For Depth Anything V1 and V2, the initial calibration coefficients are estimated from training samples only and remain trainable during fine-tuning. The learned parameters are saved in the checkpoints and reused during testing.

YOLO26-Depth is trained with the Ultralytics depth task. Its training data uses zero values for invalid target pixels as required by the depth-training pipeline. In the shared final comparison, YOLO26 predictions are compared directly with the normalized reference targets; no calibration is fitted on the test set.

## Experimental workflow

Run the notebooks in this order:

1. **Dataset preparation** — prepare grouped splits, globally normalized targets, validity masks, and the corrected manifest.
2. **MiDaS fine-tuning** — train MiDaS v3.1 and retain the best corrected-mask validation checkpoint.
3. **Depth Anything V1 fine-tuning** — train V1 Small using the corrected manifest and masks.
4. **Depth Anything V2 fine-tuning** — train V2 Small using the same split and target protocol.
5. **YOLO26-Depth fine-tuning** — convert the prepared data to the Ultralytics depth dataset layout and fine-tune YOLO26-Depth Nano.
6. **Four-model comparison** — load the selected checkpoints and evaluate all models once on the same held-out test set.

The four model-training notebooks can be run independently after dataset preparation. The comparison notebook requires the prepared dataset and all four selected model checkpoints.

### Evaluation protocol

- Test split: the same 64 images for all four models.
- Common evaluation resolution: \(256 \times 256\).
- YOLO26 inference resolution: \(640 \times 640\), followed by resizing to the common evaluation grid.
- Reference targets and corrected validity masks are shared across models.
- Invalid pixels are excluded from MAE and RMSE.
- MiDaS and both Depth Anything models use their saved affine calibration parameters.
- YOLO26 output is compared directly with the normalized target.
- No test-set calibration or parameter fitting is performed.

**Metrics**
- Pixel-weighted MAE and RMSE aggregate errors over all valid test pixels.
- Image-wise mean and median MAE/RMSE give each test image equal weight.
- Parameter count and checkpoint file size describe model size.
- Inference latency is measured after five warm-up iterations. The timed pipeline includes model-specific preprocessing, input transfer, inference, postprocessing, prediction transfer to CPU, and resizing to \(256 \times 256\). Disk loading, metric calculation, and output saving are excluded. Measurements were collected on an NVIDIA Tesla T4 GPU at batch size one.

Because the model input resolutions and preprocessing differ, latency should be interpreted as approximate pipeline latency under the selected configurations, not as a strict same-resolution benchmark.

## Setup

The notebooks were developed for a Kaggle GPU environment (NVIDIA Tesla T4). They expect the prepared dataset and trained checkpoints to be attached as notebook inputs or placed at paths configured near the beginning of each notebook.

### 1. Clone the repository

```bash
git clone https://github.com/<YOUR-USERNAME>/<YOUR-REPOSITORY>.git
cd <YOUR-REPOSITORY>
```

### 2. Create an environment

```bash
python -m venv .venv
```

Activate it:

**Windows**
```powershell
.venv\Scripts\activate
```

**Linux/macOS**
```bash
source .venv/bin/activate
```

### 3. Install PyTorch

Install a PyTorch build compatible with your operating system and CUDA version by following the official selector:

[https://pytorch.org/get-started/locally/](https://pytorch.org/get-started/locally/)

Install PyTorch before the remaining packages so that you can select the correct CUDA-enabled build. The notebooks were run with GPU acceleration; CPU execution may be substantially slower and may require smaller batch sizes.

### 4. Install the remaining dependencies

```bash
python -m pip install -r requirements.txt
```

### 5. Run the notebooks

Open the notebooks in Jupyter or upload them to Kaggle. Configure dataset and checkpoint paths in the path-configuration cells. The preparation notebook must be run first; model training requires its outputs. The final comparison requires all four selected checkpoints and the same corrected preparation manifest.

## Reproducibility notes

- Use the corrected preparation manifest for every model; do not create a new split separately in each training notebook.
- Select model checkpoints using validation data only.
- Keep the test partition untouched until all model choices and training decisions are finalized.
- Preserve the training-derived normalization limits (16 and 173) when reproducing this experiment.
- Keep the same validity-mask protocol and evaluation grid across all four models.
- Record software versions, checkpoint identifiers, training settings, and hardware when rerunning the experiments.

The exported results archive may contain selected prediction previews and arrays for qualitative inspection. These are not a substitute for the complete per-image metric tables or the trained model checkpoints.

## Limitations

1. The scanner-derived grayscale targets do not have a verified physical depth scale.
2. The validity masks are automated/corrected heuristic masks, not independently verified geometric ground truth.
3. The test set contains 64 images.
4. Model training configurations and input resolutions differ.
5. Inference timings are approximate and reflect the complete model-specific inference pipelines.

Consequently, the results support a comparison of agreement with the prepared normalized scanner-intensity targets and computational requirements. They do not establish physical pothole-depth measurement accuracy. Further validation requires metrically calibrated depth ground truth, independently verified masks, and a larger and more diverse road dataset.

## References

- Pothole440 dataset: [Youngjincha/Pothole440](https://github.com/Youngjincha/Pothole440)
- MiDaS: [Ranftl et al., MiDaS v3.1](https://arxiv.org/abs/2307.14460)
- Depth Anything V1: [Yang et al., 2024](https://arxiv.org/abs/2401.10891)
- Depth Anything V2: [Yang et al., 2024](https://arxiv.org/abs/2406.09414)
- Ultralytics depth task: [Monocular Depth Estimation with Ultralytics YOLO](https://docs.ultralytics.com/tasks/depth)

---

**Author:** Siddharth Kumar  
**Affiliation:** Department of Electrical Engineering, Indian Institute of Technology Bombay  
**Seminar:** EE694
