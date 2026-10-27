# Percentage Misconception Tutor

A tutor that teaches percentages by finding the **specific wrong rule** behind a student's wrong answer, instead of only marking it wrong. It tracks each learner's misconceptions across sessions and checks whether targeted practice reduces the error types it diagnoses.

> **Design principle:** Python does all the maths and makes every diagnosis. An optional LLM only (1) turns typed working into a matching rule and (2) rewords our fixed explanations. The LLM never decides a diagnosis.

---

## Contents

1. [What it does](#what-it-does)
2. [The five misconception rules](#the-five-misconception-rules)
3. [How a diagnosis works](#how-a-diagnosis-works)
4. [Feedback shown to the student](#feedback-shown-to-the-student)
5. [Tracking misconceptions across sessions](#tracking-misconceptions-across-sessions)
6. [Evidence that targeted practice works](#evidence-that-targeted-practice-works)
7. [Evaluation](#evaluation)
8. [Role of the LLM](#role-of-the-llm)
9. [Project structure](#project-structure)
10. [Quick start](#quick-start)
11. [Limitations and ethics](#limitations-and-ethics)

---

## What it does

1. The student sees a percentage problem and enters a **number**.
2. They pick the **method they used** and rate their **confidence (1-5)**.
3. The tutor runs **every misconception rule** on the problem and reports which rules reproduce the student's answer.
4. The result is labelled **suspected**, **confirmed** or **uncertain**.
5. The tutor shows the **wrong method and the right method side by side**, using the student's own numbers, with a short hint and a verified follow-up problem.
6. Every attempt is stored, so mistakes can be followed across sessions.

---

## The five misconception rules

Each mistake is written as a small, deterministic Python rule that takes a problem and returns the answer a student holding that misconception would give.

| ID | Misconception | Example of the faulty reasoning |
|----|---------------|---------------------------------|
| `WRONG_BASE` | Takes the percentage of the wrong quantity | Computes a discount on the sale price instead of the original price |
| `ADDITIVE_SUCCESSIVE` | Adds successive percentage changes | +20% then +10% treated as +30% |
| `WRONG_INVERSE` | Reverses a percentage change incorrectly | Undoing a +25% rise by subtracting 25% |
| `PP_CONFUSION` | Confuses percentage points with percent | A move from 10% to 15% read as "up 5%" instead of "up 50%" |
| `PART_PART_AS_WHOLE` | Reads a part:part ratio as part:whole | Ratio 2:3 read as 2/3 of the total instead of 2/5 |

Illustrative rule interface:

```python
# rules/base.py
from dataclasses import dataclass

@dataclass(frozen=True)
class Problem:
    kind: str          # e.g. "successive_change"
    params: dict       # the numbers in the problem

class Rule:
    id: str
    def wrong_answer(self, problem: Problem) -> float | None:
        """Answer a student holding this misconception would give,
        or None if the rule does not apply to this problem kind."""
        raise NotImplementedError
```

Rules are pure functions with no randomness and no LLM calls, so every diagnosis is reproducible and unit-testable.

---

## How a diagnosis works

### Step 1: First attempt gives a *suspected* diagnosis

The tutor computes the correct answer and runs every rule. If one or more rules reproduce the student's answer (within a stated tolerance), those rules become the **suspected** diagnosis. A single answer can never do more than this, because several different rules can coincide on one problem.

### Step 2: Second probe

Code selects a second problem such that **the correct answer and every rule's wrong answer are all different**. This makes the probe discriminating: each possible explanation predicts a distinct answer.

| Second answer | Outcome |
|---------------|---------|
| Explained by the **same rule** as the first wrong answer | **Confirmed** |
| **Correct** | Treated as a **possible slip** (not a confirmed misconception) |
| Anything else (a different rule, or no rule) | **Uncertain** |

### Confidence

The 1-5 confidence rating is a **supporting signal only**. It is recorded and analysed, but it **never decides** the diagnosis or changes a label.

### Status summary

| Label | Meaning |
|-------|---------|
| `suspected` | One wrong answer matches one or more rules |
| `confirmed` | The same rule explains two wrong answers on a discriminating pair |
| `uncertain` | The evidence does not support a single rule |

---

## Feedback shown to the student

For a diagnosed rule, the tutor shows:

- **Wrong method vs right method, side by side**, computed on the student's own numbers.
- A **short hint** (fixed text per rule).
- A **verified follow-up problem**: code checks that it targets the same misconception and that the rule's wrong answer differs from the correct one, so the problem can actually tell the two apart.

Explanations are fixed templates written by us. The optional LLM may rephrase them in friendlier words but cannot change their content (see [Role of the LLM](#role-of-the-llm)).

---

## Tracking misconceptions across sessions

Every attempt is stored, giving each learner a misconception history.

Suggested minimum schema (SQLite is enough for the pilot):

```sql
CREATE TABLE attempts (
  id            INTEGER PRIMARY KEY,
  student_id    TEXT NOT NULL,          -- pseudonymous ID only
  session_id    TEXT NOT NULL,
  ts            TEXT NOT NULL,          -- ISO 8601
  problem_id    TEXT NOT NULL,
  problem_kind  TEXT NOT NULL,
  answer        REAL,
  correct       INTEGER NOT NULL,       -- 0/1, decided by Python
  method_chosen TEXT,                   -- student's self-reported method
  confidence    INTEGER CHECK (confidence BETWEEN 1 AND 5),
  matched_rules TEXT,                   -- JSON list of rule IDs
  label         TEXT,                   -- suspected | confirmed | uncertain | correct | possible_slip
  is_probe      INTEGER DEFAULT 0,
  practice_arm  TEXT                    -- targeted | random | none
);
```

From this the tutor reports, per student and per misconception:

- Which misconceptions are **suspected / confirmed** and when they were first seen.
- **Recurrence** across sessions (does the same rule reappear?).
- **Resolution**: a misconception is treated as resolved after a defined run of correct answers on problems that discriminate for that rule (threshold set in `config.yaml`).

---

## Evidence that targeted practice works

We run a **small peer pilot** comparing two practice conditions.

| Arm | Practice given |
|-----|----------------|
| **Targeted** | Problems chosen to exercise the student's diagnosed misconception |
| **Random** | Problems of the same difficulty, drawn at random |

**Measure:** the learner's error rate **for the targeted error type**, before and after the practice block, compared between arms.

**Design notes**

- Assign students to arms by random allocation, or alternate by session order if group sizes are too small.
- Use a pre-test and post-test with matched problem sets (code verifies matched difficulty and that rules' answers stay distinct).
- Keep the same measurement window for both arms.

**Reporting.** With a small sample the pilot **reports trends only**: per-arm before/after error rates, the change, and per-student trajectories. We do **not** claim statistical significance or causal effects. Suggested outputs:

- Before/after error rate per misconception, per arm (table and plot).
- Per-student trajectory plots across sessions.
- Raw counts alongside every percentage, so readers can see how small the numbers are.

```bash
python -m analysis.pilot_report --db data/tutor.db --out reports/pilot/
```

---

## Evaluation

Evaluation uses **peer-written answers**: classmates write wrong answers (and the working or method behind them) for a set of problems, and independent labellers assign the misconception behind each.

| Metric | What it tells us |
|--------|------------------|
| **Accuracy** | Share of answers where the tutor's diagnosis matches the human label |
| **Per-mistake F1** | Precision/recall for each of the five rules separately, so weak rules are visible |
| **Labeller agreement** | Agreement between human labellers (e.g. Cohen's kappa), setting a ceiling for what's achievable |
| **Wrongly forced diagnoses** | Cases where the tutor assigns a rule although the human label is "none / other / slip". Measures over-diagnosis |
| **Follow-up success rate** | Share of students who answer the verified follow-up problem correctly |
| **Confidence vs confirmation** | Whether higher self-rated confidence predicts which diagnoses the second probe confirms (analysis only; confidence is never used to diagnose) |

**Baseline:** a **zero-shot LLM** given the same problem and answer and asked to name the misconception, evaluated on the same peer-written set with the same metrics.

Because several rules can give the same wrong answer on one problem, we report results **separately** for ambiguous and unambiguous items, so the gain from the second probe is visible.

```bash
python -m eval.run --data data/peer_answers.csv --baseline zero_shot_llm --out reports/eval/
```

---

## Role of the LLM

The LLM is optional and strictly limited.

| Allowed | Not allowed |
|---------|-------------|
| Convert a student's **typed working** into one of the existing rule IDs (the result is then re-checked by running that rule in Python) | Compute any answer |
| **Reword** our fixed explanations and hints in friendlier language | Decide, change or upgrade a diagnosis |
| | Invent new misconceptions, hints or follow-up problems |

If the LLM proposes a rule, Python checks that the rule actually reproduces the student's answer. If it does not, the proposal is discarded. With the LLM switched off, the tutor still works end to end using the fixed templates.

---

## Project structure

```
.
├── README.md
├── config.yaml              # tolerance, resolution threshold, arm assignment
├── rules/                   # the five misconception rules (pure Python)
│   ├── base.py
│   ├── wrong_base.py
│   ├── additive_successive.py
│   ├── wrong_inverse.py
│   ├── pp_confusion.py
│   └── part_part_as_whole.py
├── tutor/
│   ├── diagnose.py          # runs all rules, assigns suspected/confirmed/uncertain
│   ├── probe.py             # picks a discriminating second problem
│   ├── feedback.py          # side-by-side wrong vs right, hints, follow-ups
│   └── llm.py               # optional: working -> rule, rewording
├── store/
│   ├── schema.sql
│   └── db.py                # attempt storage and per-student history
├── analysis/
│   └── pilot_report.py      # before/after, targeted vs random, trends only
├── eval/
│   ├── run.py               # accuracy, per-mistake F1, agreement, forced diagnoses
│   └── baseline_llm.py      # zero-shot LLM baseline
├── data/
└── tests/
```

---

## Quick start

```bash
git clone <repo-url>
cd percentage-tutor
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

pytest                       # rule and diagnosis tests
python -m tutor.cli          # run a tutoring session in the terminal
```

Enable the optional LLM by setting an API key and turning it on in `config.yaml`:

```yaml
llm:
  enabled: false     # tutor works fully without it
```

Suggested tests to keep in `tests/`:

- Each rule returns the expected wrong answer on hand-checked problems.
- `probe.py` only returns problems where the correct answer and all five rule answers are pairwise different.
- A correct second answer yields `possible_slip`, never `confirmed`.
- Changing the confidence rating never changes the label.

---

## Limitations and ethics

- **Five rules only.** Wrong answers from other causes (arithmetic slips, misreading) are labelled `uncertain`, not forced into a rule.
- **Single answers are ambiguous.** Rules can coincide, which is why one answer yields only `suspected`.
- **Small pilot.** Results are indicative trends, not evidence of effect size or causation.
- **Self-reported method and confidence** may be inaccurate; both are used as supporting information only.
- **Privacy.** Store pseudonymous student IDs only; obtain consent for the peer pilot and for storing attempts; report aggregate trends rather than identifiable individuals.
