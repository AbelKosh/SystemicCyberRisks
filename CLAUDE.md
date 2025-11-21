# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository evaluates systemic cyber risks using vulnerability data from MITRE CVE and NIST NVD databases. The goal is to predict which vulnerabilities are likely to be exploited in the wild using machine learning models trained on CVSS scores and other technical characteristics.

**Key Dataset**: CISA Known Exploited Vulnerabilities (KEV) catalog serves as ground truth labels for supervised learning.

## Common Commands

### Embedding-Based Prediction Pipeline

Predict exploitation using text embeddings of vulnerability descriptions:

```bash
# Full pipeline with Sentence Transformers (local, no API needed)
./embedding_pipeline/run_full_pipeline.sh sentencetransformer

# OR: Full pipeline with OpenAI embeddings (requires API key in .env)
./embedding_pipeline/run_full_pipeline.sh openai

# OR: Run steps individually
python embedding_pipeline/1_extract_descriptions.py
python embedding_pipeline/2_preprocess_text.py
python embedding_pipeline/3b_generate_embeddings_sentencetransformer.py
python embedding_pipeline/4_train_models.py --embedding-type sentencetransformer
```

**Note**: Embedding generation takes 2-3 hours for 330K descriptions on Apple Silicon. Results include both Logistic Regression and Random Forest models with full performance comparison.

**Achieved Performance**:
- Logistic Regression: ROC-AUC 0.872, Average Precision 0.023
- Random Forest: ROC-AUC 0.888, Average Precision 0.058

### Hybrid Model (CVSS + Embeddings) - Best Performance

Combine CVSS features with text embeddings for superior prediction:

```bash
# Train hybrid model (requires both CVSS data and embeddings)
python embedding_pipeline/run_hybrid_model.py
```

**Achieved Performance**:
- **Hybrid Model: ROC-AUC 0.912** (best overall)
- CVSS-only: ROC-AUC 0.846
- Embedding-only: ROC-AUC 0.864

The hybrid approach concatenates 40 CVSS features with 384-dimensional embeddings for 424 total features, achieving a 6.5% improvement over CVSS-only models.

### Data Generation Pipeline

Generate the raw vulnerability dataset from CVE JSON files and NIST NVD API:

```bash
cd generate_data
python create_vulnerabilities_dataset.py
```

This orchestrates three sequential steps:
1. `read_nvd_api.py` - Queries NIST NVD API (takes several hours due to rate limits)
2. `merge_files.py` - Combines annual vulnerability files
3. `pull_description.py` - Extracts vulnerability descriptions

**Note**: Requires CVE data in `cves/YYYY/XXXX/CVE-YYYY-XXXX.json` format. Download from https://www.cve.org/Downloads or UNC Longleaf server.

### Baseline Modeling (Quick Start)

Run the complete data cleaning and baseline modeling pipeline:

```bash
python modeling/baseline_abel_koshy_07_25.py
```

Produces:
- `data/data.csv` - Cleaned dataset
- `roc_curve.png` - ROC curve visualization
- Console output with model metrics

### Interactive Analysis with Jupyter Notebooks

For detailed exploratory analysis and advanced modeling:

```bash
# Start Jupyter and run notebooks in sequence:
jupyter notebook notebooks/Baseline_Model/0_Data_Loading_and_EDA.ipynb
jupyter notebook notebooks/Baseline_Model/1_Data_Preprocessing.ipynb
jupyter notebook notebooks/Baseline_Model/2_Baseline_Modeling.ipynb
jupyter notebook notebooks/Baseline_Model/3_Advanced_Modeling.ipynb
jupyter notebook notebooks/Baseline_Model/4_Cross_Validation.ipynb
jupyter notebook notebooks/Baseline_Model/5_Hybrid_Model_CVSS_Plus_Embeddings.ipynb
```

### Environment Setup

```bash
# Create and activate virtual environment
python -m venv venv
source venv/bin/activate  # On macOS/Linux
# venv\Scripts\activate   # On Windows

# Install core dependencies
pip install requests pandas numpy matplotlib scikit-learn seaborn jupyter

# For embedding pipeline (choose one or both):
pip install sentence-transformers torch  # Local embeddings (recommended)
pip install openai python-dotenv          # OpenAI API embeddings (optional)

# For NLP preprocessing
pip install nltk

# Download NLTK data (run in Python)
python -c "import nltk; nltk.download('stopwords'); nltk.download('punkt')"
```

## Architecture and Data Flow

### Three Complementary Approaches

