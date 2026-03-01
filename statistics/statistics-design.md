# Statistics Design: App Usage Collection (Android + Windows)

## Objective
Collect per-device app usage statistics to support parental insights such as:
- total time spent per app per day
- app open count and session duration
- active usage windows (hourly/day-of-week patterns)

## Scope
- Platforms: Android and Windows
- Metrics: foreground app usage only (initial version)
- Aggregation: local collection + periodic sync to backend
- Time granularity: session-level raw events, daily aggregates for UI

## Core Data Model
Use one normalized event schema for both platforms.

```json
{
  "device_id": "string",
  "platform": "android|windows",
  "app_id": "string",
  "app_name": "string",
  "event_type": "session_start|session_end|heartbeat",
  "ts_utc": "2026-02-28T10:15:00Z",
  "session_id": "string",
  "meta": {
    "version": "string",
    "source": "os_api"
  }
}
```

Derived daily aggregate example:

```json
{
  "device_id": "string",
  "platform": "android",
  "date": "2026-02-28",
  "app_id": "com.example.app",
  "total_foreground_seconds": 5400,
  "launch_count": 12,
  "first_seen_utc": "2026-02-28T06:21:00Z",
  "last_seen_utc": "2026-02-28T22:08:00Z"
}
```

## Android Collection

### Preferred API
- `UsageStatsManager` + `UsageEvents`
- Requires user-granted Usage Access (`PACKAGE_USAGE_STATS`) in system settings.

### Collection Flow
1. Verify Usage Access permission.
2. Poll `UsageEvents` using a moving watermark (`last_collected_ts`).
3. Parse foreground transitions (`MOVE_TO_FOREGROUND`, `MOVE_TO_BACKGROUND`).
4. Build sessions per package.
5. Persist raw events locally (SQLite) and compute daily aggregates.

### Notes
- Some OEMs aggressively restrict background work; use `WorkManager` with retry/backoff.
- Reboots/time changes require watermark correction.
- App labels should be cached and periodically refreshed via `PackageManager`.

## Windows Collection

### Preferred Approach
Use active window/process tracking from a background service/agent:
- `GetForegroundWindow`
- `GetWindowThreadProcessId`
- process metadata via `QueryFullProcessImageName` / process APIs

### Collection Flow
1. Poll active foreground window/process every N seconds (e.g., 2-5s).
2. Detect app switch boundaries and synthesize session start/end events.
3. Resolve executable path/process name to stable `app_id`.
4. Store events locally and derive daily aggregates.

### Notes
- Polling interval trades precision vs battery/CPU.
- Elevated/system processes may limit metadata visibility.
- Multi-window/browser scenarios may require extra normalization rules.

## Option B: Managed Provider (RescueTime)

Use RescueTime as an external source for cross-device usage instead of building native collectors first.

### When to Choose This Option
- Need faster time-to-market for Android + Windows coverage.
- Accept dependency on a third-party service and account model.
- Aggregated activity data is sufficient for initial analytics.

### High-Level Flow
1. Child/managed user installs RescueTime on Android and Windows, grants required permissions.
2. Parent connects a RescueTime account in our app (API key or OAuth if available for our use case).
3. Our backend runs scheduled imports from RescueTime API.
4. Imported records are normalized into our `daily_app_usage` model.
5. Dashboard reads from unified internal tables (same as native collectors).

### API Integration Details
- Primary endpoint: `https://www.rescuetime.com/anapi/data`
- Typical query parameters:
  - `key=<api_key>`
  - `format=json`
  - `perspective=interval`
  - `restrict_begin=YYYY-MM-DD`
  - `restrict_end=YYYY-MM-DD`
  - `resolution_time=hour` (or `day`)
  - `restrict_source_type=computers,mobile` (if supported by account/data)
- Import schedule:
  - Backfill: last 30-90 days on first connect.
  - Incremental: hourly or every 6 hours.
  - Reconciliation: re-import last 2 days to catch delayed sync.

### Mapping to Internal Schema
- `platform`:
  - `computers` -> `windows` (or `desktop` if we add macOS later)
  - `mobile` -> `android` (best effort based on account/device mix)
- `app_id`:
  - Prefer stable activity identifier from RescueTime record.
  - Fallback to normalized app/site name.
- `total_foreground_seconds`:
  - Map from RescueTime duration field for each interval row.
- `launch_count`:
  - Not always directly available; derive only if source provides event-like rows.

### Known Constraints
- Data is typically interval/aggregate based, not guaranteed raw foreground session events.
- Exact app/package mapping may be weaker than native OS-level collectors.
- Device attribution detail depends on what RescueTime exposes for the account/plan.
- API fields/limits may change; integration should validate schema on each sync run.

### Security and Compliance
- Store API credentials encrypted at rest.
- Use least-privilege token scope when possible.
- Provide explicit consent text that usage data is imported from a third-party provider.
- Support account disconnect and data deletion workflows.

### Recommended Architecture
- Keep a provider abstraction:
  - `provider_type = native_android | native_windows | rescuetime`
  - Normalization pipeline writes into shared internal tables.
- This allows starting with RescueTime and later migrating to native collection without dashboard rewrite.

