# FairLend: milestones

> Format: **Goal → Why now → Concepts → What to build (Core / Stretch) → Best practices → Definition of Done → Pitfalls**. Total: 7 milestones over about 4 days.

---

## 💸 Zero-cost rule: this whole project costs $0 and needs no card

Everything runs **on my PC**. My laptop has an **NVIDIA RTX 4080 Laptop GPU with 12 GB VRAM**, which is enough to fine-tune the image model in M5 locally, so I don't need to pay for cloud GPUs.

| Need | Free choice ✅ | Avoid ❌ |
|------|---------------|---------|
| Datasets | UCI Credit Default (free download), Kaggle datasets (free account) | Paid data vendors |
| Tabular models + XAI | scikit-learn, LightGBM/XGBoost, InterpretML, SHAP, LIME, DiCE, Fairlearn, Quantus (all open source) | Commercial XAI platforms |
| GPU for M5 | **My RTX 4080 (12 GB)**. I install the CUDA build of PyTorch with uv (search "uv pytorch CUDA index"). Backup: **Google Colab free tier** or **Kaggle free GPU** (phone verification, no card) | Colab Pro, paid cloud GPUs |
| Experiment tracking | **MLflow** running locally | Paid SaaS tiers |
| Demo app | **Streamlit/Gradio locally**. To share it: **Hugging Face Spaces** (free CPU) or **Streamlit Community Cloud** (free) | Paid hosting |

**My hard rule:** never enter a card. The GPU is free because it's mine, so I watch temperatures and plug the laptop in while training.

---

## M0: Framing: who needs an explanation, and of what? (Day 1, first ~2 h)

**Goal:** a clear map of the XAI landscape and of the regulatory and business reasons for it, *before* I touch data.

**Why now:** "explainability" means different things to different people. If I don't define **who** the explanation is for and **what decision** it supports, I'll produce pretty SHAP plots that answer nobody's question, which is the most common XAI failure in industry.

**Concepts to learn**
- Interpretability vs explainability; transparency vs post-hoc explanation
- The taxonomy: **intrinsic vs post-hoc**, **global vs local**, **model-specific vs model-agnostic**, feature attribution vs example-based vs counterfactual
- Stakeholders and their needs (developer, domain expert, end user, regulator)
- Regulation: EU AI Act (high-risk systems, transparency, human oversight, with the **2026 Digital Omnibus delay** to Dec 2027 for Annex III), GDPR Art. 22 and the "right to explanation" debate, Morocco Law 09-08, US adverse action notices (ECOA / Regulation B)
- Rudin (2019), "Stop explaining black box machine learning models for high stakes decisions…". Read the abstract and the main argument.

**What to build**
- Core: `docs/00_framing.md` containing a stakeholder → question → explanation-type table (extend the one in the project idea), the decision being automated, the cost of errors (false approval vs false rejection), and the regulatory notes.

**Definition of Done**
- [ ] Framing doc written; I can explain the XAI taxonomy without notes

---

## M1: Data, EDA, and glass-box baselines (Day 1, ~4 h)

**Goal:** understand the data and build **inherently interpretable** models first.

**Why now:** good practice says *start interpretable*. If a glass-box model performs within ~1–2% of a black box, the glass box often wins in regulated industries. I can only make that call once I have a strong interpretable baseline.

**Concepts to learn**
- Data leakage (features that are only known *after* the decision); train/validation/test splits; stratification
- Class imbalance; the right metrics: **ROC-AUC, PR-AUC, Brier score**, plus business-cost-based thresholds
- Logistic regression coefficients, odds ratios, **WoE / IV** and credit scorecards (search "credit scorecard WoE IV")
- Decision trees (depth-limited) and their readable rules
- **Explainable Boosting Machines (EBM)**: GAMs with interactions (search "InterpretML EBM GA2M")

