# design.md

This file defines technical architecture and implementation design derived from requirements.

## High-Level Architecture
1. **Scheduler / Orchestrator**
   - Runs periodic jobs (every 15 minutes).
   - Handles daily reset and nighttime policies.
   - Handles weekly review cycle.

2. **Usage Collector**
   - Pulls/receives screen-time statistics.
   - Normalizes data into categories (study, streaming, gaming, KiKA, other).

3. **Policy Engine**
   - Evaluates rules:
     - phased base limits,
     - category-weighted usage for entertainment categories,
     - homework-gated KiKA allowance (daytime unlimited after homework completion),
     - daily penalty caps (component caps + total cap),
     - weekly trend-triggered support mode when motivation/involvement declines,
     - deterministic scoring and unlock/lock decisions,
     - penalties and bonuses,
     - unlock conditions,
     - nighttime partial lock policy.

4. **Telegram Bot Interface**
   - Parent + son + bot group notifications (group is primary interface).
   - Son is the primary point of contact for learning dialogs; parent remains observer/supervisor.
   - Receives unlock requests and conversational inputs.
   - Sends questions, reminders, decisions, and remaining time.
   - Uses private thread/DM for sensitive corrective details when possible.
   - Runs daily emotional + school reflection check-in after classes.
   - Suggests 2-3 concrete offline/social activities with friends and rotates options across sports, outdoor, study/social, and creative categories.
   - Supports two input modes:
     - command mode (`/unlock_request`, etc.),
     - free conversation mode when bot is tagged or directly replied-to in group thread.
   - Supports unlock-intent auto-trigger:
     - if son asks for unlock in natural language, map it to unlock flow without requiring slash command.

5. **Learning Verification Agent**
  - Fetches tomorrow schedule from Untis API.
  - Processes subjects strictly sequentially (one subject fully completed before moving to the next).
  - For each subject performs checks:
    - conspect (notes) from previous class + neatness,
    - topic understanding,
    - 5 topic questions,
    - homework presence + revision cooperation + neatness.
  - Produces a separate per-subject result (penalties, bonuses, and short strengths-first feedback) before advancing.
  - Frames notes/topic checks as low-pressure retrieval practice to refresh what was learned and strengthen involvement.
  - Runs short English-Spanish themed vocabulary drill when configured.
  - Uses open-discussion support with iterative rephrasing and starter templates (no fixed attempt limit) before final subject conclusion.
  - Computes penalties, bonuses, and completion status.

6. **Wellbeing and Participation Coach**
   - Asks short daily reflection prompts about class comfort and social climate.
   - Runs a short school small-talk segment daily to surface social context in a low-pressure way.
   - Extracts classmate/friend names from son messages and updates friend-memory records.
   - Tracks participation micro-goals (raise hand at least once; ask or answer once when possible).
   - Converts effort into small bonuses to support confidence recovery.
   - Uses graded exposure for participation anxiety: first raise hand, then short answer, then fuller answer.
   - Encourages social activities, proposes varied friend plans, and awards higher bonuses for activity diversity.
   - Matches suggested activities to known friend interests when confidence is sufficient.

7. **Network Enforcement Adapter (NextDNS API)**
   - Applies/revokes profile rules for categories:
     - streaming blocked/unblocked,
     - gaming blocked/unblocked.

8. **State Store**
   - Persists daily counters, penalties, evaluation results, lock state, and audit logs.

9. **Inference Router (Hybrid Cost Strategy)**
   - Routes requests between deterministic code, `gpt-5-mini`, and `gpt-5`.
   - Target monthly mix: ~`75% gpt-5-mini` / ~`25% gpt-5` (token-weighted).
   - Default to `gpt-5-mini` for routine dialog, rephrasing, reminders, and structured extraction.
   - Escalate to `gpt-5` for high-stakes turns: distress coaching, repeated no-progress, refusal/conflict handling, and weekly narrative summaries.
   - Keep rule math deterministic (no model call): penalty/bonus caps, mastery checks, unlock decisions, and vocab mistake-rate calculations.

