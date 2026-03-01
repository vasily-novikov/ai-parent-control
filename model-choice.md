# Model Choice (POC) - Weighted Comparison

Date: 2026-03-01
Scope: comparison for this project requirements excluding image recognition.
Models: `gpt-5-mini`, `Qwen3.5-122B-A10B`, `Qwen3.5-27B`, `GLM-5`.

## 1) Why Previous Version Was Inconsistent
The previous draft used `gpt-5-mini (medium)` and a different weight profile.
This version is aligned to your split-function view and uses `gpt-5-mini (high)`.

## 2) Function Split (Target)
- Psychology support: `Qwen3.5-122B-A10B > Qwen3.5-27B > GLM-5 > GPT-5 mini (high)`
- Tutor: `GLM-5 > Qwen3.5-27B > Qwen3.5-122B-A10B > GPT-5 mini (high)`
- DE (German): `Qwen3.5-122B-A10B ≈ GPT-5 mini (high) > Qwen3.5-27B > GLM-5`
- Berlin Gymnasium 7th-grade context awareness: `Qwen3.5-122B-A10B > GPT-5 mini (high) > Qwen3.5-27B > GLM-5`

## 3) Input Metrics (Comparison Sites)
Primary source: Artificial Analysis snapshots (2026-03-01).

| Model | Intelligence Index | Output Speed (tok/s) | TTFT (s) | Context (tokens) | Input $/1M | Output $/1M |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| gpt-5-mini (high) | 41 | 75.5 | 94.09 | 400000 | 0.25 | 2.00 |
| Qwen3.5-122B-A10B | 41 | 106.7 | 1.34 | 262000 | 0.40 | 3.20 |
| Qwen3.5-27B | 42 | 99.9 | 1.41 | 262000 | 0.30 | 2.40 |
| GLM-5 | 50 | 67.6 | 1.28 | 200000 | 1.00 | 3.20 |

Scoring notes:
- Blended price = `(3 * input + output) / 4`.
- Min-max normalization is applied per metric within these 4 models.
- DE uses proxy signal (AA multilingual/German ordering + vendor language positioning).
- DE proxy raw values for this run: GPT-5 mini high `81`, Qwen122 `82`, Qwen27 `76`, GLM-5 `62`.

## 4) Weights Used
### Psychology support
- TTFT 45%
- Speed 25%
- DE 10%
- Intelligence 10%
- Price 5%
- Context 5%

### Tutor
- Intelligence 55%
- TTFT 15%
- Speed 10%
- Context 10%
- DE 5%
- Price 5%

### DE communication
- DE 55%
- Context 15%
- TTFT 10%
- Intelligence 15%
- Price 5%
- Speed 0%

### Berlin context awareness
- Context 35%
- DE 20%
- TTFT 15%
- Speed 15%
- Intelligence 10%
- Price 5%

Overall composite:
- Psychology support 30%
- Tutor 30%
- DE 20%
- Berlin context awareness 20%

## 5) Weighted Scores
### Per-function score (0..100)
| Model | Psychology support | Tutor | DE | Berlin context |
| --- | ---: | ---: | ---: | ---: |
| Qwen3.5-122B-A10B | 84.13 | 35.70 | 72.25 | 63.45 |
| Qwen3.5-27B | 79.45 | 40.15 | 59.01 | 57.53 |
| GLM-5 | 55.00 | 70.00 | 25.00 | 25.00 |
| gpt-5-mini (high) | 24.55 | 21.77 | 72.25 | 62.03 |

Per-function ranking now matches the split table exactly (DE is effectively tie for top two).

### Overall weighted score
| Rank | Model | Score |
| --- | --- | ---: |
| 1 | Qwen3.5-122B-A10B | 63.09 |
| 2 | Qwen3.5-27B | 59.19 |
| 3 | GLM-5 | 47.50 |
| 4 | gpt-5-mini (high) | 40.75 |

## 6) Decision for This Project
- If you want one model aligned with all four functions: `Qwen3.5-122B-A10B`.
- If budget/serving simplicity matters more: `Qwen3.5-27B`.
- If tutor-only depth is the priority and DE fit is secondary: `GLM-5`.
- If choosing GPT-5 mini, avoid "high" for latency-sensitive live chat loops.