## Alternatives to RescueTime (Current Shortlist)

These options can be evaluated if RescueTime is not selected or as secondary providers.

### Comparison Table (Freshness, Plan, Platforms)

| Option | Sync/Freshness vs RescueTime Lite (30 min) | Plan (Free or Cost) | Supported platforms | Important note |
| --- | --- | --- | --- | --- |
| RescueTime | Lite: ~30 min; Premium: ~5 min | Lite: Free. Paid: Solo from $7/mo annual ($9 monthly), Solo+ from $12/mo annual ($15 monthly) | Windows, macOS, Android, iOS, web | Best fit for Android + Windows + API |
| ManicTime | No fixed sync interval publicly documented | 30-day trial, then from $7/user/month | Windows, macOS, Linux, Android, iOS | Android phone-usage tracking needs sideload plugin |
| DeskTime | No fixed sync interval publicly documented | No permanent free tier (14-day trial); Pro $6.42/user/mo annual, Premium $9.17/user/mo annual, Enterprise custom | Windows, macOS, Linux; Android/iOS mobile app | Mobile is mostly manual/timer-oriented |
| Hubstaff | Mobile uploads about every ~10 min (time/location); app/url tracking is desktop-only | Free plan (1 seat) + paid Starter/Grow/Team/Enterprise | Windows, macOS, Linux, ChromeOS desktop; Android/iOS mobile | Mobile app does not capture apps/URLs |
| Toggl Track | Auto cloud sync; no fixed minute cadence published | Free (up to 5 users); Starter $9/user/mo; Premium $18/user/mo; Enterprise custom | Web, Windows, macOS, Android, iOS, browser extension | More time-tracking than screentime telemetry |
| Clockify | Auto-tracker data is local-only (not auto-uploaded) | Free forever + paid tiers (Basic $3.99, Standard $5.49, Pro $7.99 annual rates) | Windows, macOS, Linux, Android, iOS, web/extensions | Requires local-to-server conversion workflow |

### 1) ManicTime
- Strengths:
  - Public API (`Timeline API`) with cloud/on-prem deployment options.
  - Strong desktop tracking, including Windows.
- Caveats:
  - Android phone-usage data requires extra plugin/sideload flow.
  - More setup/operations overhead than RescueTime.

### 2) DeskTime
- Strengths:
  - API access for employee apps/projects data.
  - Mature desktop time tracking on Windows.
- Caveats:
  - Android experience is primarily mobile timer-based, not equivalent to deep automatic app-usage telemetry.

### 3) Hubstaff
- Strengths:
  - Public API and broad platform support.
  - Good for workforce time tracking use cases.
- Caveats:
  - App/URL tracking is desktop-focused; mobile does not provide equivalent app/URL capture depth.

### 4) Toggl Track / Clockify
- Strengths:
  - Public APIs and broad integrations.
  - Good for manual/semi-automatic time entry and reporting.
- Caveats:
  - Not designed as unified cross-device screentime telemetry providers.
  - Automatic activity timelines are less suitable for parental-style app usage analytics.

### Working Recommendation
- Primary managed-provider candidate: `RescueTime`
- Secondary candidate: `ManicTime` (if team accepts server/plugin complexity)
- Keep provider abstraction so alternatives can be tested without dashboard refactor.

## Local Storage and Sync
- Local DB tables:
  - `usage_events`
  - `daily_app_usage`
  - `sync_state`
- Sync strategy:
  - incremental upload by event id / timestamp watermark
  - idempotent ingestion on backend (dedupe by event_id/session_id)
  - exponential backoff for failures

## Privacy and Compliance
- Do not collect content, keystrokes, or URLs in initial version.
- Minimize metadata to app identity + timing.
- Provide clear guardian/child disclosure and consent flow.
- Encrypt data at rest and in transit.

## Reliability Considerations
- Clock drift and timezone changes: store UTC, render in local TZ.
- Offline periods: keep local queue and resend.
- Duplicate events: enforce unique keys and merge logic.
- Long-running sessions across midnight: split for daily aggregation.

## Open Questions
1. What minimum sampling interval on Windows is acceptable for accuracy vs resource use?
2. Do we need near-real-time dashboard updates or daily batch is sufficient?
3. How long should raw events be retained before compaction?
4. Should we track browser-level stats now or defer to later phase?
5. For RescueTime option, what data fidelity is acceptable (interval aggregates vs raw sessions)?
6. Should RescueTime be primary ingestion or only fallback/bootstrap source?

## Proposed MVP Plan
1. Implement Android collection with Usage Access + daily aggregates.
2. Implement Windows foreground process polling + session synthesis.
3. Unify ingestion API and storage schema across platforms.
4. Add dashboard endpoints for top apps, daily totals, and trend lines.
5. Validate with 7-day pilot and measure collection gaps.

## Alternative MVP Plan (RescueTime-First)
1. Implement RescueTime account connect flow and secure credential storage.
2. Build scheduled importer (backfill + incremental + reconciliation).
3. Normalize into `daily_app_usage` and expose the same dashboard endpoints.
4. Add provider-quality flags in UI (e.g., "aggregate data source").
5. Run side-by-side validation against native collectors for 1-2 weeks before deciding long-term source.