10. **Photo Evaluation Subagent (Homework + Class Notes)**
   - Evaluates uploaded photos for `homework` and `class_notes` artifacts using a separate vision flow.
   - Returns compact structured output only (no long image-text dumps into main agent context).
   - Extracts evidence fields: presence, neatness, readability, likely topic, confidence, and issues list.
   - Uses `gpt-5-mini` by default for clear images; escalates to `gpt-5` when image quality/confidence is low.
   - Uses Telegram message/file references as image source in MVP (no separate object-storage dependency).

## AWS Hosting Considerations (Lambda + DynamoDB)
- Recommended deployment stack:
  - `AWS Lambda` for Telegram webhook handlers, scheduled policy jobs, and weekly digest generation.
  - `API Gateway HTTP API` as Telegram webhook ingress to Lambda.
  - `EventBridge Scheduler` for cron-like jobs in `Europe/Berlin` timezone:
    - every 15 minutes usage/policy cycle,
    - daily reset,
    - weekly digest on Sunday at 17:00.
  - `DynamoDB` as primary state and audit store.
- Lambda execution model:
  - hard limit per invocation is 15 minutes; design all handlers to complete well under this.
  - webhook handler should acknowledge quickly and offload longer steps when needed.
  - if future photo processing grows, move long-running work behind an async queue pattern.
- DynamoDB usage guidance:
  - persist operational entities: `daily_policy_state`, `subject_checks`, `friend_registry`, `social_activity_log`, `weekly_review`, `access_state`, and audit logs.
  - use TTL for high-volume transient records (e.g., processed webhook events, verbose routing logs).
  - add query paths (GSI or table layout) for week-over-week aggregations used by Sunday digest.
- Idempotency and safety:
  - deduplicate Telegram updates using `update_id` + `chat_id` keys stored in DynamoDB.
  - deduplicate NextDNS state writes using a deterministic policy-action key to avoid flip-flops and retries causing duplicate actions.
- Media storage:
  - no mandatory `S3` dependency for MVP, because homework/notes evidence is stored as Telegram file/message references.
  - introduce S3 only if independent media retention/export becomes a requirement.

## Data Model (Suggested)
- `daily_usage`:
  - `date`
  - `used_minutes_total`
  - `used_minutes_streaming`
  - `used_minutes_gaming`
  - `used_minutes_kika`
  - `used_minutes_kika_counted` (after homework-gate rules)
  - `used_minutes_other`
  - `used_minutes_entertainment_weighted`
- `daily_policy_state`:
  - `phase_name`
  - `base_minutes` (phase-dependent)
  - `penalty_minutes`
  - `penalty_minutes_capped_total` (int, daily cap applied)
  - `bonus_minutes`
  - `homework_gate_open` (bool)
  - `kika_unlimited_daytime` (bool)
  - `missing_homework_penalty_minutes_today` (int, capped at `30`)
  - `refusal_penalty_minutes_today` (int, capped at `30`)
  - `effective_minutes_allowed`
  - `minutes_remaining`
- `subject_checks`:
  - `date`
  - `subject`
  - `conspect_present` (bool)
  - `conspect_neat` (bool)
  - `topic_identified` (bool)
  - `questions_asked` (int, target 5)
  - `questions_correct` (int)
  - `homework_present` (bool)
  - `homework_correct` (score or bool)
  - `homework_clean` (bool)
  - `homework_photo_used` (bool)
  - `notes_photo_used` (bool)
  - `homework_photo_confidence` (float, optional)
  - `notes_photo_confidence` (float, optional)
  - `homework_revision_requested` (bool)
  - `homework_revision_completed` (bool)
  - `homework_revision_refused` (bool)
  - `homework_refusal_confirmed` (bool, true only after full support sequence)
  - `homework_refusal_reason` (enum: `explicit_no`, `stopped_after_retry`, `other`)
  - `vocab_theme` (string, optional)
  - `vocab_pairs_asked` (int, optional)
  - `vocab_pairs_correct` (int, optional)
  - `vocab_attempts_used` (int, optional)
  - `vocab_mistake_rate_per_attempt` (float[], optional)
  - `hint_used` (bool)
  - `retry_used` (bool)
  - `rephrase_support_count` (int)
  - `discussion_turns_count` (int)
  - `subject_result_finalized` (bool)
  - `positive_feedback_points` (string[])
  - `next_step_message` (string)
  - `penalty_applied`
  - `bonus_applied`