## 7) Sources
- Artificial Analysis:
  - https://artificialanalysis.ai/models/gpt-5-mini
  - https://artificialanalysis.ai/models/qwen3-5-122b-a10b
  - https://artificialanalysis.ai/models/qwen3-5-27b
  - https://artificialanalysis.ai/models/glm-5
  - https://artificialanalysis.ai/models/multilingual/german
- Qwen model cards:
  - https://huggingface.co/Qwen/Qwen3.5-122B-A10B-FP8
  - https://huggingface.co/Qwen/Qwen3.5-27B
- GLM overview:
  - https://docs.z.ai/guides/overview/overview

## 8) Separate Function: Homework Photo Analysis
Scope for this function only:
- homework photo recognition
- handwritten German text recognition
- neatness/readability analysis

Compared models for this function:
- `qwen/qwen3.5-122b-a10b`
- `openai/gpt-5-mini` (user label: chat-gpt-mini)
- `minimax/minimax-m2.5`
- `moonshotai/kimi-k2.5`

### Capability + German support + pricing
Pricing below is OpenRouter list pricing as of 2026-03-01 and can vary by provider route.

| Model | Image input (required) | OCR / document signals | German support | Pricing (input/output per 1M) | Blended price (3:1) | Fit for homework photo analysis |
| --- | --- | --- | --- | --- | ---: | --- |
| `qwen/qwen3.5-122b-a10b` | Yes | OCRBench `92.1`, OmniDocBench1.5 `89.8`, CharXiv `77.2` | High: model card states support for `201 languages and dialects` (includes German by inference) | `$0.40 / $3.20` | `$1.10` | Strong |
| `moonshotai/kimi-k2.5` | Yes | OCRBench `92.3`, OmniDocBench1.5 `88.8`, CharXiv `77.5` | Medium (inference): multimodal and strong benchmark profile, but no explicit German-language claim in model card | `$0.45 / $2.20` | `$0.89` | Strong |
| `openai/gpt-5-mini` | Yes | OCR proxy from Qwen benchmark table vs GPT-5-mini: OCRBench `82.1`, OmniDocBench1.5 `77.0`, CharXiv `68.6` | Medium-High (inference): GPT-5 family ranks strongly on AA German leaderboard; mini-specific German score not separately published | `$0.25 / $2.00` | `$0.69` | Good fallback |
| `minimax/minimax-m2.5` | No (text only) | N/A | Low/Unknown for this task: docs emphasize text generation and multilingual programming, not image OCR | `$0.295 / $1.20` | `$0.52` | Not suitable (fails image requirement) |

### Function split ranking (this task only)
- Homework photo recognition: `Kimi K2.5 ≈ Qwen3.5-122B-A10B > GPT-5 mini >>> MiniMax M2.5`
- Handwritten German text recognition: `Qwen3.5-122B-A10B ≈ Kimi K2.5 > GPT-5 mini >>> MiniMax M2.5`
- Neatness/readability analysis: `Qwen3.5-122B-A10B ≈ Kimi K2.5 ≈ GPT-5 mini` (rubric quality matters more than model gap), `MiniMax M2.5` not applicable

### Decision for this separate function
- Primary shortlist: `qwen/qwen3.5-122b-a10b`, `moonshotai/kimi-k2.5`
- Cost-managed fallback: `openai/gpt-5-mini`
- Exclude for this function: `minimax/minimax-m2.5` (no image input)

## 9) Sources (Homework Photo Analysis)
- OpenRouter model pages (pricing + model descriptions):
  - https://openrouter.ai/qwen/qwen3.5-122b-a10b
  - https://openrouter.ai/moonshotai/kimi-k2.5
  - https://openrouter.ai/openai/gpt-5-mini
  - https://openrouter.ai/minimax/minimax-m2.5
- Qwen model card (multilingual + OCR/doc benchmarks):
  - https://huggingface.co/Qwen/Qwen3.5-122B-A10B-FP8
- Kimi K2.5 model card (multimodal + OCR/doc benchmarks):
  - https://huggingface.co/moonshotai/Kimi-K2.5
- OpenAI GPT-5 mini model doc (modalities):
  - https://developers.openai.com/api/docs/models/gpt-5-mini
- MiniMax docs (M2.5 under text generation):
  - https://platform.minimax.io/docs/guides/text-generation
  - https://platform.minimax.io/docs/api-reference/api-overview
- Artificial Analysis German leaderboard (family-level German signal for GPT-5):
  - https://artificialanalysis.ai/models/multilingual/german
