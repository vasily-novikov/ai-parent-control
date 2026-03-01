# Scoring Sheet Template (Markdown)

Use this table for each evaluation run.

| Model | JSON only | JSON parseable | Schema complete | Math correct | OCR fidelity (0-10) | Neatness quality (0-10) | Formal compliance quality (0-10) | Actionability (0-10) | Confidence calibration (0-10) | Overall (0-10) | Strict (0-10) | Notes |
|---|---|---|---|---|---:|---:|---:|---:|---:|---:|---:|---|
| model-a | Pass/Fail | Pass/Fail | Pass/Fail | Pass/Fail | 0 | 0 | 0 | 0 | 0 | 0.0 | 0.0 | short comments |
| model-b | Pass/Fail | Pass/Fail | Pass/Fail | Pass/Fail | 0 | 0 | 0 | 0 | 0 | 0.0 | 0.0 | short comments |
| model-c | Pass/Fail | Pass/Fail | Pass/Fail | Pass/Fail | 0 | 0 | 0 | 0 | 0 | 0.0 | 0.0 | short comments |

Scoring:
- `Overall (0-10)` = average of the five quality columns.
- `Strict (0-10)`:
  - start with `Overall`,
  - if any hard check fails, cap at `6.0`,
  - if `JSON only` fails, cap at `5.0`.

