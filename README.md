[![Python](https://img.shields.io/badge/Python-3.8%2B-blue)](https://www.python.org/)
[![TensorFlow](https://img.shields.io/badge/TensorFlow-2.x-orange)](https://www.tensorflow.org/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

---

## 📌 Overview

This repository contains the complete source code, datasets, and trained model files for the paper:

**GAC-BiT-CNN: Methyltransferase Prediction from Protein Language Model Embeddings using a Hybrid Generative Adversarial Capsule, Bidirectional Transformer and Convolutional Network**


GAC-BiT-CNN is a novel hybrid deep learning architecture that integrates:
- **G**enerative **A**dversarial **C**apsule Networks (GAC)
- **Bi**directional **T**ransformers (BiT)
- **C**onvolutional **N**eural **N**etworks (CNN)

Four protein language model embeddings are evaluated individually:
- **ProtGen** (1536-dim)
- **ESM-2** (1280-dim)
- **ProtT5-BFD** (1024-dim)
- **ProtT5-XL-UniRef50** (1024-dim)

The best model uses the **fused** feature set: **ProtT5-BFD + ProtT5-XL** concatenated (**2048-dim**).

---

## 📊 Results Summary

| Dataset                          | Accuracy | Sensitivity | Specificity | AUC    | MCC   |
|-----------------------------------|----------|-------------|-------------|--------|-------|
| Training (5-Fold CV)              | 98.34%   | 98.39%      | 98.29%      | 99.78% | 0.967 |
| Test Dataset (held-out)           | 95.50%   | 95.23%      | 95.76%      | 98.70% | 0.901 |
| Independent (Blind) Dataset       | 93.00%   | 89.41%      | 96.60%      | 97.51% | 0.862 |

All results use the fused (ProtT5-BFD + ProtT5-XL) feature set.

Performance decreases progressively from training → test → independent evaluation, reflecting a realistic and expected generalisation gradient rather than overfitting to the training distribution. Note: the 300 negatives of the independent dataset are identical to benchmark sequences, so independent-set specificity is optimistic (see Section 2.1.5 of the paper).

---

## 📁 Repository Structure

```
GAC-BiT-CNN-MT/
│
├── data/
│   ├── data_training.txt          # Training dataset (FASTA format) — 2290 MT + 2290 non-MT
│   ├── data_testing.txt           # Held-out test dataset (FASTA format) — 572 MT + 572 non-MT
│   └── external_dataset.txt       # Independent (blind) dataset (FASTA format) — 300 MT + 300 non-MT
│
├── code/
│   ├── feature_extraction.py      # Feature extraction using ProtGen, ESM-2, ProtT5-BFD, ProtT5-XL
│   ├── GAC_BiT_CNN_Training.py    # Model training with 5-fold cross-validation
│   ├── GAC_BiT_CNN_Testing.py     # Test set and independent (blind) dataset evaluation
│   └── fuse_features.py           # Fuse ProtT5-BFD + ProtT5-XL embeddings into one feature set
│
├── results/
│   └── sample_output.txt          # Sample output metrics from 5-fold cross-validation
│
├── README.md                      # This file
├── requirements.txt               # Python dependencies
└── LICENSE                        # MIT License
```

---

## 🗂️ Dataset Description

All datasets are provided in **FASTA format**.

| File | Positives (MT) | Negatives (non-MT) | Total |
|------|----------------|---------------------|-------|
| `data/data_training.txt` | 2,290 | 2,290 | 4,580 |
| `data/data_testing.txt`  | 572   | 572   | 1,144 |
| `data/external_dataset.txt` (blind/independent) | 300 | 300 | 600 |

**Dataset Construction:** the benchmark datasets of PMTPred (Yadav et al., 2024) are used as released by the original authors.
- Positive sequences retrieved from NCBI using query "Methyltransferase"
- Negative sequences retrieved from NCBI (non-methyltransferase proteins)
- Redundancy removed using **CD-HIT** at **40% sequence identity threshold**
- Sequences shorter than 50 amino acids were excluded
- The held-out test dataset is drawn from the same overall sequence collection as training (80:20 split); only 4 of the 1,144 test sequences share ≥40% identity with a training sequence
- The independent dataset contains 300 Swiss-Prot methyltransferases with known 3D structures and 300 non-methyltransferases; it was never used for training, validation or hyperparameter tuning. Its 300 negatives are identical to benchmark sequences, and 87 positives share ≥40% identity with training sequences

---

## ⚙️ Installation

### 1. Clone the repository
```bash
git clone https://github.com/MuhammadMuazzam/GAT-BiT-CNN.git
cd GAT-BiT-CNN
```

### 2. Create a virtual environment (recommended)
```bash
python -m venv venv
source venv/bin/activate        # Linux/Mac
venv\Scripts\activate           # Windows
```

### 3. Install dependencies
```bash
pip install -r requirements.txt
```

---

## 🚀 Usage

### Step 1 — Feature Extraction

Extract embeddings from protein sequences using all four protein language models:

```bash
python code/feature_extraction.py
```

This will generate four CSV files:
- `ProtGen_embeddings.csv` (1536 features)
- `ESM2_embeddings.csv` (1280 features)
- `ProtT5_BFD_embeddings.csv` (1024 features)
- `ProtT5_XL_UNIREF50_embeddings.csv` (1024 features)

### Step 2 — Fuse Feature Sets

Combine the ProtT5-BFD and ProtT5-XL embeddings into one fused feature vector (2048-dim):

```bash
python code/fuse_features.py
```

### Step 3 — Train the Model

Train GAC-BiT-CNN using 5-fold cross-validation:

```bash
python code/GAC_BiT_CNN_Training.py
```

The trained model will be saved as:
- `GAC_BiTCN.keras` — Full model
- `GAC_BiTCN.weights.h5` — Weights only
- `GAC_BiTCN.json` — Model architecture

### Step 4 — Evaluate on Test and Independent (Blind) Datasets

```bash
python code/GAC_BiT_CNN_Testing.py
```

---

## 🧬 Model Architecture

```
Input (Fused Embeddings: ProtT5-BFD + ProtT5-XL, 2048-dim)
         │
     CNN Branch
 (Multi-scale residual TCN)
  kernel: 3,5,7 / dilation: 1,2,3
         │
    ┌────┼──────────────────────────┐
    │    │                          │
    │  BiT Branch                GAC Branch
    │  (Bidirectional            (Capsule Network with
    │   Transformer +             Gaussian noise and
    │   BiLSTM)                   Dynamic Routing)
    │    │                          │
    └────┴──────────────────────────┘
                              │
                     Concatenate Features
                              │
                     Dense (128) + Dropout(0.3)
                              │
                     Sigmoid Output (Binary)
```

### Hyperparameters

| Component | Parameter | Value |
|-----------|-----------|-------|
| Capsule Network | Primary Capsule Dim | 8 |
| Capsule Network | Num Capsules | 4 |
| Capsule Network | Routing Iterations | 3 |
| Capsule Network | Gaussian noise (std) | 0.01 |
| CNN Branch | Kernel Sizes | (3, 5, 7) |
| CNN Branch | Dilation Rates | (1, 2, 3) |
| CNN Branch | Filters | 64 |
| BiT Branch | Attention Heads | 2 |
| BiT Branch | Key Dimension | 32 |
| BiT Branch | LSTM Units | 64 |
| Training | Batch Size | 32 |
| Training | Learning Rate | 1e-4 |
| Classifier | Dense / Dropout | 128 / 0.3 |
| Training | Max Epochs | 100 (early stopping, patience 10) |
| Training | LR schedule | ReduceLROnPlateau (factor 0.5, patience 5) |

---

## 📦 Protein Language Models Used

| Model | Embedding Dim | Pretraining Database |
|-------|--------------|---------------------|
| ProtGen (progen2-medium) | 1536 | UniRef90 + BFD (280M sequences) |
| ESM-2 (esm2_t33_650M_UR50D) | 1280 | UniRef90 (250M sequences) |
| ProtT5-BFD (prot_t5_xl_bfd) | 1024 | BFD (2.1B sequences) |
| ProtT5-XL-UniRef50 (prot_t5_xl_uniref50) | 1024 | UniRef50 (curated) |
| **Fused (ProtT5-BFD + ProtT5-XL)** | **2048** | BFD + UniRef50 |

---

## 📈 Comparison with State-of-the-Art

Evaluated on the held-out test dataset:

| Method | Accuracy | Sensitivity | Specificity | AUC | MCC |
|--------|----------|-------------|-------------|-----|-----|
| PMTPred [Yadav et al., 2024] | 87.94% | 89.38% | 88.48% | 90.40% | 0.759 |
| **GAC-BiT-CNN (Proposed)** | **95.50%** | **95.23%** | **95.76%** | **98.70%** | **0.901** |

---

## 📋 Requirements

See `requirements.txt` for full dependency list. Key packages:

- Python >= 3.8
- TensorFlow >= 2.10
- PyTorch >= 1.13
- Transformers >= 4.30
- scikit-learn >= 1.0
- Biopython >= 1.79
- pandas, numpy, matplotlib

---

## 📄 Citation

If you use this code or dataset in your research, please cite:

```bibtex

}
```

---

## 📬 Contact

- **Muhammad Muazzam** — muazzammuhammad043@gmail.com
- **Farman Ali** — farman335@yahoo.com | farman.buic@bahria.edu.pk

Department of Computer Science, Bahria University Islamabad, Pakistan

---

## 📝 License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.
