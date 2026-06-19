# Run Output - Day 17

## verify.py

Command:

```powershell
.\.venv\Scripts\python.exe verify.py
```

Result: `16/16 checks - ALL PASS`

## pytest

Command:

```powershell
.\.venv\Scripts\python.exe -m pytest -q
```

Result: `18 passed`

Note: pytest emitted one cache warning because the sandbox could not create `.pytest_cache`; tests still passed.

## dbt build

Command:

```powershell
cd dbt_project
..\.venv-dbt\Scripts\dbt.exe build
```

Result: `PASS=11 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=11`

## generated artifacts

- `quarantine.csv`
- `datasets/eval_golden.jsonl`
- `datasets/preference_pairs.jsonl`
- `bonus/DESIGN.md`
