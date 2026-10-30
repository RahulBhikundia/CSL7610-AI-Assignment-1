# General-Purpose Classical AI Logical Reasoning System

An end-to-end symbolic AI reasoning engine that determines whether a natural-language query statement is logically entailed by a knowledge base of facts and rules ($\text{KB} \models q$).

The system converts natural-language inputs into structured symbolic representations and performs all final logical deductions strictly through classical AI reasoning algorithms—without relying on generative AI for inference.

## 📁 Directory Structure

```text
submission/
├── Data                    # Folder containing provided datasets
├── reasoning_agent.ipynb   # Complete end-to-end implementation, evaluation, and CSV export
├── report.pdf              # Technical report (representations, algorithms, experiments, traces)
├── submission.csv          # Generated predictions (id, predicted_label)
└── README.md               # Markdown documentation
```

---

## 🧠 System Architecture & Reasoning Pipeline

The central reasoning pipeline follows a strict two-stage classical design:

```text
Natural Language KB ➔ Parser / Internal Representation ➔ Classical Reasoning Engine ➔ True / False
```

### 1. Internal Knowledge Representations

Depending on the structure of the input knowledge base, the system maps natural language into one of three formal representations:

- **First-Order Horn Clauses & Atomic Facts:** Represents ground facts $P(c)$ and universally quantified conditional rules $P_1(?x) \wedge \dots \wedge P_k(?x) \rightarrow Q(?x)$ for syllogistic and rule-chaining tasks (`RuleTaker` and general natural language).
- **Structural Causal Models (SCMs):** Represents directed causal graphs $\mathcal{G} = (\mathcal{V}, \mathcal{E})$ with structural equations $v_i := f_i(\text{Pa}(v_i), u_i)$ and observational mappings $\mathcal{O}$ for deterministic counterfactual queries (`CounterBench`).
- **First-Order Logic Abstract Syntax Trees (FOL AST):** Represents arbitrary quantified formulas involving $\forall$, $\exists$, $\neg$, $\wedge$, $\vee$, $\rightarrow$, $\leftrightarrow$, and $\oplus$, paired with memoized Tseitin Conjunctive Normal Form (CNF) encoding (`FOLIO`).

### 2. Classical AI Reasoning Algorithms Implemented

All entailment decisions are computed from scratch with classical AI reasoning. Forward chaining (RuleTaker), the causal engine (CounterBench) and DPLL refutation (FOLIO) form the evaluated path. Backward chaining is used for the natural-language fallback on general knowledge bases. A\* proof search is implemented as an alternative prover but is not used in the benchmark runs.

1. **Forward Chaining with Robinson Unification:** Data-driven fixed-point inference that unifies rule premises against known ground facts $\mathcal{F}_t$ until the query atom is derived or saturation is reached ($\mathcal{F}_{t+1} = \mathcal{F}_t$).
2. **Backward Chaining (DFS with Cycle Detection):** Goal-driven Depth-First Search that recursively unifies the target query with rule conclusions and proves subgoal conjunctions while preventing infinite loops on cyclic rule graphs.
3. **$A^*$ Heuristic Proof Search:** Formulates proof discovery as a heuristic graph search where state $s$ is the set of unproven subgoals, path cost $g(s)$ is the number of rule applications, and heuristic $h(s) = |s \setminus \mathcal{F}_0|$ guides search toward minimum-length proofs.
4. **Pearl's 3-Step Counterfactual Algorithm (Abduction–Action–Prediction):**
   - _Abduction:_ Fixes observed exogenous variables ("We observed ...") to their observed values; observations of derived variables are not used.
   - _Action:_ Applies counterfactual interventions $\text{do}(X = x)$ by overriding the structural equations of intervened variables.
   - _Prediction:_ Forward-propagates truth values through the modified causal graph to evaluate the query.
