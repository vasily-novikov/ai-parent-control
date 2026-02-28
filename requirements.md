# requirements.md

This file defines product, behavioral, and policy requirements.

## System Goal
Parental control system for a 13-year-old son that:
- Monitors screen-time usage every 15 minutes.
- Sends warning notifications to a Telegram group chat shared with parent, son, and bot.
- Can enforce internet access restrictions through NextDNS API.
- Uses daily learning checks to build long-term motivation and involvement in study and social life before granting additional access.
- Prioritizes engagement, retrieval, and consistency over short-term homework correctness.

## Family Context Summary (Added Feb 2026)
- Student profile:
  - 7th grade at Herder Gymnasium (Berlin Westend), MINT-focused.
- Migration/adaptation context:
  - Moved to Germany 2 years and 3 months ago.
  - Adaptation period was difficult.
  - German has improved significantly but still limits learning speed/depth.
- Academic status:
  - Last year: many grade 4s, no 5s; probation year passed.
  - Current challenge: deeper physics learning demands in 7th grade.
- Current media usage baseline:
  - ~32 hours/week YouTube.
  - ~10 hours/week games.
  - Total entertainment screen time ~42 hours/week.
- Parenting context:
  - During adaptation, screen time was intentionally less restricted to reduce pressure.
  - Current goal: improve learning involvement while preserving trust/relationship.
- Content exception request:
  - Keep KiKA (ARD kids app) accessible as German-language educational content.
  - Example: "Checker Tobi" is both motivating and school-relevant.
- School climate and social context:
  - 2-3 classmates are often rude (not targeted bullying only at him; broader behavior issue).
  - Class teacher is already involved and trying to improve class atmosphere.
  - Son has friends and is not socially isolated.
  - However, he feels uncomfortable in class climate overall.
- Participation/marks context:
  - Due to language load and class atmosphere, he rarely raises his hand.
  - Lower classroom participation reduces marks (important in German school grading).
- Identity and motivation shift after migration:
  - In Russian school he had top grades and strong social confidence.
  - In Germany he experiences a lower-status academic role and reduced self-belief.
  - Current risk: "I cannot be truly successful here because I am not a native speaker."
- New behavior priorities (added Feb 2026):
  - Neatly written class notes should be rewarded; missing notes should be penalized.
  - Homework neatness should be explicitly rewarded/punished.
  - Homework completion is mandatory; correctness alone is not a hard blocker.
  - Each subject must be processed sequentially and evaluated separately.
  - Topic questions and notes checks are for involvement and memory refresh, not short-term perfection grading.
  - Refusal to correct or improve homework neatness after support should be penalized (`-15 min` per subject).
  - Correct and neat homework should receive explicit bonuses.
  - Social in-person activities with friends should be encouraged (soccer highlighted).
  - Diverse social activities with friends should receive stronger rewards than repeating one activity.
  - Daily communication should include explicit psychological support and confidence-building.
  - English-Spanish themed vocabulary training should be part of the routine.

## Core Rules
- Long-term target for entertainment screen time: **2 hours/day**.
- Practical health target for this case:
  - `~2h/day` average entertainment is a realistic and healthy target for a 13-year-old in this context.
  - Avoid pushing lower immediately; only reduce below 2h/day if sleep, mood, and school progress stay stable.
- Use gradual reduction from current baseline (~42h/week) to avoid conflict and rebound:
  - Weeks 1-2: max 35h/week entertainment.
  - Weeks 3-4: max 28h/week.
  - Weeks 5-6: max 21h/week.
  - Weeks 7-8: max 14h/week (2h/day target reached).
- Night restriction window: **20:00 to 08:00** (local time).
- System timezone: **Europe/Berlin** (`CET/CEST`, with DST).
- Night restriction is **partial**, not full:
  - Block streaming services.
  - Block gaming services.
  - Allow educational/basic internet.
- KiKA policy:
  - KiKA is whitelisted as educational German content.
  - If all required daily homework is submitted and there is no refusal to revise/neaten after support, KiKA is unlimited during daytime and excluded from entertainment budget counting.
  - If homework is not complete yet, KiKA counts at `0.5x` with cap `60 min/day`.
  - KiKA is still blocked during 20:00-08:00 for sleep hygiene.
