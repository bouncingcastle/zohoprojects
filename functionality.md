# System Purpose And Functional Specification

## System Purpose
The system provides a daily executive-level view of delivery health for active Agent Studio implementations in Zoho Projects.

It is designed to:
- Continuously monitor implementation activity and risk signals.
- Convert operational data into decision-ready insights.
- Feed a downstream AI interpretation layer (Zia/LLM) with structured context.
- Publish concise leadership summaries to Slack.

## Deploy Mode
For Zoho Flow deployment, this repository now uses a single copy/paste deploy artifact:
- `buildAgentStudioPortfolioSnapshotPaged_DEPLOY`

The deploy artifact is self-contained to avoid cross-function dependency issues in Zoho Flow custom functions.

## Business Outcome
By 1 run per day, stakeholders should be able to answer:
- Which Agent Studio projects are healthy, at risk, or stalled?
- What changed in the last 24 hours?
- What actions are needed today?

## High-Level Functionality
Each day the system:
1. Discovers active projects.
2. Filters projects by implementation type.
3. Extracts meaningful activity from the last 24 hours.
4. Normalizes activity into structured events.
5. Detects project signals (progress, risks, stagnation).
6. Formats a readable activity report.
7. Uses AI (Zia) to interpret activity and generate executive narrative.
8. Posts an executive summary to Slack.

## Purpose-Fit Analysis Against Current Script
File: `listActiveProjects`

Current functions:
- `listActiveProjects()`
- `listAgentStudioProjects()`
- `getProjectRiskIndicators(project_id, project_updated_date, project_owner)`
- `normalizeProjectRecordEvent(project_id, project_name, event_time, event_type, entity_type, entity_id, severity, actor, summary, metadata)`
- `extractProjectActivity24h(project_id, project_name)`
- `buildProjectRiskActivitySnapshot(project_id, project_name, project_updated_date, project_owner)`
- `buildPortfolioSummaryFromSnapshots(snapshots)`
- `buildAgentStudioPortfolioSnapshot()`
- `compactEventForFlow(event_rec)`
- `shrinkSnapshotForFlow(snapshot, include_activity_text, max_events_per_project_text)`
- `buildAgentStudioPortfolioSnapshotLite(max_projects_text, include_activity_text, max_events_per_project_text)`
- `buildAgentStudioPortfolioSnapshotPaged(start_index_text, max_projects_text, include_activity_text, max_events_per_project_text)`

Coverage status:
- Implemented: Discover active projects.
- Implemented: Filter by implementation type (`Agent Studio`) and exclude onboarding.
- Implemented (first pass): Extract 24-hour activity from tasks and bugs via timestamp-long fields.
- Implemented (first pass): Normalize activity into canonical event records.
- Partially implemented: Detect project risk signals via tasks, bugs, stale updates, and owner checks.
- Implemented: Return structured per-project payload for downstream Flow steps and LLM summarization.
- Not implemented in current script: In-script readable narrative formatting.
- Intentionally not implemented in current script: Slack posting (removed by design).
- Not implemented in current script: Zia/LLM summarization call.

Conclusion:
- The script is purpose-fit as a data acquisition and risk signal engine.
- It is now functionally ready for a manual Zoho Flow pipeline.
- The remaining gap is AI narrative generation + Slack delivery in downstream Flow steps.

## Zoho Flow Wiring Model
This codebase is designed as function modules, not a single orchestrator.

Recommended manual Flow wiring:
1. Call `listAgentStudioProjects()`.
2. Loop each returned project.
3. Call `buildProjectRiskActivitySnapshot(project_id, project_name, updated_date, owner)` for each item.
4. Aggregate snapshots into one report object in Flow.
5. Send aggregate report to your LLM/Zia step.
6. Post final executive summary to Slack in a separate Flow action.

Low-loop alternative for Zoho Flow limitations:
1. Call `buildAgentStudioPortfolioSnapshotLite(...)` once.
2. Use returned `portfolio` + `project_snapshots` directly as LLM input.
3. Post LLM result to Slack in a separate Flow action.

Scale-safe alternative for ~100 active projects:
1. Start with `start_index_text = "1"`.
2. Call `buildAgentStudioPortfolioSnapshotPaged(start_index_text,"10","true","5")`.
3. Accumulate `project_snapshots` across page calls in Flow.
4. Repeat while `window.has_more == true` using `window.next_index`.
5. Send accumulated snapshots to LLM/Zia and then to Slack.

Recommended Flow-safe parameters:
- `max_projects_text`: `"10"` to `"25"`
- `include_activity_text`: `"true"` or `"false"`
- `max_events_per_project_text`: `"5"` to `"10"`

## Functional Architecture

### Stage 1: Project Discovery
Input:
- Zoho Projects portal scope.

