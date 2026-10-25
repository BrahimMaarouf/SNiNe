# FairLend: an explainable, audited credit-risk model

**Field:** Explainable AI (XAI) · **Duration:** 4 days (~24 h) · **Level at the end:** an ML engineer who can explain, audit, and defend a model in front of a regulator, a business user, or a customer

---

## 1. The story (project idea)

A bank (fictional) wants to replace its manual loan-approval process with an ML model. Its **risk committee**, its **regulator**, and its **customers** each ask a different question:

| Stakeholder | Their question | XAI tool I'll use |
|-------------|----------------|---------------------|
| Risk committee | "What does the model rely on overall? Is it sensible?" | Global explanations: permutation importance, PDP/ICE/ALE, global SHAP |
| Loan officer | "Why was **this** applicant rejected?" | Local explanations: SHAP, LIME, reason codes |
| Rejected customer | "What can I **change** to get approved?" | Counterfactual explanations / algorithmic recourse |
| Regulator / ethics team | "Is it fair? Can you prove the explanations are reliable?" | Fairness metrics, explanation evaluation, model card |
| Data science lead | "Do we even need a black box?" | Glass-box models (EBM, scorecards) vs gradient boosting |

I build the model **and** the full explanation and audit package. Then, in one milestone, I move to **deep learning XAI** (Grad-CAM / Integrated Gradients on an image classifier) so that I cover both tabular and neural network explainability.

---

## 2. Why this project covers the field

- **Credit scoring is the textbook high-stakes XAI use case.** Under the **EU AI Act** it is classified as a **high-risk AI system (Annex III)**. Note for 2026: the "Digital Omnibus" amendment agreed in mid-2026 **pushes the Annex III high-risk obligations from 2 Aug 2026 to 2 Dec 2027**. Check the final text in the Official Journal. US lenders must give **"adverse action" reason codes**, and GDPR Art. 22 / Morocco's **Law 09-08** (CNDP) cover automated decisions about personal data.
- It covers **every major XAI family**: intrinsic vs post-hoc, global vs local, model-agnostic vs model-specific, feature attribution vs example-based (counterfactuals), and **the evaluation of explanations themselves**, which many courses skip.

---

## 3. Datasets

| Use | Dataset | Why |
|-----|---------|-----|
| **Main (tabular)** | **Default of Credit Card Clients** (UCI, Taiwan, 30k rows) | Real, medium-sized, contains **sensitive attributes** (sex, age, education, marriage) for fairness analysis |
| Alternative / stretch | Home Credit Default Risk (Kaggle) or German Credit (UCI, small) | Home Credit is much richer but takes longer |
| **Deep learning milestone** | A small image dataset: **Oxford-IIIT Pets** or **Chest X-Ray Pneumonia** (Kaggle), with a **pretrained** CNN/ViT fine-tuned briefly on **my own RTX 4080 GPU** (free) | I need a model whose explanations I can *see* |
| Stretch (NLP) | A sentiment model from Hugging Face | Bridges to project 04 |

> 💸 **The whole project costs $0 and needs no card**: free datasets, open-source tools, and my own GPU. See the zero-cost rule at the top of `01_MILESTONES.md`.

---

## 4. Stack (current as of October 2026)

| Purpose | Tool | Note |
|---------|------|------|
| Modelling | **scikit-learn**, **LightGBM** or **XGBoost** | Gradient-boosted trees are still the strongest choice for tabular data in industry |
| Glass-box models | **InterpretML (Explainable Boosting Machine)**, logistic regression scorecard, shallow decision trees | |
| Global model-agnostic | `sklearn.inspection` (permutation importance, PDP/ICE), **ALE** (via `alibi`-style implementations or PyALE; check licences) | `alibi` moved to a business-source licence in 2024, so check it before using it in commercial work |
| SHAP | **shap** (TreeSHAP, KernelSHAP, plots) | |
| LIME | **lime** | |
| Counterfactuals | **DiCE** (dice-ml) | |
| Fairness | **Fairlearn** (metrics + mitigation) | |
| Deep learning XAI | **PyTorch + Captum** (Grad-CAM, Integrated Gradients, saliency, occlusion) | |
| Explanation evaluation | **Quantus** (faithfulness, robustness, randomisation metrics) | |
| Experiment tracking | **MLflow** (also used in project 05) | |
| UI | **Streamlit** or **Gradio** | |
| Documentation | **Model Card** (Mitchell et al.), **Datasheet for Datasets** (Gebru et al.) | |

---

## 5. Architecture / workflow

```
 data/ ─▶ EDA + data card ─▶ glass-box baselines (LogReg scorecard, EBM)
                                  │ compare performance
                                  ▼
                        black box (LightGBM, tuned, calibrated)
                                  │
     ┌──────────────┬─────────────┼──────────────┬────────────────┐
     ▼              ▼             ▼              ▼                ▼
  Global XAI     Local XAI    Counterfactuals  Fairness audit   Explanation
  (PI, PDP/ICE,  (SHAP, LIME, (DiCE recourse)  (Fairlearn)      evaluation
   ALE, SHAP)     reason codes)                                  (stability,
                                                                 faithfulness)
     └──────────────┴─────────────┴──────────────┴────────────────┘
                                  ▼
       Model Card + XAI Audit Report + Streamlit "Loan Officer Console"

 Separate track: pretrained image model ─▶ Grad-CAM / IG (Captum) ─▶ sanity checks
```

---

## 6. Final deliverables
1. Repo `fairlend/` with notebooks **for exploration only** and a `src/` package for the reusable logic (real-world practice), plus tests.
2. A **Model Card** + **Datasheet** (Markdown).
3. An **XAI Audit Report** (4–6 pages): performance, global behaviour, local examples, recourse, fairness, explanation reliability, limitations, and a recommendation (glass box or black box?).
4. A **Streamlit "Loan Officer Console"**: enter or select an applicant → prediction + calibrated probability + top-5 reason codes + a counterfactual suggestion.
5. A deep learning XAI notebook with Grad-CAM/IG visuals **and** sanity-check results.

## 7. If I skipped earlier projects
None are required. Basic scikit-learn and pandas are enough. The model I build here is the one I'll **ship in project 05 (MLOps)**, so build it cleanly.