- `photo_evaluations`:
  - `timestamp`
  - `date`
  - `subject`
  - `artifact_type` (enum: `homework`, `class_notes`)
  - `image_ref` (Telegram `file_id` or message reference)
  - `model_used` (enum: `gpt-5-mini`, `gpt-5`)
  - `present_detected` (bool)
  - `neatness_score` (1-5)
  - `readability_score` (1-5)
  - `topic_guess` (string)
  - `confidence` (0.0-1.0)
  - `issues` (string[])
  - `escalated` (bool)
  - `escalation_reason` (string, optional)
- `daily_wellbeing`:
  - `date`
  - `mood_score` (1-5)
  - `class_comfort_score` (1-5)
  - `rude_incidents_reported` (int)
  - `school_small_talk_done` (bool)
  - `names_mentioned` (string[])
  - `notes`
- `friend_registry`:
  - `friend_alias` (string, unique normalized label)
  - `display_name` (string)
  - `name_variants` (string[])
  - `interest_tags` (string[]) (e.g., `football`, `basketball`, `library`, `coding`)
  - `interest_confidence` (0.0-1.0)
  - `last_mentioned_at` (timestamp)
  - `last_activity_with_son_at` (timestamp, optional)
  - `mention_count` (int)
  - `active` (bool)
- `participation_tracker`:
  - `date`
  - `subject`
  - `participation_goal` (e.g., 1 contribution)
  - `participation_attempted` (bool)
  - `participation_success` (bool)
  - `report_source` (enum: `child_self_report`, `parent_observation`, `teacher_feedback`)
  - `confidence_level` (enum: `high`, `medium`, `low`)
  - `conflict_flag` (bool)
  - `bonus_minutes_awarded`
- `social_activity_log`:
  - `date`
  - `activity_name` (e.g., `soccer`, `library_with_friend`)
  - `activity_category` (e.g., `sports`, `outdoor`, `study_social`, `creative`)
  - `with_friends` (bool)
  - `friend_aliases` (string[], optional, normalized nicknames for weekly diversity stats)
  - `completed` (bool)
  - `is_diverse_vs_recent_days` (bool)
  - `has_new_friend_vs_recent_weeks` (bool)
  - `bonus_minutes_awarded`
- `access_state`:
  - `streaming_blocked` (bool)
  - `gaming_blocked` (bool)
  - `kika_blocked` (bool)
  - `reason`
  - `updated_at`
- `weekly_review`:
  - `week_start`
  - `target_minutes`
  - `actual_minutes`
  - `school_goal_completion_rate`
  - `study_engagement_score` (trend)
  - `usage_total_minutes_week`
  - `usage_by_category_week` (streaming/gaming/kika/other)
  - `usage_delta_vs_prev_week`
  - `homework_completion_rate_week`
  - `homework_neatness_rate_week`
  - `homework_refusal_events_week`
  - `homework_photo_low_confidence_events_week`
  - `class_participation_attempts` (count)
  - `class_participation_answers` (count)
  - `social_activities_completed_week` (count)
  - `social_activity_diversity_count` (count)
  - `social_unique_friends_count` (count)
  - `social_top_friend_share_percent` (0-100)
  - `motivation_self_report_trend` (improving/stable/declining)
  - `improvements_summary` (string[])
  - `degradation_summary` (string[])
  - `support_mode_triggered` (bool)
  - `support_mode_reason` (string)
  - `parent_notes`
  - `child_feedback`
- `llm_routing_log`:
  - `timestamp`
  - `flow_id`
  - `step_name`
  - `model_selected` (enum: `deterministic`, `gpt-5-mini`, `gpt-5`)
  - `routing_reason` (e.g., `default_low_risk`, `stress_high`, `no_progress`, `conflict_flag`)
  - `tokens_input`
  - `tokens_output`
  - `estimated_cost_usd`
- `telegram_context`:
  - `chat_id`
  - `thread_id` (optional)
  - `message_id`
  - `sender_role` (enum: `son`, `parent`, `bot`)
  - `is_tagged_message` (bool)
  - `is_reply_to_bot` (bool)
  - `detected_intent` (enum: `unlock_or_policy`, `learning_help`, `emotional_support`, `status`, `other`)
  - `intent_confidence` (float)
  - `auto_mapped_to_unlock_request` (bool)
  - `handled_by_bot` (bool)
