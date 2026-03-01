# Notes Analysis Evaluation Rubric

Purpose: compare model outputs for class-note photo analysis in a repeatable way.

## Inputs per run
- `input_file` (photo or PDF)
- `prompt_file` (analysis prompt)
- `model_output_file` (raw model response)

## Hard Requirement Checks (Pass/Fail)
1. `json_only_output`: response must contain only JSON (no preamble/reasoning text).
2. `json_parseable`: JSON parses successfully.
3. `schema_complete`: required top-level and nested keys exist.
4. `weighted_math_correct`: `neatness.percent` and `formal_compliance_berlin_gymnasium.percent` match rubric math.

## Quality Scores (0-10)
Use this scale for each criterion:
- `0-2`: poor / mostly wrong
- `3-4`: weak
- `5-6`: acceptable
- `7-8`: good
- `9-10`: very good / excellent

### Criteria
1. `ocr_content_fidelity_0_10`
- How accurate extracted content is versus the real note image.
- Penalize hallucinated terms/claims not visible in input.

2. `neatness_assessment_quality_0_10`
- Are neatness judgments grounded in visible evidence?
- Are positives and improvements specific and fair?

3. `formal_compliance_quality_0_10`
- Does evaluation correctly apply Berlin Gymnasium-style formal criteria?
- Are assumptions explicit when school-specific rules are unknown?

4. `actionability_0_10`
- Are improvement suggestions concrete, practical, and student-appropriate?

5. `confidence_calibration_0_10`
- Do confidence values match uncertainty and image quality?
- Penalize overconfidence when OCR is clearly uncertain.

## Recommended Aggregation
- `overall_quality_0_10` = average of the 5 quality criteria.
- Keep hard checks separate (do not hide failures inside average).

Optional strict score:
- `strict_score_0_10` = `overall_quality_0_10`
- If any hard check fails, cap `strict_score_0_10` at `6.0`.
- If `json_only_output` fails, cap `strict_score_0_10` at `5.0`.

## Report Template (Markdown)
| Model | JSON only | JSON parseable | Schema complete | Math correct | OCR fidelity | Neatness quality | Formal compliance quality | Actionability | Confidence calibration | Overall (0-10) | Notes |
|---|---|---|---|---|---:|---:|---:|---:|---:|---:|---|
| model-a | Pass/Fail | Pass/Fail | Pass/Fail | Pass/Fail | 0-10 | 0-10 | 0-10 | 0-10 | 0-10 | 0-10 | short comments |

## Current Run Example (Readable Table)
| Model | JSON only | JSON parseable | Schema complete | Math correct | OCR fidelity | Neatness quality | Formal compliance quality | Actionability | Confidence calibration | Overall (0-10) | Strict (0-10) | Notes |
|---|---|---|---|---|---:|---:|---:|---:|---:|---:|---:|---|
| chatgpt-5-mini | Fail | Pass | Pass | Pass | 9 | 9 | 8 | 8 | 6 | 8.0 | 5.0 | Best fidelity on this sample; violated JSON-only rule |
| qwen3.5-122b-a10b | Fail | Pass | Pass | Pass | 6 | 6 | 6 | 6 | 7 | 6.2 | 5.0 | More OCR distortions and weaker grounding |
| kimi-k2.5 | Fail | Pass | Pass | Pass | 7 | 8 | 7 | 9 | 4 | 7.0 | 5.0 | Very actionable feedback; overconfident OCR/confidence values |
| chatgpt5.2 | Pass | Fail | Fail | Fail | 8 | 5 | 4 | 7 | 5 | 5.8 | 5.8 | JSON invalid (unescaped quotes); subscores use point-scale style, not 0-100 weighted schema |
