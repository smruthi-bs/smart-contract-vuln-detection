# GNN-Based Function-Level Vulnerability Detection in Smart Contracts

A graph neural network pipeline for detecting vulnerable functions in Solidity smart contracts. Each contract is represented as a function-level graph with inter-function call edges and state-flow edges that connect functions sharing on-chain state dependencies. A GraphSAGE model performs function-level vulnerability classification, with MC Dropout uncertainty estimates and GNNExplainer-based source-line explanations to support security auditing.

**Authors:** Smruthi B S, Subashri V, Bhaskarjyoti Das

---

## Overview

Smart contracts are self-executing programs deployed on blockchain networks. However, vulnerabilities in their implementation can expose decentralized applications to significant security risks and financial losses.

Traditional smart contract vulnerability detection approaches often rely on static analysis tools and predefined vulnerability patterns. While useful, these approaches may struggle to capture complex relationships and dependencies between different functions within a contract.

This project explores a **Graph Neural Network (GNN)-based approach to smart contract vulnerability detection** by representing Solidity functions as program graphs.

The proposed framework captures two types of relationships:

- **Function-call edges:** Represent interactions and invocation relationships between functions.
- **State-flow edges:** Represent dependencies between functions through shared state variables.

These graph representations are used to train Graph Neural Networks, particularly GraphSAGE, to identify vulnerable functions.

The framework additionally incorporates uncertainty estimation and explainability techniques to provide insights into model predictions and the structural components contributing to vulnerability detection.

## Key Contributions

- **Function-level graph representation:** models Solidity contracts as graphs whose nodes represent functions.
- **State-flow edges:** supplements call relationships with shared-state read/write dependencies. The ablation in the paper shows a modest F1 improvement of `+0.027` when state-flow edges are included.
- **Imbalance-aware learning:** uses focal loss (`γ = 3.0`) with class weighting to focus training on difficult examples. Across five seeds, focal loss improves recall over weighted cross-entropy; the F1 difference is not statistically significant.
- **Hard-negative fine-tuning:** adds a subsequent fine-tuning stage using a small pool of Curated training graphs, additional Wild graphs, and hand-crafted Solidity examples.
- **Uncertainty estimation:** performs 30 stochastic MC Dropout forward passes and uses the standard deviation of the vulnerable-class probability as an uncertainty signal.
- **Source-line explanations:** uses GNNExplainer feature and edge masks, then maps node importance to Solidity source-line locations.

## Methodology

### 1. Graph Construction

Solidity contracts are parsed using Slither. Each function becomes a node in a directed graph.

Two edge types are constructed:

- **Call edges:** added when a function calls another function in the same contract.
- **State-flow edges:** added between distinct functions when one writes a state variable that the other reads. This is a lightweight shared-state dependency heuristic; it is not a full statement-level data-flow graph.

### 2. Node Features

Each function is represented by a 12-dimensional feature vector:

1. Lines of code (LOC)
2. Number of parameters
3. Number of external calls
4. Whether the function sends Ether
5. Visibility indicator
6. Whether the function is a modifier
7. Number of state variables read
8. Number of state variables written
9. Whether the function is payable
10. Cyclomatic complexity
11. Whether the function calls `selfdestruct`
12. Whether the function uses a modifier

Continuous features are min-max normalised per graph during graph construction.

### 3. Labelling and Datasets

- **SmartBugs Wild:** Slither detector outputs are mapped to source lines and used to create function-level pseudo-labels.
- **SmartBugs Curated:** manually annotated vulnerability lines are mapped to functions to create the evaluation labels.

The graph-construction run reported 8,111 successfully parsed Wild graphs and 121 successfully parsed Curated contracts, containing 1,392 function nodes.

The six vulnerability categories are reentrancy, unchecked low-level calls, time manipulation, bad randomness, access control, and denial of service.

### 4. Model and Training