- `llm_monthly_budget`:
  - `month`
  - `mini_input_tokens`
  - `mini_output_tokens`
  - `gpt5_input_tokens`
  - `gpt5_output_tokens`
  - `mini_share_percent`
  - `gpt5_share_percent`
  - `target_mix` (default `75/25`)
  - `alerts_triggered`
- `idempotency_keys`:
  - `key` (string, unique)
  - `scope` (enum: `telegram_update`, `policy_action`)
  - `chat_id`
  - `created_at`
  - `expires_at` (TTL timestamp)
  - `status` (enum: `processed`, `ignored_duplicate`, `failed`)

## Timing and Jobs
### Every 15 minutes
1. Collect latest usage statistics.
2. Recompute remaining time.
3. If threshold crossed (e.g., 80%, 100%, overage), send Telegram notifications.
4. Enforce policy through NextDNS profile update.

### Daily (e.g., 00:00 local time)
1. Reset daily base budget to phase-derived allowance (120 minutes when phase target is reached).
2. Load carry-over policy if enabled (optional).
3. Prepare next day learning tasks from Untis schedule.
4. All day-boundary logic uses `Europe/Berlin` timezone.

### After School (e.g., 16:00-18:00 local time)
1. Bot asks "How did today go?" in simple German.
2. Using today's Untis schedule, bot asks 2-4 class-specific prompts:
   - what was covered,
   - where it was hard,
   - any rude interaction/stress.
3. If stress is high, switch evening mode to lighter support and avoid stacked penalties.
4. Run short school small talk (3-10 minutes):
   - ask 2-4 prompts about classes, breaks, and peers ("Who did you spend break with?", "Any plans with friends?"),
   - extract person names from son responses,
   - upsert names into `friend_registry` and attach simple interest tags when explicitly stated,
   - if extracted-name confidence is low, ask one clarifying question before saving.
5. Review class participation with low-conflict logging:
   - ask two short check-ins (`raised_hand?`, `answered_or_asked?`) with options yes/no/unsure,
   - child self-report is accepted as default evidence for same-day bonus,
   - if report conflict appears, set `conflict_flag=true`, keep same-day bonus unchanged, and resolve only in weekly review (no same-day punishment).
6. If he raised his hand at least once, award `+5 min`; if he also gave an answer/question, award additional `+5 min`.
7. Bot suggests 2-3 varied offline activities with friends (e.g., one sports, one outdoor, one study/social), including at least one option not used recently.
   - preference order: match activity to friend interests from `friend_registry` > generic diversified suggestions.
   - sample output style: "Football with Max" + "Library with Leon" + "Bike ride with someone new."
8. If completed social activity is reported, apply social bonus with category-diversity and people-diversity multipliers by policy.

### Weekly (Sunday 17:00, Europe/Berlin)
1. Generate weekly report (usage, school task completion, motivation/involvement trend).
2. Post digest to Telegram group automatically with:
   - time-usage statistics for the week (total + category split),
   - achievements,
   - homework statistics,
   - dedicated social block (completed activities, category diversity, distinct friends involved, comparison vs previous week),
   - explicit comparison vs previous week.
3. Format weekly feedback in fixed order:
   - first improvements,
   - then degradations as constructive next-focus points.
4. Keep digest tone positive and encouraging.
5. If `social_top_friend_share_percent > 70` and `social_activities_completed_week >= 3`, add a supportive suggestion to involve one more friend next week (no penalty).
6. Suggest next phase or keep current phase.
7. Prompt parent-son 15-minute review conversation.
8. Automatic support-mode trigger:
   - if `motivation_self_report_trend=declining` for 2 consecutive weeks, or
   - class participation attempts drop by `>=30%` week-over-week,
   then enable 7-day support mode:
   - freeze phase tightening for one week,
   - reduce non-safety penalties by `50%` (rounded),
   - keep bonuses fully active,
   - set one minimal daily participation micro-goal.

### Night Window Handler (20:00–08:00)
- Force partial lock regardless of remaining time:
  - streaming = blocked,
  - gaming = blocked,
  - KiKA = blocked.
- At 08:00, return control to normal daytime policy logic.

## Unlock Flow (Telegram-Initiated)
1. Son sends unlock request to bot (slash command or natural-language unlock intent).
2. Agent starts learning verification workflow (German by default, simplified phrasing).
   - Inference Router selects `gpt-5-mini` by default for this step.