Processing:
- Paginate projects.
- Exclude inactive lifecycle states.

Output:
- Candidate active project list.

### Stage 2: Project Type Filtering
Input:
- Candidate projects + custom fields.

Processing:
- Keep `Implementation Type` containing `Agent Studio`.
- Exclude onboarding implementation subtypes.

Output:
- Active Agent Studio project set.

### Stage 3: 24-Hour Activity Extraction
Input:
- Active Agent Studio projects.

Processing:
- Query tasks, bugs, milestones, comments, status changes, ownership changes, and due-date changes.
- Restrict to `event_time >= now - 24h`.

Output:
- Raw activity records per project.

### Stage 4: Event Normalization
Input:
- Raw activity records from heterogeneous endpoints.

Processing:
- Convert records into a single event schema.
- Enforce consistent timestamps, actors, entity references, and semantic event categories.

Canonical event schema:
- `project_id`
- `project_name`
- `event_time`
- `event_type` (task_completed, task_overdue, bug_opened, bug_escalated, comment_added, milestone_changed, owner_changed)
- `entity_type` (task, bug, milestone, project)
- `entity_id`
- `severity` (low, medium, high)
- `actor`
- `summary`
- `metadata` (map)

Output:
- `normalized_events[]`

### Stage 5: Signal Detection
Input:
- `normalized_events[]` + snapshot indicators.

Processing:
- Detect positive progress signals.
- Detect risk signals.
- Detect stagnation/no-activity patterns.
- Score project health.

Signal examples:
- Progress: tasks closed, bugs resolved, milestone moved forward.
- Risk: overdue growth, high-severity bug creation, owner unset, deadline compression.
- Stagnation: no meaningful updates over threshold.

Output:
- Per-project signal map + risk level + confidence notes.

### Stage 6: Human-Readable Report Assembly
Input:
- Per-project signals and scores.

Processing:
- Build a compact report object for AI and non-AI consumers.
- Group by risk level.
- Add top drivers and recommended actions.

Output:
- `report_payload` suited for Zia/LLM prompt input.

### Stage 7: AI Interpretation (Zia)
Input:
- `report_payload`.

Processing:
- Generate executive narrative:
  - what improved
  - what deteriorated
  - where intervention is needed
  - immediate priorities

Output:
- `executive_summary`
- `key_actions[]`
- `watchlist[]`

### Stage 8: Slack Delivery
Input:
- AI output + core metrics.

Processing:
- Compose channel message with concise sections.

Output:
- Daily Slack executive post.

## Current Output Contract (Script)
From `buildProjectRiskActivitySnapshot(...)` per project:
- `project_id`
- `project_name`
- `owner`
- `risk_score`
- `risk_level`
- `flags`
- `indicators`
- `risk_api_errors`
- `activity_24h`
- `activity_stats_24h`
- `activity_api_errors`

This is suitable as upstream input for:
- LLM summarization stage.
- Separate Slack posting stage.

From `buildAgentStudioPortfolioSnapshot()` (single-call mode):
- `source` (Agent Studio project source payload)
- `project_snapshots` (full per-project snapshots)
- `portfolio`:
  - `stats` (`scanned`, `high`, `medium`, `low`, `total_events_24h`, `api_error_count`, `generated_at`)
  - `rows` (ordered high -> medium -> low)
  - `top_risks`
  - `portfolio_flags`
  - `api_errors`

From `buildAgentStudioPortfolioSnapshotLite(...)` (payload-safe mode):
- `source`
- `project_snapshots` (trimmed snapshots)
- `portfolio` (summary over trimmed snapshots)
- `limits` (effective caps and processed counts)

From `buildAgentStudioPortfolioSnapshotPaged(...)` (cursor mode):
- `window`:
  - `start_index`
  - `end_index`
  - `processed`
  - `requested`
  - `total_projects`
  - `has_more`
  - `next_index`
- `limits`
- `project_snapshots` (trimmed snapshots for this page only)
- `portfolio` (summary for this page)
- `source_stats`

## Recommended Next Expansion (Without Reintroducing Slack In Script)
1. Extend `extractProjectActivity24h(project_id)` to include milestones, comments, and owner changes.
2. Improve timestamp fallback handling when long timestamp fields are absent.
3. Add `buildProjectSignals(normalized_events, current_indicators)` returning progress/risk/stagnation signals.
4. Add `buildLlmSummaryInput(report)` returning compact prompt-ready context object.

## Non-Functional Requirements
- Daily schedule reliability with partial-failure tolerance.
- Endpoint-level error capture without aborting full run.
- Deterministic output schema for downstream LLM consumption.
- Pagination safety for larger portfolios.
- Auditable debug traces for each run.
- Payload-size safety for Zoho Flow action limits.
- Deluge compatibility: avoid advanced or unsupported language constructs.