The primary model is a three-layer GraphSAGE network with a hidden dimension of 64, batch normalisation, dropout, and a two-layer MLP node classifier. A GCN implementation is also included as a baseline.

Focal loss with `γ = 3.0` and a safe-to-vulnerable class-weight ratio of `1:12` is used for the main Wild-trained model. Training uses Adam, early stopping, and a Wild validation split.

A hard-negative fine-tuning stage follows pre-training. Fine-tuned results are evaluated on a stratified 30% held-out split of SmartBugs Curated; the remaining Curated graphs are included in the fine-tuning pool. Consequently, the fine-tuned result should be interpreted as performance on this held-out Curated split, not as zero-shot performance on the entire independent Curated benchmark.

### 5. Uncertainty and Explainability

At inference time, dropout remains active for 30 forward passes. The mean vulnerable-class probability is used for prediction, while the standard deviation across passes provides an epistemic-uncertainty signal.

GNNExplainer produces node-feature and edge-importance masks. The notebook maps node-importance scores to Solidity source-line locations using source mappings extracted during parsing. These explanations are illustrative audit aids; they have not been validated through a user study with security auditors.

## Experimental Results

The following results are reported in the accompanying research paper. F1, precision, and recall refer to the vulnerable (positive) function class.

| Configuration | F1 | Precision | Recall |
|---|---:|---:|---:|
| Weighted cross-entropy baseline | 0.292 | 0.209 | 0.484 |
| Slither static-analysis baseline | 0.500 | 0.401 | 0.665 |
| GCN + focal loss (5-seed mean) | 0.184 ± 0.020 | 0.127 ± 0.012 | 0.330 ± 0.045 |
| GraphSAGE + focal loss, zero-shot | 0.284 | 0.183 | 0.634 |
| GraphSAGE + focal loss + MC Dropout, zero-shot | 0.312 | 0.215 | 0.571 |
| Hard-negative fine-tuning, no MC Dropout (5-seed mean) | 0.484 ± 0.109 | 0.462 ± 0.172 | 0.576 ± 0.096 |
| **Hard-negative fine-tuning + MC Dropout (5-seed mean)** | **0.605 ± 0.115** | **0.580 ± 0.157** | **0.662 ± 0.050** |

**Best reported configuration:** fine-tuned GraphSAGE + focal loss + MC Dropout, with F1 `0.605 ± 0.115` on the held-out 30% Curated split.

### Additional Findings

- **Focal loss:** across five seeds, mean recall was `0.653 ± 0.018` for focal loss versus `0.480 ± 0.048` for weighted cross-entropy. The recall difference was statistically significant (`p = 0.0004`); the F1 difference was not (`p = 0.310`).
- **State-flow edge ablation:** on the matched Curated-only ablation split, F1 was `0.233` with state-flow edges and `0.206` without them (`+0.027`). The paper notes that this is a modest result and was not tested for statistical significance.
- **Precision-recall performance:** the fine-tuned model achieved an AUC-PR of `0.701` on the held-out 30% Curated split.
- **Uncertainty behaviour:** the paper reports the expected qualitative pattern of lower MC Dropout variance for predictions near 0 or 1 and higher variance for mid-range predictions. The relationship between uncertainty and misclassification has not yet been established.
- **Explainability example:** the paper analyses a reentrancy-flagged `withdraw()` function in the FibonacciBalance contract. The feature-importance example is illustrative and should not be interpreted as an aggregate ranking across the dataset.

**Evaluation note:** the zero-shot configurations are evaluated on the full 121-contract Curated set. Fine-tuned configurations are evaluated only on the held-out 30% split, because the other Curated graphs participate in fine-tuning. The fine-tuned score is therefore not directly comparable to a fully untouched, full-benchmark zero-shot result. The Slither baseline is reported separately as a benchmark reference.

## Known Limitations

