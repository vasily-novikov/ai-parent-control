# Model Choice

Date: 2026-03-01  
Scope: compact decision doc for this project.

## Final Recommendation
- Main conversation model: `openai/gpt-5-mini`
- Default reasoning level: `medium` (use `high` only for difficult tutoring turns)
- Homework photo analysis: `openai/gpt-5-mini` primary, fallback to `moonshotai/kimi-k2.5`, then `qwen/qwen3.5-122b-a10b`
- Exclude for photo analysis: `minimax/minimax-m2.5` (no image input)

## Why This Choice
- In our repo sample evaluation, `gpt-5-mini` produced the best overall photo-feedback quality.
- `moonshotai/kimi-k2.5` and `qwen/qwen3.5-122b-a10b` are strong OCR/vision backups.
- In current human review, Kimi feedback style was preferred over Qwen (more specific and less rigid/formal).
- `gpt-5-mini` also has a strong cost/quality profile for continuous Telegram conversation loops.

## Main Conversation Models (Text-First)
| Model | Strengths | Tradeoffs | Recommended role |
|---|---|---|---|
| `openai/gpt-5-mini` | Best practical balance for this project; strong German coaching style in tests; low cost | Can add non-JSON text unless schema mode is enforced | **Default** |
| `qwen/qwen3.5-122b-a10b` | Very fast TTFT; strong reasoning benchmarks | Higher ops/cost complexity if self-hosted | Fallback / specialist |
| `qwen/qwen3.5-27b` | Good cost/latency tradeoff | Lower ceiling vs 122B | Budget fallback |
| `z-ai/glm-5` | High reasoning benchmark signal | Weaker fit for German-first family flow in our weighting | Tutor-only alternative |

## Homework Photo Analysis Models (Vision/OCR)
Pricing is indicative (OpenRouter list, 2026-03-01).

| Model | Image input | OCR/Doc signal | German support (practical) | Price in/out per 1M | Decision |
|---|---|---|---|---|---|
| `openai/gpt-5-mini` | Yes | Good in our sample; lower published OCR proxy vs Qwen/Kimi | Good | `$0.25 / $2.00` | **Primary** |
| `moonshotai/kimi-k2.5` | Yes | OCRBench `92.3`, OmniDocBench1.5 `88.8`, CharXiv `77.5` | Medium-Good | `$0.45 / $2.20` | **Fallback #1** |
| `qwen/qwen3.5-122b-a10b` | Yes | OCRBench `92.1`, OmniDocBench1.5 `89.8`, CharXiv `77.2` | Good | `$0.40 / $3.20` | Fallback #2 |
| `minimax/minimax-m2.5` | No | N/A | N/A for this function | `$0.295 / $1.20` | Exclude |

## Routing Policy
1. Use `openai/gpt-5-mini` for all normal chat and note-photo analysis calls.
2. Enforce strict JSON schema mode for scoring/assessment outputs.
3. If OCR confidence is low or schema fails after one retry, reroute to `moonshotai/kimi-k2.5`.
4. If needed, second fallback is `qwen/qwen3.5-122b-a10b`.

## Sources
- Artificial Analysis:
  - https://artificialanalysis.ai/models/gpt-5-mini
  - https://artificialanalysis.ai/models/qwen3-5-122b-a10b
  - https://artificialanalysis.ai/models/qwen3-5-27b
  - https://artificialanalysis.ai/models/glm-5
  - https://artificialanalysis.ai/models/multilingual/german
- Model cards / docs:
  - https://huggingface.co/Qwen/Qwen3.5-122B-A10B-FP8
  - https://huggingface.co/moonshotai/Kimi-K2.5
  - https://developers.openai.com/api/docs/models/gpt-5-mini
  - https://platform.minimax.io/docs/guides/text-generation
  - https://platform.minimax.io/docs/api-reference/api-overview
- Pricing pages:
  - https://openrouter.ai/openai/gpt-5-mini
  - https://openrouter.ai/qwen/qwen3.5-122b-a10b
  - https://openrouter.ai/moonshotai/kimi-k2.5
  - https://openrouter.ai/minimax/minimax-m2.5