- Son can request unlock via Telegram bot.
- Telegram interaction mode:
  - Group members: parent + son + bot.
  - Son is the primary point of contact for learning dialogs.
  - Bot must support both command mode (e.g., `/unlock_request`) and free conversation when tagged in the group.
  - If son expresses unlock intent in free speech (German/English/Spanish), bot must auto-switch to unlock-request flow.
  - Bot should run a short daily school small-talk check-in (low pressure, 3-10 minutes).
  - If classmate/friend names appear in son messages, bot should extract and track those names.
  - Bot should maintain a friend registry with known interests to support personalized activity suggestions.
  - Social suggestions should be friend-specific when possible (example: suggest soccer with a friend who likes football).
  - In MVP, homework/notes photo evidence uses Telegram message history and file references (no separate object storage dependency).
- Weekly report schedule:
  - Every Sunday at `17:00` (`Europe/Berlin`), bot posts weekly statistics in the Telegram group.
- Agent communication language preference:
  - Preferred: **German**.
  - Allowed fallback: **English** or **Spanish**.
  - Prohibited: **Russian**.

## Psychological Design Principles
- Relationship first:
  - System language must be neutral and respectful (no shaming).
  - Focus on coaching and routines, not punishment loops.
  - Feedback order is fixed: first name specific strengths, then one concrete next step.
  - Correct language softly: first validate meaning, then offer 1 short correction.
  - Validate emotions and effort before any correction ("I see this was hard today").
- Predictability and fairness:
  - Same rules every day.
  - Visible counters and reasons for every lock/unlock.
- Motivation balance:
  - Combine penalties with bonuses for effort and improvement.
  - Target at least 3 positive feedback events for each corrective message.
  - Cap cumulative refusal penalties to avoid punishment stacking spirals.
  - Optimize for sustained weekly engagement trends, not one-day correctness spikes.
- Autonomy support:
  - Son can choose order of school tasks and unlock timing windows.
  - In current implementation, parent override is disabled to keep rules predictable.
- Adaptation-aware communication:
  - German-first, but with simpler language and clarifications.
  - If input quality is low due to language barriers, give hints before penalty.
  - If he cannot answer, provide guided help (hint ladder + starter phrase) before scoring.
  - For topic discussion, do not enforce a fixed attempt limit.
  - Help with rephrasing both the question and his answer until understanding is reached.
  - Keep topic work as an open discussion that ends with an appropriate short conclusion.
  - Before any refusal penalty, run a minimum support sequence: hint -> starter sentence -> 3-5 minute break -> retry; continue with additional support attempts if he stays engaged.
  - Grammar coaching should stay low-pressure and private where possible.
- Self-efficacy rebuilding:
  - Track and highlight weekly evidence of progress ("you improved in X").
  - Frame setbacks as adaptation steps, not fixed ability.
  - Add one short confidence statement daily tied to evidence ("you solved this faster than last week").
- School-climate sensitivity:
  - Do not interpret low class participation as laziness by default.
  - Include stress/comfort check before learning penalties.
  - If class stress is high, switch to supportive mode first (short task + encouragement) before penalties.


## Penalty and Bonus Logic (Balanced)
- Missing conspect/notes (per subject): `-10 min`.
- Conspect present but messy/dirty (per subject): `-5 min`.
- Neat, readable conspect (per subject): `+10 min`.
- Homework unclean (per subject): `-5 min`.
- Refusal to correct or improve homework neatness after support (per subject): `-15 min`.
- Refusal definition:
  - penalty only if refusal is confirmed after full support sequence (hint -> starter sentence -> short break -> retry).
  - confusion, language errors, or slow response are not refusal by themselves.
- Daily guardrail:
  - total penalties across all categories are capped at `-45 min/day`.
  - missing-homework penalties are capped at `-30 min/day` total.
  - refusal penalties are capped at `-30 min/day` total (`max 2` refusal penalties per day).
  - in weekly support mode, apply the `50%` non-safety penalty reduction before daily caps.
  - no same-day rollback of awarded participation bonuses due to evidence disputes.
- Homework complete and neat (per subject): `+10 min`.
- Homework correct and neat (per subject): additional `+5 min`.
- KiKA daytime rule:
  - all required homework submitted and no revision refusal -> unlimited KiKA during daytime (not counted against entertainment limit),
  - homework incomplete -> KiKA remains at `0.5x` with cap `60 min/day`.
