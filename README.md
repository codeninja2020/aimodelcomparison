# AI Model Comparison — FEM experimental dataset analysis

Out of curiosity I recently revisited a notebook I created some time back to analyse a dataset from computational experiments. The notebook (and supporting files) use pandas, numpy, seaborn and scipy to explore and visualise results from a set of FEM simulations and the inputs produced by a genetic algorithm (GA).

The goal of this repository is to compare the reporting features two AI models produced from the same experimental results versus the human effort required to interpret them.

## Short summary

- I ran a small experiment comparing two models' ability to analyse the same dataset: Anthropic Claude and OpenAI Codex.
- Codex was given full context (experimental results plus algorithm design and other notes). Claude was given only the flat results CSV to see whether it would request more context.
- Both models produced plausible analyses. Codex produced deeper, more problem-specific insight (likely due to the extra context). Claude's output was high-level and, importantly, did not request the additional context I intentionally withheld.
- The main lesson: selecting an optimal solution from scientific results often requires domain knowledge and interpretation beyond the raw numbers. This raises questions about how transformative AI is on scientific workflows when context is limited.

## Repository contents

Top-level files:

- `volumes.csv` — raw CSV of FEM simulation inputs and outputs produced by the genetic algorithm (large dataset used in the experiment).
- `codex-analysis.md` — exported analysis and outputs from Codex given the full context.
- `claude_analysis.md` — exported analysis and outputs from Claude given only the CSV.
- `prompt.md` — the prompt and instructions used for one or both models (see file for exact wording).
- `human-effort.pdf` — short write-up / notes on the human effort estimation for the task (PDF).

## Data and experiment

The dataset contains FEM simulation outputs and the GA-generated inputs used to produce those simulations. Each row in `volumes.csv` corresponds to one simulation run and includes the input parameters (design variables) and the computed outputs (performance metrics) from the FEM solver.

The experiment compared what each model would extract and report from the dataset, and how a human expert's effort compares to the automated outputs.

Model setup
- Codex: provided full context (experimental results plus notes on algorithm design and any other supporting context available to me at the time).
- Claude: provided only the flattened CSV of results. I intentionally withheld contextual design notes to observe whether Claude would ask clarifying questions.

## Reproducing the quick inspection

To quickly inspect the dataset and the analysis files locally:

1. Clone this repository:

```
git clone https://github.com/codeninja2020/aimodelcomparison.git
cd aimodelcomparison
```

2. (Optional) Create a virtual environment and install the libraries used in the original notebook:

```
python -m venv .venv
source .venv/bin/activate  # or .venv\Scripts\activate on Windows
pip install --upgrade pip
pip install pandas numpy seaborn scipy matplotlib
```

3. Open the CSV with pandas (example):

```python
import pandas as pd
df = pd.read_csv('volumes.csv')
print(df.shape)
print(df.head())
```

4. Read the model outputs and notes:
- `codex-analysis.md` — Codex's exported analysis (contains figures, tables and narrative).
- `claude_analysis.md` — Claude's exported analysis (narrative based on the CSV only).
- `prompt.md` — the prompt(s) used during the runs.
- `human-effort.pdf` — a short human-effort write-up to compare manual analysis time/costs.

## Observations and interpretation (author notes)

- Both models produced useful analysis, but the depth and specificity depended strongly on context provided to the model.
- Codex's analysis was more in-depth and actionable because it had extra context; Claude's output was higher-level and did not attempt to request the withheld context.
- Important conceptual point: choosing an "optimal" scientific solution is not purely numeric. Domain knowledge and interpretation of what metrics mean in context are critical.

Open questions that came out of this small experiment:

- Are we overstating AI's contribution to transformative scientific work when models are provided only dataset snapshots without domain context?
- Is the main value proposition of these models the speed and convenience of summarisation, or can they truly replace deep domain expertise when context is missing?

These questions are intentionally left open in the repository. The analysis files contain the raw outputs so you can judge for yourself.

## Try asking

- How can I extend the analysis in `codex-analysis.md` to include statistical hypothesis tests between top candidate designs?
- Where in `volumes.csv` should I look for unreliable FEM outputs (outliers / solver failures)?
- What additional context (files or notes) would most improve Claude's analysis if I re-run the experiment?

## License

This repository currently does not include an explicit license. If you want this work reused, add a LICENSE file.

## Contact

Author: codeninja2020

Feedback and PRs welcome — the repository is small and meant for exploration and discussion.