3. Agent asks short check-in ("ready now / need 10 min / need help") to reduce escalation.
   - If stress score is high, start with easier item first to rebuild momentum.
4. Agent fetches tomorrow schedule via Untis API.
5. For each subject (strictly sequential):
   - finish full evaluation and feedback for current subject before opening the next subject.
   - Ask for previous class conspect.
     - if class-notes photo is provided, call Photo Evaluation Subagent (`artifact_type=class_notes`) and use structured result.
     - If missing, first request a 2-3 sentence recap to refresh memory before scoring.
     - If still missing after recap prompt: apply `-10 minutes` penalty.
     - If present but messy/dirty: apply `-5 minutes` penalty.
     - If neat and readable: apply `+10 minutes` bonus.
     - If son provides a short make-up summary, recover `+5 minutes`.
     - if photo confidence is low (`<0.75`), ask for a clearer photo or continue with oral recap without immediate extra penalty.
   - Determine discussed topic:
     - Try to infer from schedule/material.
     - If unclear, ask son directly.
   - Ask 5 questions on that topic:
     - difficulty-adaptive,
     - framed as refresh and involvement coaching (not an exam),
     - run as open discussion with iterative rephrasing and starter phrases (no fixed attempt cap).
     - evaluate answers using subject-difficulty mastery tiers:
       - Tier H (`Mathematik`, `Physik`, `Chemie`, `Informatik`): `>=3/5` and at least one application question correct,
       - Tier M (`Biologie`, `Geografie/Erdkunde`, `Geschichte`, `Politik/WAT`, `Nawi`): `>=3/5`,
       - Tier L (`Deutsch`, `Englisch`, `Spanisch`, other foreign languages): `>=2/5` with `5/5` attempted,
       - unlisted subjects default to Tier M.
     - keep discussing until an appropriate subject conclusion can be stated clearly.
     - route routine turns to `gpt-5-mini`; escalate to `gpt-5` when `no_progress_after_3_turns` or emotional stress spikes.
   - Check homework:
     - if homework photo is provided, call Photo Evaluation Subagent (`artifact_type=homework`) and use structured result.
     - missing homework -> apply `-15 minutes` penalty per subject.
     - incorrect homework -> not a hard blocker; switch to assisted correction mode.
     - if revision/neatness improvement is requested, run minimum refusal check sequence before any refusal penalty:
       1) give one specific hint,
       2) give one starter sentence/template,
       3) offer short `3-5 min` break,
       4) ask for one retry.
     - if he stays engaged after step 4, continue additional support attempts before considering refusal.
     - define refusal strictly as either:
       - two explicit declines after steps 1-4, or
       - no retry submission after step 4 with explicit message "I won't do it now".
     - if refusal is confirmed by this definition -> apply `-15 minutes` penalty per subject.
     - complete and neat homework -> apply `+10 minutes` bonus.
     - correct and neat homework -> apply additional `+5 minutes` bonus.
     - when all required homework is submitted and no revision refusal is logged, open daytime unlimited KiKA for the day.
     - if homework-photo confidence is low (`<0.75`), request better image or brief oral walkthrough before finalizing penalties.
   - Optional language block:
     - run 6-10 themed English-Spanish word pairs (both directions),
     - compute `mistake_rate = wrong_words / total_words` per attempt,
     - if `mistake_rate > 70%`, repeat with rephrased prompts and short examples,
     - allow up to `5` attempts while this high-mistake condition remains true,
     - stop extra attempts early once `mistake_rate <= 70%`,
     - close with supportive summary of progress and key words to remember.
   - Close the current subject with a separate mini-summary:
     - first state 1-2 concrete things done well,
     - then state one short next-step improvement.
     - use `gpt-5` for final subject summary only when conflict/stress complexity is high; otherwise keep `gpt-5-mini`.
6. After all subjects evaluated:
   - Recalculate `effective_minutes_allowed`.
   - If mandatory conditions are met, unblock streaming/gaming.
   - Keep continuous monitoring and auto-relock when limit is exhausted or night window starts.
7. If son asks for `30 min` to play with friends:
   - allow one-time social pass if question routine is complete and homework is done,
   - log decision and update remaining-time counters transparently.

