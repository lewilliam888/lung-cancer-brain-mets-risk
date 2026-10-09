# Lung Cancer Brain Metastasis Risk

This project predicts which patients with stage IV non-small cell lung cancer (NSCLC) will develop brain or central nervous system (CNS) metastases within 2 years, among patients whose brain imaging was clear at diagnosis. A risk score is trained on tumor genomics and clinical data from MSK MetTropism, then evaluated against time-to-event outcomes in an external cohort from AACR Project GENIE.

## Motivation

CNS metastases are often found only after symptoms such as sudden weakness or trouble walking appear. A risk estimate made when the tumor is first sequenced could identify the patients who need regular brain MRI, so spread is detected earlier. The aim is earlier detection.

## Data

| Cohort | Role | Size | Access |
| --- | --- | --- | --- |
| [MSK MetTropism](https://www.cbioportal.org/study/summary?id=msk_met_2021) | Training and cross-validation | 25,775 samples across 50 tumor types (metastatic NSCLC subset used) | Free on cBioPortal |
| [GENIE BPC NSCLC v2.0-public](https://aacrprojectgenie.org/data/bpc-nsclc-v2-0-public/) | External validation | 1,846 patients from 4 institutions | Synapse account and data use agreement |

Only GENIE patients from DFCI, VICC and UHN are used for validation. MSKCC patients appear in both cohorts and are excluded from the test set.

No data is committed to this repository. See [Setup](#setup) to download it.

## Approach

**Training cohort.** Metastatic NSCLC patients in MSK MetTropism (`IS_DIST_MET_MAPPED`), one tumor sample each. These patients routinely receive brain MRI at staging, which makes the CNS label (`DMETS_DX_CNS_BRAIN`) more reliable. Samples taken from a CNS site are dropped because they reveal the label. Follow-up time is included as a covariate.

**Test cohort.** Stage IV GENIE patients whose first brain imaging after diagnosis shows no CNS involvement. The event is the first imaging report showing CNS involvement, measured from diagnosis. Death is a competing event, and other patients are censored at last follow-up. Stage I to III patients form a secondary analysis.

**Features.** Driver alterations (EGFR, ALK, ROS1, RET, MET, KRAS, BRAF, ERBB2, TP53, STK11, KEAP1, SMARCA4, CDKN2A), tumor mutational burden, fraction of genome altered, age, sex and histologic subtype. Only genes covered by every sequencing panel in both cohorts are kept.

**Models.**
1. Baseline: logistic regression on subtype, age and sex
2. Baseline plus EGFR and ALK status
3. Penalized logistic regression on all features
4. Gradient-boosted trees (XGBoost)

**Evaluation.** Models are compared with nested cross-validation on MSK, and the final score is frozen before GENIE is opened. On GENIE, discrimination is time-dependent AUC at 1 and 2 years under competing risks, with 95% bootstrap confidence intervals. The score is recalibrated to 2-year cumulative incidence with a cross-validated Fine-Gray model, followed by a calibration plot and a decision curve for MRI surveillance. SHAP values are used for interpretation.

Reporting follows the [TRIPOD+AI](https://www.tripod-statement.org/) guideline.

## Project status

- [ ] Phase 1: Data download and cohort description
- [ ] Phase 2: Baseline models and evaluation code
- [ ] Phase 3: Model development on MSK and freezing the score
- [ ] Phase 4: Time-to-event validation on GENIE
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
- The training label records CNS metastasis at any time, while the test outcome is new CNS metastasis within 2 years. Recalibration in GENIE narrows this gap.
- The test set only includes patients with documented baseline brain imaging, which limits its size.
- MSK MetTropism does not include smoking history or stage at diagnosis.
- Both cohorts come from US and Canadian academic centers and may not represent other populations.

## References

- Nguyen B, et al. Genomic characterization of metastatic patterns from prospective clinical sequencing of 25,000 patients. *Cell*. 2022.
- AACR Project GENIE Consortium. [GENIE BPC NSCLC v2.0-public](https://aacrprojectgenie.org/data/bpc-nsclc-v2-0-public/).
- Collins GS, et al. TRIPOD+AI statement. *BMJ*. 2024.