- **Pseudo-label noise:** Slither-based labels on Wild may contain false positives and false negatives.
- **Solidity-version coverage:** 6,889 of the 15,000 candidate Wild contracts were skipped during graph construction, including contracts that could not be parsed or used unsupported Solidity versions.
- **Rare categories:** some vulnerability categories have very few positive examples in the Curated evaluation split, making per-class metrics unstable.
- **Modest state-flow contribution:** the measured F1 improvement from adding state-flow edges is small (`+0.027`) and needs further evaluation on larger datasets.
- **Uncertainty calibration:** MC Dropout uncertainty scores have not been formally calibrated, and their relationship with prediction errors has not been quantified.
- **Scope of detection:** vulnerabilities outside the six selected categories and Slither detector coverage, such as logic bugs and oracle manipulation, remain untested.
- **Explainability validation:** source-line heatmaps are illustrative explanations; their usefulness and impact on auditor performance have not been evaluated in a controlled user study.
- **Baseline comparison:** a direct BugSweeper comparison was not performed because a public implementation was unavailable and the benchmark differs.

## Repository Structure

```text
smart-contract-vuln-detection/
│
├── GRAPHite_vuln_detection.ipynb
│
├── paper/
│   └── ICCIS_2026_Paper.pdf    
│
├── results/
│   ├── auc_pr_curve.png
│   ├── uncertainty_histogram.png
│   ├── confidence_scatter.png
│   ├── state_flow_degree.png
│   ├── feature_importance.png
│   ├── line_heatmap.png
│   └── edge_importance_graph.png
│
├── requirements.txt
│
└── README.md
```

## Running the Notebook

The notebook is designed for Google Colab with GPU acceleration. It downloads or clones the required SmartBugs datasets and uses Slither and Solidity compiler tooling during parsing.

1. Open the notebook in Google Colab.
2. Select a runtime with GPU acceleration if available.
3. Install the Python dependencies listed in `requirements.txt` and the Solidity compiler tooling required by the notebook.
4. Run the notebook cells in order, ensuring the dataset download and parsing steps complete successfully.

Exact package and compiler versions may need to be adjusted for the Colab runtime. The notebook installs Solidity compiler version `0.4.25` in its setup cell; graph parsing also attempts to install compiler versions specified by individual contracts.

## Future Work

- Evaluate per-vulnerability-category performance on larger and more representative test sets.
- Calibrate MC Dropout uncertainty, for example using temperature scaling or conformal prediction.
- Expand Solidity compiler-version coverage and improve parsing success.
- Investigate additional graph signals and evaluate state-flow edges on larger benchmarks.
- Explore unsupervised anomaly detection and extension to other EVM-compatible chains.
- Conduct a controlled study to assess whether source-line explanations help auditors.
- Reproduce or benchmark against function-level graph approaches such as BugSweeper when a comparable implementation and benchmark are available.

## Research Presentation & Publication

This research work was accepted and presented at the **8th International Conference on Communication and Intelligent Systems (ICCIS 2026)**.

**Authors:**
- Smruthi B S
- Subashri V
- Bhaskarjyoti Das

### Publication Details

The conference proceedings are scheduled to be published in **Springer's Lecture Notes in Networks and Systems (LNNS)** book series, which is indexed in Scopus.

- **Conference:** 8th International Conference on Communication and Intelligent Systems (ICCIS 2026)
- **Presentation Status:** Accepted and Presented
- **Publication Series:** Springer Lecture Notes in Networks and Systems (LNNS)
- **Publication Status:** Forthcoming

### Presented Paper

The accepted and presented version of the research paper is available in the paper/ directory.

The final published version and official citation details will be updated once the conference proceedings are released.

## Acknowledgements

We acknowledge the SmartBugs benchmark datasets and the open-source tools and libraries that supported the development and evaluation of this research.


---

**Research Areas:** Smart Contract Security | Blockchain | Graph Neural Networks | Deep Learning | Explainable AI