## Group Conversation Flow (Tag-Initiated)
1. Incoming group message is accepted by bot if:
   - it is a supported slash command, or
   - bot is explicitly tagged, or
   - message is a direct reply to bot.
2. If accepted, classify intent:
   - `unlock_or_policy`, `learning_help`, `emotional_support`, `status`, `other`.
   - for son messages, detect natural-language unlock intent (e.g., "please unlock", "kannst du entsperren", "puedo desbloquear?").
   - independently run name/entity extraction on son free-text and update `friend_registry` when confidence is high enough.
3. Route intent:
   - `unlock_or_policy` -> unlock flow,
   - `learning_help`/`emotional_support` -> conversational coaching flow,
   - `status` -> deterministic summary reply.
   - if unlock-intent confidence is low/ambiguous, ask one short clarification before routing.
4. Default responder target is son; parent mentions are informational unless policy conflict/safety issue appears.
5. Ignore unrelated group chatter to reduce noise and cost.


## API Integration Notes
### Untis API
- Inputs: student/class identifier, target date (tomorrow).
- Outputs required:
  - subjects,
  - class times,
  - lesson topics (if available),
  - homework metadata (if available).

### NextDNS API
- Maintain at least three profile states:
  - `restricted_night` (streaming+gaming+KiKA blocked),
  - `learning_mode` (entertainment blocked, educational allowed),
  - `normal` (daytime policy-driven).
- Use idempotent updates to avoid frequent flip-flops.
- Apply all time-window decisions in `Europe/Berlin` timezone.

### Telegram Bot API
- Commands (suggested):
  - `/status`
  - `/unlock_request`
  - `/remaining_time`
  - `/weekly_report`
  - `/focus_mode`
  - `/activities`
  - `/friends`
  - `/friend_play_30`
  - `/vocab_theme`
  - `/help`
- Group handling rules:
  - handle tagged free-text and replies-to-bot in addition to slash commands,
  - auto-map son's natural-language unlock requests to the same path as `/unlock_request`,
  - prefer responding in-thread to keep group readable,
  - use Telegram `file_id`/message references for homework and notes photo processing.

### OpenAI API (Hybrid Routing)
- Primary models:
  - default: `gpt-5-mini`,
  - escalation: `gpt-5`.
- Deterministic-first boundary:
  - run policy computation in code before any model call where possible.
- Escalation triggers to `gpt-5`:
  - `stress_high`,
  - `refusal_risk`,
  - `conflict_flag=true`,
  - `no_progress_after_3_turns`,
  - weekly narrative generation,
  - `photo_confidence_low` for homework/notes image evaluation.
- Mix target and monitoring:
  - keep monthly token mix near `75% mini / 25% gpt-5`,
  - raise alert if `gpt-5` share exceeds `35%`,
  - auto-shift low-risk flows back to mini when over target.

### Photo Evaluation Subagent API
- Tool name (suggested): `evaluate_learning_photo`.
- Input:
  - `subject`,
  - `artifact_type` (`homework` or `class_notes`),
  - `image_ref` (Telegram `file_id` or message reference),
  - `language_hint` (optional).
- Output (strict JSON):
  - `present_detected`,
  - `neatness_score`,
  - `readability_score`,
  - `topic_guess`,
  - `confidence`,
  - `issues`,
  - `needs_escalation`.
- Context hygiene:
  - store full photo analysis in `photo_evaluations`,
  - pass only compact fields into main conversation context.

## Safety / Reliability
- Fail closed for restricted categories if policy engine is unavailable during night window.
- Rate-limit enforcement calls to NextDNS.
- Keep audit log for every decision:
  - input data,
  - rules triggered,
  - API actions taken.
- If `gpt-5` is unavailable, fall back to `gpt-5-mini` plus conservative deterministic policy decisions.
- If monthly cost alert is triggered, preserve `gpt-5` for high-stakes turns and downgrade routine turns to `gpt-5-mini`.
- Do not apply photo-based penalties when photo confidence is low; require clearer photo or oral fallback first.
- Keep Telegram photo references plus structured photo-eval outputs in audit logs for dispute resolution.
- Do not process non-tagged unrelated group messages (unless they are supported commands).
- Parent override:
  - Not available in the current implementation.
  - Access decisions follow policy engine output only.
