# GNN-Based Function-Level Vulnerability Detection in Smart Contracts

A graph neural network pipeline that detects vulnerable functions in Solidity smart contracts. Contracts are represented as function-level graphs combining call relationships and a novel state-flow edge type, and a GraphSAGE model is trained to flag risky functions with uncertainty estimates and source-line explanations to support manual audit.

Course project for *Interdisciplinary Deep Learning on Graphs* (UE23AM342BA1), PES University.

**Authors**: Smruthi B S, Subashri V

## Overview

Smart contracts are immutable once deployed, so vulnerabilities can't be patched later, rather they have to be caught before deployment. This project frames vulnerability detection as node classification on a graph: each function in a contract is a node, edges capture both function-call relationships and shared state-variable dependencies, and the model learns which nodes correspond to vulnerable functions across 6 vulnerability categories (reentrancy, unchecked low-level calls, time manipulation, bad randomness, access control, denial of service).

## What's different from prior work (e.g. BugSweeper)

- **State-flow edges**: alongside standard call edges, a second edge type captures shared state-variable read/write dependencies between functions which is not present in prior graph-based approaches. In ablation, adding state-flow edges improved F1 by +0.509 (0.694 vs. 0.185 on curated-only training), confirming this signal carries real vulnerability information beyond call structure alone.
- **MC Dropout uncertainty estimation**: rather than a single binary prediction, the model runs 30 stochastic forward passes per function to produce a per-function confidence score, letting auditors prioritize uncertain flags.
- **Source-line explainability**: GNNExplainer importance scores are mapped back to exact Solidity line numbers, producing a human-readable vulnerability heatmap rather than a function-level black-box verdict.

## Approach

- **Graph construction**: each contract is parsed into a function-level graph with two edge types — call edges (function-call graph) and state-flow edges (shared state-variable reads/writes).
- **Node features**: 12 features per function, including `sends_ether`, `calls_selfdestruct`, `cyclomatic_complexity`, `has_modifier_usage`, and 8 additional structural features.
- **Labeling**: SmartBugs Wild contracts (8,111 function-level graphs) were pseudo-labeled via Slither static analysis. SmartBugs Curated (121 graphs, expert-annotated) was used as the held-out test set.
- **Class imbalance**: vulnerable nodes are rare (~15:1 safe-to-vulnerable ratio). Focal loss (γ=3.0) with class weighting (1:12) was validated via a 5-seed ablation, improving recall by +27.8% over weighted cross-entropy (0.629 vs. 0.492).
- **Model**: GraphSAGE for node-level binary classification.
- **Fine-tuning**: a hard-negative fine-tuning pass to improve discrimination on harder examples.
- **Uncertainty + explainability**: MC Dropout (30 passes) for per-prediction confidence, GNNExplainer for node/edge importance and line-level heatmaps.

## Dataset

| Property | SmartBugs Wild (Train) | SmartBugs Curated (Test) |
|---|---|---|
| Source | GitHub Solidity contracts | Expert-annotated benchmark |
| Size | 8,111 function-level graphs | 121 graphs |
| Labels | Pseudo-labeled via Slither | Manually verified |
| Vulnerability types | 6 categories | 6 categories |

## Results

| Model | F1 | Precision | Recall |
|---|---|---|---|
| Weighted CE, no early stop (baseline) | 0.292 | 0.209 | 0.484 |
| Curated-only training | 0.694 | 0.581 | 0.862 |
| SAGE + Focal Loss (Wild to Curated) | 0.182 | 0.168 | 0.199 |
| SAGE + Focal Loss + MC Dropout (best) | 0.517 | 0.423 | 0.665 |
| Finetuned on Hard Negatives | 0.460 | 0.357 | 0.646 |
| Finetuned + MC Dropout | 0.507 | 0.415 | 0.652 |

**Best model**: SAGE + Focal Loss + MC Dropout (F1=0.517, Precision=0.423, Recall=0.665) — best balance of precision and recall.

Key findings:
- Focal loss (gamma=3.0) improves recall by +27.8% over weighted cross-entropy (0.629 vs. 0.492, averaged across 5 seeds).
- State-flow edges contribute a +0.509 F1 delta in the curated ablation (0.694 vs. 0.185).
- MC Dropout (30 forward passes) produces per-function uncertainty scores (mean approx. 0.114), with higher uncertainty observed for vulnerable nodes. This is useful for audit prioritization.
- GNNExplainer identifies vars_read, sends_ether, and ext_calls as the top vulnerability signals, and shows that state-flow edges (the novel edge type) are frequently flagged as critical for reentrancy/access-control vulnerabilities; validating the new edge type.

## Known Limitations

- Slither-based pseudo-labels are noisy and can introduce mislabeling in the Wild training set.
- Many Wild contracts use deprecated Solidity pragma versions that Slither cannot parse, limiting dataset coverage.
- The ~15:1 class imbalance means a strong overall F1 can still mask low recall on rarer vulnerability classes (e.g., bad randomness).
- MC Dropout uncertainty scores are not calibrated, so it's unclear whether high-uncertainty predictions correspond to genuinely ambiguous code.
- Generalization to vulnerability patterns not covered by Slither's detectors (e.g., logic bugs, oracle manipulation) remains untested.

## Tech Stack

PyTorch, PyTorch Geometric, Slither, GraphSAGE, GNNExplainer

## Repository Contents

- `vulnerability_detection.ipynb` — full pipeline: data collection, graph construction (with state-flow edges), pseudo-labeling, feature extraction, training (with focal loss and 5-seed ablations), hard-negative fine-tuning, evaluation, and GNNExplainer-based explainability. Designed to run on Google Colab with GPU acceleration.

## Future Work

- Per-vulnerability-class F1 breakdown
- Head-to-head benchmarking against BugSweeper
- Calibration of MC Dropout uncertainty scores
- Expanded coverage for deprecated Solidity versions
