# Lung Cancer Brain Metastasis Risk

This project predicts which patients with non-small cell lung cancer (NSCLC) will develop brain or central nervous system (CNS) metastases, using tumor genomics and basic clinical data. The model is trained on MSK MetTropism and validated on an external cohort from AACR Project GENIE.

## Motivation

Brain and leptomeningeal metastases are often found only after symptoms appear, when treatment options are narrower. A risk estimate made when the tumor is first sequenced could help decide which patients get regular brain MRI and which can safely skip it.

## Data

| Cohort | Role | Size | Access |
| --- | --- | --- | --- |
| [MSK MetTropism](https://www.cbioportal.org/study/summary?id=msk_met_2021) | Training and cross-validation | 25,775 samples across 50 tumor types (NSCLC subset used) | Free on cBioPortal |
| [GENIE BPC NSCLC v2.0-public](https://aacrprojectgenie.org/data/bpc-nsclc-v2-0-public/) | External validation | 1,846 patients from 4 institutions | Synapse account and data use agreement |

The outcome is recorded CNS/brain metastasis (`DMETS_DX_CNS_BRAIN`). For external validation, only GENIE patients from DFCI, VICC and UHN are used. MSKCC patients appear in both cohorts and are excluded from the test set.

No data is committed to this repository. See [Setup](#setup) to download it.

## Approach

**Cohort.** One tumor sample per NSCLC patient. Samples taken from a CNS site are dropped because they reveal the label.

**Features.** Driver alterations (EGFR, ALK, ROS1, RET, MET, KRAS, BRAF, ERBB2, TP53, STK11, KEAP1, SMARCA4, CDKN2A), tumor mutational burden, fraction of genome altered, age, sex, histologic subtype and sample site. Only genes covered by every sequencing panel in both cohorts are kept.

**Models.**
1. Baseline: logistic regression on subtype, age and sex
2. Baseline plus EGFR and ALK status
3. Penalized logistic regression on all features
4. Gradient-boosted trees (XGBoost)

**Evaluation.** Nested cross-validation on MSK, then a single run on the frozen model against GENIE. Metrics are AUROC and AUPRC with 95% bootstrap confidence intervals, calibration plots, Brier score and decision curve analysis. Results are also reported by subtype, sex and EGFR status. SHAP values are used for interpretation.

Reporting follows the [TRIPOD+AI](https://www.tripod-statement.org/) guideline.

## Project status

- [ ] Phase 1: Data download and cohort description
- [ ] Phase 2: Baseline models and evaluation code
- [ ] Phase 3: Model development on MSK
- [ ] Phase 4: External validation on GENIE
- [ ] Phase 5: Interpretation, report and demo

## Repository structure

```
lung-cancer-brain-mets-risk/
├── data/
│   ├── raw/            # downloaded files, git-ignored
│   └── processed/      # cleaned tables, git-ignored
├── notebooks/          # exploration and figures
├── src/                # data loading, features, models, evaluation
├── results/            # metrics and plots
├── environment.yml
└── README.md
```

## Setup

```bash
git clone https://github.com/<your-username>/lung-cancer-brain-mets-risk.git
cd lung-cancer-brain-mets-risk
conda env create -f environment.yml
conda activate brain-mets
```

**MSK MetTropism.** Download the study from its [cBioPortal page](https://www.cbioportal.org/study/summary?id=msk_met_2021) and extract it to `data/raw/msk_met_2021/`.

**GENIE BPC NSCLC.** Create a [Synapse](https://www.synapse.org) account, request access to [syn27056697](https://www.synapse.org/Synapse:syn27056697) and accept the data use terms. Then create a personal access token and run:

```bash
export SYNAPSE_AUTH_TOKEN=<your-token>
python -c "import os, synapseclient, synapseutils; syn = synapseclient.login(authToken=os.environ['SYNAPSE_AUTH_TOKEN']); synapseutils.syncFromSynapse(syn, 'syn27056697', path='data/raw/genie_nsclc')"
```

## Limitations

- The CNS label combines brain parenchymal and leptomeningeal metastases, so the model cannot predict leptomeningeal spread on its own.
- Metastases are only recorded if they were imaged and documented, so some cases are likely missed.
- Patients who live longer have more time to develop CNS metastases. Follow-up time is adjusted for, but some bias may remain.
- MSK MetTropism does not include smoking history or stage at diagnosis.
- Both cohorts come from US and Canadian academic centers and may not represent other populations.

## References

- Nguyen B, et al. Genomic characterization of metastatic patterns from prospective clinical sequencing of 25,000 patients. *Cell*. 2022.
- AACR Project GENIE Consortium. [GENIE BPC NSCLC v2.0-public](https://aacrprojectgenie.org/data/bpc-nsclc-v2-0-public/).
- Collins GS, et al. TRIPOD+AI statement. *BMJ*. 2024.