- Mastery thresholds for 5 topic questions (7th-grade MINT Gymnasium, Berlin):
  - Tier H (high difficulty MINT subjects): `Mathematik`, `Physik`, `Chemie`, `Informatik`.
    - mastery threshold = `>=3/5` correct AND at least one application-style question correct.
  - Tier M (medium difficulty subjects): `Biologie`, `Geografie/Erdkunde`, `Geschichte`, `Politik/WAT`, `Nawi`.
    - mastery threshold = `>=3/5` correct.
  - Tier L (language-load subjects): `Deutsch`, `Englisch`, `Spanisch`, other foreign languages.
    - mastery threshold = `>=2/5` correct with all `5/5` attempted.
  - Default mapping for unlisted subjects: Tier M.
- Mastery handling rule:
  - if subject threshold is not met after open discussion + rephrasing support -> guided support mode and reduced question bonus for that subject.
  - homework correctness alone still does not block unlock.
- Notes/topic checks intent:
  - assess engagement and memory refresh rather than punish mistakes.
  - effortful participation in recall counts positively even with partial correctness.
- Bonuses:
  - All 5 questions attempted for subject: `+5 min`.
  - Subject mastery threshold met (by subject difficulty tier): `+5 min` (bonus only, not a blocker).
  - High achievement (`5/5`) in Tier H subject: additional `+2 min`.
  - Subject feedback quality:
    - each subject summary must include at least one explicit positive observation.
  - All homework complete and clean for the day: `+10 min`.
  - Raised hand in at least one class: `+5 min`.
  - Gave at least one class answer/question after raising hand: `+5 min`.
  - Honest participation self-report with `unsure` (when anxious or uncertain): `+2 min` reflection bonus (once/day).
  - Soccer activity with peers: `+15 min`.
  - Other social activity with friends (e.g., library): `+10 min`.
  - Diversity bonus for friend activity category different from recent days: `+10 min`.
  - Weekly variety bonus for 3+ distinct friend activity categories: `+15 min`.
  - Weekly people-diversity bonus for social activities with at least 2 distinct friends: `+10 min`.
  - Extra people-diversity bonus for social activities with at least 3 distinct friends: additional `+5 min`.
  - One-time `30 min` friend play pass after successful question routine + homework complete.
  - English-Spanish themed vocabulary drill completed: `+5 min`.
  - Clear evidence of improved German expression vs previous week: `+5 min`.
- Optional anti-rebellion floor:
  - Minimum `45 min/day` entertainment access if child engages with learning flow respectfully.

Formula:
- `effective_minutes_allowed = max(0, base_minutes - total_penalty_minutes + total_bonus_minutes)`
- `total_penalty_minutes` must apply component caps first (`missing_homework<=30`, `refusal<=30`) and then overall cap (`<=45`).
- in support mode, reduce non-safety penalties by `50%` before applying caps.

## Notification Strategy (Telegram)
- Informational:
  - daily budget at start of day,
  - current remaining minutes,
  - learning check progress,
  - daily school small-talk prompt,
  - one suggested offline/social activity for today.
- Warning:
  - at 80% usage,
  - at 100% usage,
  - when penalties applied.
- Positive reinforcement:
  - per subject, lead with explicit praise for what was done well before correction.
  - when tasks are completed,
  - when improvement vs last week is detected,
  - when self-initiated study starts on time.
  - when classroom participation effort occurs (raise hand, even if imperfect result).
  - when he answers or asks a question in class discussion.
  - when notes/homework are neat and readable.
  - when social activity with friends is completed.
  - when a new friend activity type is tried (rewarding variety).
- Enforcement:
  - lock applied (reason + duration/context),
  - unlock granted (new remaining time),
  - relock due to night window.
- Group behavior:
  - process explicit commands always,
  - process free-text only when bot is tagged or directly replied-to in group thread,
  - map son's natural-language unlock requests to unlock flow even without slash command,
  - keep unrelated group chatter ignored.
- Weekly digest post (Sunday `17:00`, `Europe/Berlin`):
  - include weekly time-usage statistics (total, category split, trend vs previous week),
  - include achievements (study, participation, social activities, language/vocab),
  - include homework statistics (completion, neatness, revision cooperation/refusal, photo-confidence notes if relevant),
  - include a dedicated social section: completed social activities, activity-category diversity, and number of distinct friends involved,
  - explicitly highlight social progress vs previous week and one suggested next social step,
  - compare to previous week explicitly.
