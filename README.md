# AF3-pipeline: Scalable AlphaFold3 Protein-Protein Interaction Analysis

This repository contains an efficient pipeline for performing **large-scale protein-protein interaction predictions and scoring using AlphaFold3**.  
All scripts are tailored for parallel execution on the [Boston University Shared Computing Cluster (BU SCC)](https://www.bu.edu/tech/support/research/computing-resources/scc/), making full use of the SGE job scheduler to handle both feature generation and inference at massive scale.  

This pipeline is **modularized to focus solely on AlphaFold3** workflows — if you are interested in initial data filtering or PPI identification (e.g., Finches IDR detection), refer to a more [complete repository](https://github.com/FilyCode/ProteinProteinInteractions_Finches-filtering_AF3-prediction).


## Overview

**AlphaFold3** enables accurate structure prediction for protein complexes, supporting in silico exploration of protein-protein interactions at atomic resolution.

This repository provides a robust framework to:
- Generate AlphaFold3 input files
- Run feature extraction/data preparation (**CPU parallel**)
- Run model inference with work stealing, locking, and fail-safe global auto-cancellation (**GPU parallel**)
- Post-process and score interactions using advanced metrics (iPTM, LIS) and produce publication-ready figures

**Data preparation, inference, and analysis are fully parallelized and resumable.** Large batches or array jobs are supported out-of-the-box for high throughput.


## Workflow Summary

1. **Prepare AlphaFold3 Input JSONs** from interacting protein sequences  
    (`01_prepare_inputs.py`)

2. **Generate Features using the AlphaFold3 Data Pipeline**  
    (`02_run_data_pipeline.sh`)  

3. **Run AF3 Inference for Ready Pairs (Self-Cleaning Job Array)**  
    (`03_run_inference.sh`)  

4. **Post-process Outputs & Score Interactions**  
    (`04_post_processing_results.ipynb`)  

All steps can be run on [BU SCC](https://www.bu.edu/tech/support/research/computing-resources/scc/) and are engineered for robust, error-proof operation in large-scale jobs.


## Directory Structure

```text
af3_inputs/                    # AlphaFold3 input JSONs (by experimental or control group)
af3_data_json/                 # AF3 data pipeline output JSONs (features)
af3_outputs/                   # Final AF3 structure outputs (.cif, .json, result folders)
af3_inference_locks/           # Lock directories for polling and job control
results/                       # Analysis results, summary CSVs, score plots
jupyter_env.yml                # Conda environment for post-processing notebooks and analysis

Example tree:
  af3_inputs/
      positive-controls/
          ENSG0001_vs_ENSG0002.json
  af3_data_json/
      positive-controls/
          ensg0001_vs_ensg0002/
              ensg0001_vs_ensg0002_data.json
  af3_outputs/
      positive-controls/
          ensg0001_vs_ensg0002/
              model.cif
              confidences.json
              summary_confidences.json
  results/
      final_results.csv
      LIS-to-iPTM-plot.svg
```

## Setup & Prerequisites

1. **AlphaFold3 Model Weights**
    - Download model weights from DeepMind.
    - Set your `MODEL_DIR` directory in all scripts to point to the weights folder.

2. **Conda Environment for Analysis (Jupyter/Matplotlib/etc)**
    - Install dependencies using the included environment:
      ```bash
      conda env create -f jupyter_env.yml
      conda activate jupyter_env
      ```
    - Register the kernel for Jupyter:
      ```bash
      python -m ipykernel install --user --name=jupyter_env
      ```

3. **SCC (SGE) Module for AlphaFold3**
    - On BU SCC, load the AlphaFold3 module:
      ```bash
      module load alphafold3/3.0.0
      ```
    - _All .sh scripts expect this to provide the `run_alphafold.sh` wrapper._

---

## Usage

### 1. Prepare AF3 Inputs

Prepare a CSV with protein pairs and sequences (see example/prototype format).  
Then run:
```bash
python 01_prepare_inputs.py
```

This will generate one JSON input per protein pair under af3_inputs/, and create a master list:

```text
json_list.txt
```

### 2. Run Data Pipeline (Features)

Run the AF3 data pipeline using an SGE array job:

```bash
NUM_TASKS=$(cat json_list.txt | wc -l)
qsub -t 1-"${NUM_TASKS}" 02_run_data_pipeline.sh
```

This will generate features for each input in parallel; completed outputs go to af3_data_json/.

    Resume logic: If a feature file already exists and is non-empty, that task is skipped automatically.

### 3. Run AlphaFold3 Inference (Structures)

Run the inference step, also as a SGE array job (with GPU allocation):

```bash
qsub -t 1-"${NUM_TASKS}" 03_run_inference.sh
```

This script:

    Dynamically polls for tasks with ready features
    Uses atomic locking to ensure exactly one job claims each pair
    Self-cancels entire array as soon as all tasks are completed
    (no wasted GPU time, no job throttling issues)

### 4. Post-processing & Scoring

After inference completes, run the analysis notebook:

```bash
# Open in Jupyter, run all cells
04_post_processing_results.ipynb
```
Extracts per-pair scores, metrics (iPTM, LIS, contacts), generates summary CSV and figures

Result files are saved to:

```text
results/final_results.csv
results/LIS-to-iPTM-plot.svg
```

Output Files

```text
af3_inputs/positive-controls/ENSG0001_vs_ENSG0002.json
af3_data_json/positive-controls/ensg0001_vs_ensg0002/ensg0001_vs_ensg0002_data.json
af3_outputs/positive-controls/ensg0001_vs_ensg0002/model.cif
results/final_results.csv
results/LIS-to-iPTM-plot.svg
```

## Authorship

This AlphaFold3 pipeline was solely developed by Philipp Trollmann during his second PhD rotation in Dr. Juan Fuxman Bass's lab at Boston University.

---

All scripts are annotated for clarity and maintainability.

---

## Citation & Contact

If you use/adapt these scripts, please:

  Cite this repository and attribute credit to the author
