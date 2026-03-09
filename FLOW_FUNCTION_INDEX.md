# Zoho Flow Function Index

## Stage 1: Active Project Discovery
- File: `buildAgentStudioPortfolioView`
- Function: `buildAgentStudioPortfolioSnapshotPaged(start_index_text, max_projects_text, include_activity_text, max_events_per_project_text, project_status_filter_text, project_type_filter_text)`
- Purpose: return only active, in-scope Agent Studio projects.
- Output keys: `window`, `limits`, `source_stats`, `active_projects`.

## Stage 2: Notes-Only Scan
- File: `scanProjectNotesFromActiveProjects`
- Function: `scanProjectNotesFromActiveProjects(base_data, notes_today_only_text, max_notes_per_project_text)`
- Purpose: scan task notes for projects in `base_data.active_projects`.
- Output keys: `notes`, `stats`, `api_errors`.

## Stage 3: Risk Detail Scan
- File: `buildProjectRiskFromActiveProjects`
- Function: `buildProjectRiskFromActiveProjects(base_data)`
- Purpose: compute risk scores and indicators from tasks/bugs for `active_projects`.
- Output keys: `risk_rows`, `stats`, `api_errors`.

## Stage 4: Low-Activity + Overdue Risk
- File: `identifyLowActivityRiskFromActiveProjects`
- Function: `identifyLowActivityRiskFromActiveProjects(base_data, inactivity_days_text, overdue_threshold_text)`
- Purpose: identify projects with little activity and/or overdue task pressure.
- Output keys: `flagged_projects`, `stats`, `api_errors`.

## Recommended Wiring
1. Initialize `start_index = "1"`.
2. Call stage 1 and collect `active_projects` across pages using `window.has_more` and `window.next_index`.
3. Build one `base_data` map with merged `active_projects`.
4. Call stage 2 with `base_data` for note extraction.
5. Call stage 3 with `base_data` for risk details.
6. Call stage 4 with `base_data` for low-activity and overdue risk flags.
7. Feed outputs to LLM and Slack in later Flow steps.

## Suggested Defaults
- Stage 1: `max_projects_text = "25"`
- Stage 2: `notes_today_only_text = "true"`, `max_notes_per_project_text = "10"`
- Stage 4: `inactivity_days_text = "7"`, `overdue_threshold_text = "1"`