- Weekly feedback style:
  - always present improvements first,
  - then present degradations as constructive next-focus points,
  - if the same friend dominates most social activities, give a supportive nudge to involve one additional friend next week (no penalty),
  - keep tone positive and encouraging.

## Friend Registry and Social Matching
- Maintain a lightweight friend registry from ongoing conversation:
  - store friend names/aliases mentioned in daily school talk or free chat,
  - store inferred interests (e.g., football, gaming, library, coding),
  - update confidence over time when a name-interest pair repeats.
- Registry behavior:
  - never punish for missing/uncertain friend data,
  - ask short clarification when confidence is low ("Does Max also like basketball?").
- Daily suggestion behavior:
  - when suggesting social activity, prefer mapping activity + friend interest (e.g., sports with sports friend, library with study-oriented friend),
  - rotate across both activity categories and people for diversity.

## Language Policy
- Default bot output language: German (simple, school-appropriate phrasing).
- If son answers in English/Spanish, continue there as needed.
- Bot should not use Russian; if Russian input appears, reply in German and ask for German/English/Spanish reformulation.
- In German conversations, apply soft grammar and syntax correction:
  - answer content first,
  - explicitly acknowledge what he did well,
  - then provide one short corrected version ("You can say it like this: ..."),
  - avoid negative scoring for grammar mistakes alone.
- If he struggles to answer a question:
  - provide iterative hints and rephrasing support with no fixed limit,
  - allow him to complete the final answer in his own words,
  - end with a short conclusion summarizing the final appropriate understanding.
- English-Spanish training mode:
  - bot asks themed vocabulary pairs (school topic, daily life, sports, etc.),
  - alternate EN->ES and ES->EN prompts,
  - evaluate mistakes per attempt (`mistake_rate = wrong / total_words`),
  - if `mistake_rate > 70%` in an attempt, start the next attempt (with rephrased prompts and quick examples),
  - allow up to `5` attempts under this high-mistake condition,
  - if `mistake_rate <= 70%`, stop extra attempts and close with supportive summary.


## Recommendations for This Family Context
- Start with a collaborative launch conversation:
  - explain goals as "more success and less stress at school", not "punishment for screens".
- Use phased reduction, not immediate drop to 2h/day:
  - from ~42h/week to 14h/week over ~8 weeks is strict but realistic.
- Protect adaptation needs:
  - keep German-supportive content (KiKA, selected science channels) highly accessible; after homework completion, allow unlimited daytime KiKA.
- Add non-screen replacements tied to son interests:
  - MINT mini-projects, sports, outdoor time, library with friends, or other social activities; rotate categories to keep variety high.
- Use explicit psychological support daily:
  - validate stress, recognize effort, and give one concrete confidence-building message tied to progress.
- Preserve dignity:
  - avoid public correction in shared chat; keep detail feedback private.
- Add adaptation + class-climate buffer:
  - when class stress is high, prioritize emotional regulation and a small success task first.
- Rebuild academic identity explicitly:
  - remind him weekly that prior high performance shows ability; current gap is language/time, not intelligence.
- Target participation as a trainable skill:
  - one low-risk contribution goal per day is better than demanding full activity immediately; progress path is raise hand -> short answer -> fuller answer.
- Keep long-term success metric explicit:
  - track weekly trend in motivation/involvement (study consistency, topic recall effort, class participation, social activity), not only correctness scores.
- Review weekly and tune rules:
  - if school engagement increases and conflict decreases, keep trajectory;
  - if conflict spikes, slow phase transitions by 1-2 weeks.

## Open Decisions
- Define screen-time data source integration details (device OS API, MDM, etc.).
- Keep YouTube channel differentiation out of scope for MVP (too complex right now); treat YouTube as one entertainment category.

## MVP Scope
1. Telegram bot in group chat mode (commands + tagged free conversation).
2. 15-minute usage polling and warnings.
3. Night partial lock via NextDNS (including KiKA block at night).
4. Unlock request flow with command and natural-language trigger support.
5. Balanced penalty/bonus calculation and final unlock decision.
6. Phase-based weekly limit rollout and weekly report.
