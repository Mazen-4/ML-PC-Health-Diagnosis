# Contributing to MLDiag

## Team Roles & Ownership

| Area | Owner | Description |
|------|-------|-------------|
| `collectors/` | **Person B** | All OS sensor readers (WMI, sysfs, psutil) |
| `training/` | **Person A** | All model training notebooks and scripts |
| `models/` | **Person A** | Serialised `.joblib` model files |
| `data/` | **Person A** | Backblaze data and self-collected sensor CSVs |
| `xai/` | **Person A** | SHAP explainer wrappers |
| `rules/` | **Person B** | Rule engine (disk space, updates, startup bloat) |
| `gui/` | **Person B** | PyQt5 interface and PDF export |
| `utils/` | **Both** | Shared utilities — agree before adding |
| `tests/` | **Both** | Each person writes tests for their own modules |
| `main.py` | **Person B** | Entry point and pipeline wiring |

When editing outside your own area, open a PR and request a review from the other person before merging.

---

## Module Output Contract

Every inference module **must** return a Python dict matching this schema exactly.
Person B's GUI and Person A's models both depend on this — do not change it without
notifying both sides and updating this file.

```python
{
    "module": str,          # "storage" | "battery" | "cpu" | "ram" | "fan"
    "status": str,          # "healthy" | "monitor" | "warning" | "critical"
    "confidence": float,    # 0.0 – 1.0  (use -1.0 for rule-based modules)
    "top_features": [       # exactly 2 items; empty list if status is "healthy"
        {
            "name": str,        # human-readable feature name
            "shap_value": float,
            "current_value": float | str
        }
    ],
    "explanation": str,     # plain-English sentence shown in GUI
    "recommendation": str,  # actionable instruction for the user
    "raw_data": dict        # full sensor readings for the PDF export; can be {}
}
```

**Status → display colour mapping (Person B enforces this in the GUI):**

| Status | Colour | Meaning |
|--------|--------|---------|
| `healthy` | Green | No action needed |
| `monitor` | Blue | Worth watching; re-scan in 30 days |
| `warning` | Amber | Action recommended soon |
| `critical` | Red | Act immediately |

---

## Environment Setup

### Prerequisites
- [Anaconda or Miniconda](https://docs.conda.io/en/latest/miniconda.html)
- Git

### First-time setup

```bash
git clone https://github.com/<your-org>/mldiag.git
cd mldiag
conda env create -f environment.yml
conda activate mldiag
```

### Adding a new dependency

```bash
conda activate mldiag
conda install <package>          # prefer conda over pip when available
conda env export --no-builds > environment.yml
git add environment.yml
git commit -m "chore: add <package> to environment"
```

Never commit a manual edit to `environment.yml` — always regenerate it with `conda env export`.

---

## Branching Strategy

```
main          ← always stable; protected branch
dev           ← integration branch; both merge feature branches here
feature/A-*   ← Person A's branches  (e.g. feature/A-hdd-classifier)
feature/B-*   ← Person B's branches  (e.g. feature/B-smart-reader)
```

- **Never push directly to `main` or `dev`.**
- Open a PR into `dev`; the other person reviews and approves before merge.
- `dev` → `main` merges happen at phase gates only (both agree it's stable).

---

## Commit Message Format

```
<type>(<scope>): <short description>

type  : feat | fix | chore | docs | test | refactor
scope : storage | battery | cpu | ram | fan | gui | xai | rules | utils | ci
```

Examples:
```
feat(storage): add XGBoost inference wrapper
fix(cpu): handle missing base clock on AMD chips
chore(ci): add flake8 lint step
docs(contributing): update module output contract
```

---

## Code Style

- Formatter: **Black** (`black .` before every commit)
- Linter: **flake8** (max line length 100)
- Type hints on all public functions — use `dict`, `list`, `str`, `float`, not `Dict`, `List` etc. (Python 3.11)
- Docstrings on every public function (one-line is fine for simple helpers)

A pre-commit hook is configured in `.pre-commit-config.yaml`. Install it once:

```bash
pip install pre-commit
pre-commit install
```

---

## Testing

- Test file location: `tests/<module_name>_test.py`
- Run all tests: `pytest tests/`
- Each module must have at minimum:
  - One test with healthy dummy input → assert `status == "healthy"`
  - One test with degraded dummy input → assert `status != "healthy"`
  - One test that the output dict contains all required contract keys

Person A mocks the sensor collectors; Person B mocks the model outputs.
Neither side should have a hard dependency on the other to run their tests.

---

## Acceptance Criteria (per module)

A module is considered **done** when it passes its own tests AND meets the target below
on the held-out test set. These are minimums, not targets to optimise endlessly.

| Module | Metric | Minimum |
|--------|--------|---------|
| Storage (HDD) | F1 on failure class | ≥ 0.70 |
| Battery SoH | F1 on degraded + replace classes | ≥ 0.75 |
| CPU Throttling | Anomaly detection recall | ≥ 0.80 |
| RAM Pressure | F1 on pressure + bottleneck classes | ≥ 0.72 |
| Fan Health | Precision on fault class | ≥ 0.78 |

If a module cannot reach its minimum after reasonable tuning, open a discussion issue
before changing the criteria — don't quietly lower the bar.

---

## File Naming Conventions

| Content | Convention | Example |
|---------|-----------|---------|
| Collector scripts | `<component>_reader.py` | `battery_reader.py` |
| Training notebooks | `<component>_training.ipynb` | `hdd_training.ipynb` |
| Trained models | `<component>_model.joblib` | `storage_model.joblib` |
| Self-collected data | `<component>_<session>_<machine_id>.csv` | `cpu_load_m03.csv` |

---

## What Belongs in the Repo vs. Not

| Include | Exclude |
|---------|---------|
| `environment.yml` | Raw Backblaze CSVs (too large) |
| `.joblib` model files | Any file > 50 MB |
| Self-collected sensor CSVs | `__pycache__/`, `.conda/` |
| SHAP explainer wrappers | Personally identifiable machine info |
| GUI source + assets | Generated PDFs and reports |

Add a `.gitignore` entry for anything in the Exclude column.

---

## Questions & Decisions Log

Use GitHub Issues with the label `decision` for any architectural question that affects
both sides. Do not resolve these in chat — write the conclusion as a comment on the
issue so there is a permanent record.