**What to build**
- Core: EDA (target rate, distributions, missing values, correlations) and a **datasheet** for the dataset, including sensitive attributes and known biases.
- Core: decide **whether to use sensitive attributes (sex, age, marriage) as model features**. Document my decision (an ADR). In many jurisdictions I can't use them for the decision but **must keep them for fairness auditing**.
- Core: a preprocessing pipeline (`sklearn` `Pipeline` / `ColumnTransformer`). Fixed random seeds. A **held-out test set that I don't touch** until the final evaluation.
- Core: baselines: (1) logistic regression (scaled, regularised), (2) depth-3 decision tree, (3) **EBM**. Report AUC, PR-AUC, Brier, and **interpret each model directly** (coefficients, tree rules, EBM shape functions).
- Stretch: a points-based scorecard (WoE binning + logistic regression).

**Best practices:** pipelines prevent leakage; track every run (MLflow); notebooks for exploration, `src/` for logic.

**Definition of Done**
- [ ] Datasheet + sensitive-attribute ADR
- [ ] 3 glass-box models with metrics and their own interpretation
- [ ] The test set is still untouched

---

## M2: Black-box model and global explanations (Day 2 morning, ~4 h)

**Goal:** train a strong gradient-boosting model and explain its **overall** behaviour with several global methods, then compare them critically.

**Why now:** now I have a glass-box reference. I build the black box and ask, "Does it behave sensibly overall?" before looking at individual predictions.

**Concepts to learn**
- LightGBM/XGBoost basics, hyperparameter tuning (Optuna), early stopping
- **Probability calibration** (reliability diagrams, isotonic/Platt). Explanations of uncalibrated probabilities can mislead.
- **Impurity-based feature importance and why it's biased** (toward high-cardinality features) vs **permutation importance** (computed on validation data)
- **PDP vs ICE vs ALE**: why PDP misleads when features are correlated, and how ALE fixes it
- **SHAP**: Shapley values from game theory, efficiency/symmetry/dummy/additivity axioms, **TreeSHAP**, global summary (beeswarm) plots, dependence plots, **interaction values**
- **Monotonic constraints** (e.g. more payment delays must never *decrease* risk)

**What to build**
- Core: a tuned LightGBM/XGBoost model, calibrated. Compare it with the M1 models on validation data.
- Core: global explanations: built-in importance vs permutation importance vs mean |SHAP|. **Put the three rankings side by side and explain the disagreements.**
- Core: PDP + ICE for the top 3 features, ALE for 2 correlated features; explain the differences.
- Core: the SHAP beeswarm + 2 dependence plots + the top interaction pair.
- Core: retrain with **monotonic constraints** on features where domain logic demands them, and report the performance cost.
- Stretch: compare the EBM shape functions with the SHAP dependence plots for the same feature.

**Definition of Done**
- [ ] Calibrated black box with metrics vs glass boxes
- [ ] 3 importance methods compared, with disagreements explained
- [ ] PDP/ICE/ALE + SHAP global plots in the report, with **my interpretation** next to each plot
- [ ] Monotonic-constraint experiment done