**1. CVSS Feature-Based (Baseline)**
Uses structured vulnerability metrics (severity scores, attack vectors, etc.)
- ROC-AUC: ~0.85
- Features: ~40 CVSS metrics
- Interpretable feature importances

**2. Embedding-Based**
Uses NLP embeddings of vulnerability descriptions for prediction
- ROC-AUC: ~0.89 (Random Forest)
- Features: 384-dimensional text embeddings
- Captures semantic vulnerability patterns

**3. Hybrid Model (Best Performance)**
Combines CVSS features + text embeddings
- ROC-AUC: ~0.91
- Features: 424 total (40 CVSS + 384 embeddings)
- Leverages both structured and unstructured data

All approaches predict the same target: CISA Known Exploited Vulnerabilities.

### Three-Stage CVSS Pipeline Architecture

```
┌─────────────────┐
│ 1. Data Gen     │  CVE JSON files → NVD API → vulnerabilities.csv.gz
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 2. Cleaning     │  vulnerabilities.csv.gz + KEV catalog → data.csv
└────────┬────────┘         (filters post-2015, creates target labels)
         │
         ▼
┌─────────────────┐
│ 3. Modeling     │  data.csv → feature engineering → trained models
└─────────────────┘
```

### Critical Design Decisions

**Year Filtering**: Dataset is filtered to post-2015 vulnerabilities because:
- CVSS v3.0 was released in 2015; earlier data uses inconsistent v2.0 scoring
- Pre-2016 data has ~26% missing values in key categorical fields
- Modern threat landscape is more relevant for current predictions

**Target Variable Creation**: Binary classification where `target=1` means the CVE appears in CISA's KEV catalog (known exploited). This creates severe class imbalance (~0.6% positive class) that accurately reflects real-world exploitation rates.

**Feature Engineering Strategy**:
- Binary encoding: Scores ≥7 encoded as high-risk (1), else low-risk (0)
- Categorical encoding: One-hot encoding with `drop='first'` to avoid dummy variable trap
- Scaling: MinMaxScaler applied to `numScores` and `agreement` columns
- Dropped features: Version 4.0 CVEs, environmental/temporal CVSS metrics (too sparse), and raw text descriptions

### Embedding Pipeline Architecture (NEW)

```
embedding_pipeline/
├── 1. Extract Descriptions    → Raw CVE descriptions + KEV labels
├── 2. Preprocess Text          → NLP cleaning (stop words, URLs, etc.)
├── 3a. OpenAI Embeddings       → text-embedding-3-small (1536-dim)
├── 3b. Sentence Transformers   → all-MiniLM-L6-v2 (384-dim, local)
└── 4. Train Models             → Logistic Reg + Random Forest
                                  Train: 2015-2024, Test: 2025
```

**Key Differences from CVSS Approach:**
- Input: Unstructured text vs structured features
- Features: 384/1536 embedding dims vs ~40 CVSS features
- Interpretability: Black box vs feature importances
- Computation: Slower (embedding gen) vs fast
- Achieved ROC-AUC: 0.89 (RF) vs 0.85 (CVSS-only)

**Hybrid Pipeline**: For best results, combine both approaches using `run_hybrid_model.py` (achieves 0.91 ROC-AUC).

See `embedding_pipeline/README.md` for complete documentation.

### Key Scripts