5. **Herbrand Grounding & Bounded DPLL / Resolution Refutation:** Instantiates quantified FOL formulas over a prioritized Herbrand universe $\mathcal{D}$, translates $\text{KB} \wedge \neg \text{Query}$ into equisatisfiable CNF via Tseitin transformation, and applies a step-bounded Davis–Putnam–Logemann–Loveland (DPLL) unit resolution solver with atom-branching heuristics. The Herbrand domain is capped at three constants and the search stops after 100 nodes (treated as "not entailed"). FOLIO rows use the provided `premises-FOL` / `conclusion-FOL` fields as input logic; rows without them fall back to the natural-language parser.

---

## ⚙️ Requirements & Installation

The reasoning engine is built using Python's standard library (`re`, `json`, `time`, `heapq`, `itertools`, `difflib`) and requires only `pandas` for exporting `submission.csv`.

- **Python:** 3.8+
- **Dependencies:**
  ```bash
  pip install pandas jupyter
  ```

---

## 🚀 How to Run

### 1. Standard Evaluation & CSV Generation

1. Place the dataset `.jsonl` files (`ruletaker_train.jsonl`, `ruletaker_test.jsonl`, `ruletaker_ood_test.jsonl`, `counterbench_alpha.jsonl`, `FOLIO_train.jsonl`, `FOLIO_validation.jsonl`) in the `Data/` folder next to `reasoning_agent.ipynb` (the notebook also works if they sit directly beside it).
2. Launch Jupyter Notebook and execute all cells sequentially:
   ```bash
   jupyter notebook reasoning_agent.ipynb
   ```
3. Running the notebook will:
   - Execute the sanity-check trace on the problem statement example (_"Bob is successful."_).
   - Evaluate all public datasets and print **Reasoning Accuracy** ($\frac{\text{Correct Predictions}}{\text{Total Queries}} \times 100$) and **Total Execution Time** (in seconds).
   - Automatically generate `submission.csv` containing columns `id,predicted_label`.

### 2. Running on the Private Test Set

To evaluate on an unseen private test file (e.g., `private_test.jsonl`), place the file in `Data/` and update the `dataset_files` list in the last code cell (section **6. Unified Reasoning Pipeline**) of `reasoning_agent.ipynb`:

```python
dataset_files = [
    "private_test.jsonl"
]
```

Re-run that cell to automatically detect the schema (RuleTaker `context`/`statement`, CounterBench `given_info`/`question`, FOLIO `premises`/`conclusion`), run classical inference, print execution metrics, and write `submission.csv`. If a record has an `id` field it is used as the row id; otherwise the id is `<filename>_<row index>`. Records without labels are handled (accuracy is then reported as 0 and can be ignored).

---

## 📊 Public Benchmark Performance

| Dataset Split              | Total Queries | Correct Predictions | Reasoning Accuracy | Execution Time |
| :------------------------- | :-----------: | :-----------------: | :----------------: | :------------: |
| `ruletaker_train.jsonl`    |     1,000     |         983         |       98.30%       |    0.0916s     |
| `ruletaker_test.jsonl`     |      250      |         244         |       97.60%       |    0.0229s     |
| `ruletaker_ood_test.jsonl` |      375      |         373         |       99.47%       |    0.0376s     |
| `counterbench_alpha.jsonl` |     1,000     |         960         |       96.00%       |    0.0609s     |
| `FOLIO_train.jsonl`        |     1,001     |         774         |       77.32%       |    0.8794s     |
| `FOLIO_validation.jsonl`   |      203      |         165         |       81.28%       |    0.2014s     |
| **OVERALL TOTAL**          |   **3,829**   |      **3,499**      |     **91.38%**     |  **1.2941s**   |

Notes: times are from one run and vary slightly between machines. For FOLIO, `True` means entailed; gold `False` and `Uncertain` are both counted as "not entailed" (strict three-way matching would give about 46% / 50%, since the system never outputs `Uncertain`). Results are deterministic (sets are sorted before iteration).