**Pitfalls:** computing permutation importance on the training set; reading SHAP as **causal** (it's not: it explains the *model*, not the *world*).

---

## M3: Local explanations and reason codes (Day 2 afternoon, ~4 h)

**Goal:** explain **individual** decisions reliably, and test whether those explanations can be trusted.

**Why now:** global views tell me the model is sensible *on average*. Individual customers and loan officers need case-by-case reasons, and local explanations are where most XAI pitfalls (instability, correlated features) show up.

**Concepts to learn**
- SHAP local (waterfall/force plots); the **baseline/expected value** and why the choice of background dataset matters
- **Interventional vs observational (tree-path-dependent) SHAP**, and the effect of correlated features
- **LIME**: local surrogate models, kernel width, the neighbourhood sampling problem
- **Explanation stability/robustness**: do similar inputs get similar explanations? Does LIME give the same answer twice?
- Reason codes / adverse action notices: turning attributions into plain language a customer can understand

**What to build**
- Core: pick 6 cases: 2 confident rejects, 2 confident approvals, 2 borderline. For each, produce a SHAP waterfall and a LIME explanation, then **compare them**.
- Core: **stability experiments**: (a) run LIME 20 times on the same case and measure the variance of the top-5 features; (b) add tiny noise to an input and measure how much the SHAP ranking changes (rank correlation).
- Core: a **reason-code generator**: top-N features that pushed toward rejection → human-readable sentences ("Recent payment delays in the last 2 months increased my risk"). Exclude sensitive features from the customer-facing text, and document why.
- Stretch: try interventional vs tree-path SHAP on correlated features and show the difference.

**Definition of Done**
- [ ] 6 cases explained with SHAP vs LIME and a written comparison
- [ ] Stability numbers reported
- [ ] Reason-code generator works and its output reads naturally

---

## M4: Counterfactuals, recourse, and fairness (Day 3 morning, ~4 h)

**Goal:** tell rejected applicants **what they could change**, and audit whether the model treats groups fairly.

**Why now:** attributions explain *why*. Counterfactuals explain *what to do*, which is what customers actually want. Fairness is closely tied to explanation (a model can be accurate yet unfair, and explanations can reveal proxy discrimination), so it's audited here, once I understand the model's behaviour.

**Concepts to learn**
- Counterfactual explanations (Wachter et al. 2017); properties: **validity, proximity, sparsity, diversity, actionability, plausibility**
- **Algorithmic recourse**: immutable features (age, past history) vs actionable ones
- Fairness definitions: **demographic parity, equal opportunity, equalized odds**, predictive parity, calibration within groups, and the **impossibility theorems** (I can't satisfy all of them at once)
- Proxy discrimination (education or marriage acting as proxies for sex or age)
- Mitigation: pre-processing (reweighing), in-processing (constraints), post-processing (threshold optimizer)

**What to build**
- Core: DiCE counterfactuals for 3 rejected applicants, with **immutable features locked** and realistic ranges. Evaluate validity, sparsity, and proximity.
- Core: a Fairlearn `MetricFrame` across sex, age group, and education: selection rate, TPR, FPR, AUC per group. Identify the largest disparity.
- Core: **proxy check**: does removing sex from the features remove the disparity? (Usually not, and explaining why is the lesson.) Use SHAP to look for proxy features.
- Core: apply **one mitigation** (e.g. ThresholdOptimizer for equalized odds) and report the **accuracy–fairness trade-off** as a table.
- Stretch: check **explanation fairness**: do rejected applicants in different groups get different kinds of reasons or counterfactual costs?

**Definition of Done**
- [ ] Actionable counterfactuals for 3 cases
- [ ] Fairness metrics per group + 1 mitigation with its trade-off
- [ ] Proxy discrimination discussion written

---

## M5: Deep learning explainability (Day 3 afternoon + Day 4 morning, ~5 h)

**Goal:** explain neural network predictions with gradient-based and perturbation methods, and **test whether those explanations are real**.

**Why now:** tabular XAI is the most common in industry, but AI engineers also work with CNNs, ViTs, and transformers, where the methods are different (gradients, activations). Doing this last means I already know *what a good explanation should satisfy*.

**Concepts to learn**
- Saliency maps (vanilla gradients), **Integrated Gradients** (baselines, completeness axiom), SmoothGrad
- **Grad-CAM** (class activation maps from the last conv layer); attention rollout for ViTs and why "attention is not explanation"
- Occlusion / perturbation methods
- **Sanity checks for saliency maps** (Adebayo et al. 2018): the model-randomisation and data-randomisation tests
- Faithfulness metrics: deletion / insertion curves (ROAR idea)
- Awareness: **mechanistic interpretability** (features, circuits, sparse autoencoders) is the 2025–2026 research frontier for LLMs. Read 1 overview article.

**What to build**
- Core: fine-tune a **pretrained** ResNet/EfficientNet/ViT on Oxford Pets or Chest X-Ray for 1–3 epochs on **my RTX 4080 (12 GB)**, using mixed precision and a batch size that fits in VRAM. Colab or Kaggle free GPUs are a backup.
- Core: for 8 images (correct and wrong predictions) produce **Saliency, Integrated Gradients, Grad-CAM, and Occlusion** with Captum, side by side.
- Core: the **model-randomisation sanity check**: randomise the weights layer by layer, recompute the maps, and see which methods still produce "nice" maps (those methods fail the test).
- Core: a **deletion curve** for IG vs Grad-CAM: remove the top-attributed pixels progressively and plot the confidence drop.
- Core: look for a **"Clever Hans"** effect: does the model look at the background, a text marker, or an artifact instead of the object? (This is common in X-ray datasets.)
- Stretch: token attributions (IG) on a Hugging Face sentiment model, a bridge to project 04.
- Stretch: a short write-up after reading one sparse-autoencoder interpretability article.

**Definition of Done**
- [ ] 4 attribution methods compared on 8 images
- [ ] Sanity-check results with conclusions
- [ ] Deletion-curve plot
- [ ] Clever Hans analysis (found or ruled out, with evidence)

---

## M6: Evaluate, document, and communicate (Day 4 afternoon, ~4 h)

**Goal:** turn everything into artefacts a real organisation would require before deployment.

**Why now:** in industry, XAI work that isn't **documented and usable by non-experts** has no effect. This is also what I'll ship with the model in project 05.

**What to build**
- Core: final evaluation on the **untouched test set** (performance + calibration + fairness).
- Core: a **Model Card** (intended use, out-of-scope uses, metrics overall and by group, ethical considerations, caveats, explanation methods used and their limits).
- Core: the **XAI Audit Report**, ending with a clear recommendation: deploy the EBM (glass box) or the LightGBM + SHAP (black box + post-hoc)? Justify it with numbers.
- Core: the **Streamlit Loan Officer Console**: select or enter an applicant → probability, decision at the business threshold, top reason codes, SHAP waterfall, one counterfactual, and a **disclaimer** about the limits of the explanations.
- Core: save the final pipeline + explainer as versioned artefacts (I'll need them in project 05).
- Stretch: a "**human-in-the-loop**" design: when should a case be routed to a human (low confidence, unstable explanation, protected-group edge cases)?

**Definition of Done**
- [ ] Model Card + Datasheet + Audit Report finished
- [ ] Console works end-to-end
- [ ] Artefacts saved and versioned

---

## Self-check: interview questions

1. Global vs local, intrinsic vs post-hoc, model-agnostic vs model-specific: give one method for each.
2. Why can impurity-based feature importance be misleading? What do I use instead?
3. PDP vs ICE vs ALE: when does PDP lie?
4. Explain Shapley values intuitively. What does "additivity" guarantee? What does SHAP **not** tell me?
5. Why can LIME give different explanations for the same input, and how would I mitigate it?
6. What makes a counterfactual "actionable"? Give an example of a useless one.
7. Why doesn't removing the sensitive attribute guarantee fairness?
8. Demographic parity vs equalized odds: why can't I usually have both along with calibration?
9. What is the model-randomisation sanity check, and what does it say about some saliency methods?
10. My SHAP explanation says "income" matters most, but the business says it shouldn't. What do I investigate?
11. Under the EU AI Act, what obligations does a high-risk credit-scoring system have, and what changed in 2026?

## References to search
- **"Interpretable Machine Learning"** by Christoph Molnar (free online book, the reference text)
- Lundberg & Lee (2017), "A Unified Approach to Interpreting Model Predictions" (SHAP)
- Ribeiro et al. (2016), "Why Should I Trust You?" (LIME)
- Wachter et al. (2017), counterfactual explanations; Mothilal et al. (2020), DiCE
- Adebayo et al. (2018), "Sanity Checks for Saliency Maps"
- Fairlearn user guide; Captum tutorials; InterpretML docs
- Mitchell et al. (2019), "Model Cards for Model Reporting"