**embedding_pipeline/**
- `1_extract_descriptions.py`: Extract descriptions + KEV labels, filter 2015-2025
- `2_preprocess_text.py`: Comprehensive NLP cleaning (stop words, URLs, versions)
- `3a_generate_embeddings_openai.py`: OpenAI API embeddings (requires .env with API key)
- `3b_generate_embeddings_sentencetransformer.py`: Local embeddings (no API needed)
- `4_train_models.py`: Train both LR and RF, temporal split, generate visualizations
- `run_full_pipeline.sh`: Master script to run complete pipeline
- `run_hybrid_model.py`: Train hybrid model combining CVSS + embeddings (best performance)
- `generate_confusion_matrix.py`: Generate confusion matrix visualizations from predictions

**generate_data/create_vulnerabilities_dataset.py**
- Orchestrates the three-step data collection pipeline
- Sequential execution with error handling between steps
- Progress tracking and timing information

**modeling/baseline_abel_koshy_07_25.py**
- Combined data cleaning + logistic regression baseline (single script execution)
- Handles both `.csv` and `.csv.gz` input formats
- Uses `class_weight='balanced'` to handle class imbalance
- Produces ROC curve visualization and classification report

**notebooks/Baseline_Model/** (6 notebooks in sequence)
- `0_Data_Loading_and_EDA.ipynb`: Load raw data, filter to 2016+, create target variable, temporal analysis
- `1_Data_Preprocessing.ipynb`: Feature engineering, encoding, scaling
- `2_Baseline_Modeling.ipynb`: Logistic regression baseline with class imbalance handling
- `3_Advanced_Modeling.ipynb`: Random Forest, Gradient Boosting, ensemble methods
- `4_Cross_Validation.ipynb`: Stratified k-fold validation, hyperparameter tuning
- `5_Hybrid_Model_CVSS_Plus_Embeddings.ipynb`: Hybrid model combining CVSS + embeddings, performance comparison

**keyword_generator.py** (standalone utility)
- RAG-based keyword extraction using OpenAI embeddings + FAISS
- Requires `MITRE Tactics.docx` and `descriptions_only.csv` as inputs
- Uses MITRE ATT&CK framework to ground keyword extraction
- Outputs `extracted_keywords.csv`

## Data Files

### Input Data (Required)

- `cves/YYYY/XXXX/CVE-YYYY-XXXX.json` - Raw CVE records from MITRE
- `data/known_exploited_vulnerabilities.csv` - CISA KEV catalog (target labels)
- `.env` - Environment variables (OPENAI_API_KEY) - copy from .env.template

### Generated Data

- `data/vulnerabilities.csv.gz` - Complete vulnerability dataset from generate_data pipeline
- `data/data.csv` - Cleaned dataset ready for modeling (from baseline_model script)
- `data/processed_vulnerabilities.csv` - Intermediate output from notebook 0 (filtered to 2016+)
- `data/vulnerabilities_YYYY.csv` - Annual vulnerability files (intermediate from pipeline)

### Embedding Pipeline Data (NEW)

- `embedding_pipeline/data/descriptions_with_labels.csv` - Extracted descriptions + targets
- `embedding_pipeline/data/descriptions_preprocessed.csv` - Cleaned text for embedding
- `embedding_pipeline/data/embeddings_*.npz` - Compressed embedding arrays
- `embedding_pipeline/data/metadata_*.csv` - CVE IDs, years, targets

### Output Files

- `roc_curve.png` - Model performance visualization (from baseline_model script)
- `extracted_keywords.csv` - RAG-extracted keywords (from keyword_generator.py)
- `embedding_pipeline/results/results_*.json` - Detailed model metrics (LR + RF)
- `embedding_pipeline/results/predictions_*.csv` - Test set predictions with probabilities
- `embedding_pipeline/visualizations/*.png` - ROC curves, PR curves, confusion matrices, comparisons
- `embedding_pipeline/hybrid_results/hybrid_model_results.json` - Hybrid model performance metrics
- `embedding_pipeline/hybrid_results/models/*.pkl` - Saved model files (CVSS-only, embedding-only, hybrid)
- `embedding_pipeline/visualizations/hybrid_*.png` - Hybrid model performance visualizations

## Class Imbalance Handling

The extreme class imbalance (~205:1 ratio) requires specialized techniques:

1. **Class weighting**: Use `class_weight='balanced'` in scikit-learn models
2. **Stratified splitting**: Always use `stratify=y` in train_test_split
3. **Evaluation metrics**: Focus on ROC-AUC, precision-recall curves, not accuracy
4. **Threshold tuning**: Optimize decision threshold for recall vs precision trade-off

## Model Performance Summary

All models demonstrate that technical vulnerability characteristics have strong predictive power for exploitation likelihood:

| Approach | ROC-AUC | Avg Precision | Key Features |
|----------|---------|---------------|--------------|
| **CVSS-Only** | 0.846 | 0.045 | 40 CVSS metrics, interpretable |
| **Embedding-Only (LR)** | 0.872 | 0.023 | 384-dim text embeddings |
| **Embedding-Only (RF)** | 0.888 | 0.058 | 384-dim text embeddings |
| **Hybrid Model** | **0.912** | **0.123** | 424 features (CVSS + embeddings) |

**Key Findings**:
- Text embeddings alone outperform CVSS features (+4.2% ROC-AUC)
- Hybrid approach achieves best performance (+6.6% over CVSS-only)
- Both structured (CVSS) and unstructured (descriptions) data contain complementary predictive signals
- Random Forest outperforms Logistic Regression on embeddings due to non-linear patterns in text

## Important Notes

- The NVD API step (`read_nvd_api.py`) respects rate limits (50 requests per 30 seconds) and may take several hours
- API key is included in `read_nvd_api.py` for higher rate limits
- All scripts in `generate_data/` are designed to run from that subdirectory (use relative paths like `../data/`)
- CVSS version 4.0 records are filtered out as they are inconsistent or erroneous
- Duplicate CVE IDs are dropped (keeping first occurrence) to prevent training bias
- The `year` column is extracted from CVE ID format: `CVE-YYYY-NNNNN`
- Missing values in categorical features (attackVector, attackComplexity, etc.) are typically from pre-2016 CVSS v2.0 records

## RAG Ingestion Material

The `RAG Ingestion Material/` directory contains MITRE ATT&CK tactics in YAML format (TA001-TA040). These are used by `keyword_generator.py` for grounded keyword extraction from vulnerability descriptions.

## Complete Workflow Guide

### For First-Time Setup

1. **Clone repository and set up environment**:
   ```bash
   git clone <repository-url>
   cd SystemicCyberRisks
   python -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt  # Or follow Environment Setup above
   ```

2. **Obtain required data** (if not already present):
   - Download CVE JSON files from https://www.cve.org/Downloads
   - Place in `cves/YYYY/XXXX/CVE-YYYY-XXXX.json` structure
   - Or use existing `data/vulnerabilities.csv.gz` if available

3. **Generate dataset** (if starting from raw CVE files):
   ```bash
   cd generate_data
   python create_vulnerabilities_dataset.py
   cd ..
   ```

### For Quick Baseline Modeling

```bash
# Single-command baseline model
python modeling/baseline_abel_koshy_07_25.py
# Outputs: data/data.csv, roc_curve.png
```

### For Advanced Embedding-Based Modeling

```bash
# Complete embedding pipeline (local, no API needed)
./embedding_pipeline/run_full_pipeline.sh sentencetransformer

# Or for highest quality (requires OpenAI API key)
# First: Copy .env.template to .env and add your OPENAI_API_KEY
./embedding_pipeline/run_full_pipeline.sh openai
```

### For Best Performance (Hybrid Model)

```bash
# Prerequisites: Run both CVSS baseline and embedding pipeline first
python modeling/baseline_abel_koshy_07_25.py  # Generates CVSS data
./embedding_pipeline/run_full_pipeline.sh sentencetransformer  # Generates embeddings

# Then run hybrid model
python embedding_pipeline/run_hybrid_model.py
# Outputs: embedding_pipeline/hybrid_results/
```

### For Interactive Exploration

```bash
jupyter notebook
# Navigate to notebooks/Baseline_Model/ and run in sequence (0-5)
```

## Development Best Practices

When working with this codebase:

1. **Data Paths**: Always use relative paths from script location (e.g., `../data/` from generate_data/)
2. **Class Imbalance**: Always use `class_weight='balanced'` for classifiers
3. **Train/Test Splits**: Use temporal splits (2015-2024 train, 2025 test) or stratified random splits
4. **Evaluation Metrics**: Focus on ROC-AUC and Average Precision, not accuracy
5. **Version Filtering**: Filter out CVSS v4.0 records (inconsistent) and pre-2015 data (missing values)
6. **Reproducibility**: Set random seeds (e.g., `random_state=42`) in all sklearn functions
7. **Memory Management**: Use `.csv.gz` compression for large datasets
8. **Embeddings**: Prefer local Sentence Transformers for cost/speed; use OpenAI for quality
9. **Model Saving**: Save trained models as `.pkl` files in appropriate results directories
10. **Visualization**: Always generate ROC curves, PR curves, and confusion matrices for evaluation

## Troubleshooting

**Common Issues**:

- **"File not found" errors**: Check that you're running scripts from correct directory
- **Import errors**: Ensure all dependencies installed (check Environment Setup section)
- **NLTK data errors**: Run `nltk.download('stopwords')` and `nltk.download('punkt')`
- **Memory errors**: Use embeddings in batches or reduce dataset size for testing
- **API rate limits**: read_nvd_api.py includes built-in rate limiting; be patient
- **GPU not found**: Sentence Transformers will fall back to CPU (slower but works)
- **Missing .env file**: Copy .env.template and add your OPENAI_API_KEY if using OpenAI embeddings

## Repository Statistics

- **Total CVEs**: 330,000+ (1999-2025)
- **Known Exploited**: ~2,045 (0.6% positive class)
- **Training Samples**: ~317K (2015-2024)
- **Test Samples**: ~13K (2025)
- **Best Model**: Hybrid (ROC-AUC 0.912)
- **Feature Count**: 424 (40 CVSS + 384 embeddings)
