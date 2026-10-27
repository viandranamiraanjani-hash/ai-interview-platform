# AI Interview Platform — Fullstack Product Engineer Case Study

**Deliverable:** Product and Engineering Assessment
**Repository:** AI Interview Platform
**Branch:** `feat/fitgap-report-reliability`
**Status:** Initial investigation and proposed remediation
**Date:** 27 October 2026

---

## 1. Executive Summary

The AI Interview Platform supports interview and assessment workflows, candidate portfolios, and AI-assisted fit/gap reporting. Its architecture combines a React and TypeScript frontend with a Rails API, PostgreSQL, Redis, Sidekiq background jobs, and Gemini-powered narrative generation.

The initial code review identified a reliability risk in fit/gap report regeneration: the existing report is deleted before the replacement has completed successfully. The frontend also clears the displayed report immediately after regeneration is accepted. Together, these behaviors can reduce the availability of assessment information during a failure or slow background job.

The recommended direction is to preserve the last successful report until a replacement is committed, make asynchronous job states explicit, improve user-facing error recovery, and add regression tests for failure paths.

This document distinguishes code observations from proposed changes. Runtime behavior, automated test results, and deployment readiness must be verified before the work is represented as complete.

## 2. Product Context and User Journey

### Primary users

- **Recruiters and hiring managers:** review candidate evidence and compare it with role requirements.
- **Assessors:** contribute skill assessments and professional judgment.
- **Candidates:** provide information and participate in an evaluation process, even though they may not choose the assessment system themselves.

### End-to-end journey

1. A recruiter or hiring manager selects an assessment or vacancy.
2. The candidate participates in the interview and assessment process.
3. The platform generates or updates the candidate portfolio.
4. The system compares assessed skills with vacancy expectations.
5. The recruiter reviews matches, gaps, exceeding skills, and skills that have not been assessed.
6. The recruiter uses the report as supporting evidence alongside human judgment.
7. The report may be regenerated when relevant assessment inputs change.

### Product principles

- Assessment outputs must be traceable to their underlying evidence.
- Missing information must be represented as unknown or not assessed, not silently interpreted as a skill gap.
- AI-generated narrative must be treated as assistance, not an autonomous hiring decision.
- A transient service failure must not unnecessarily destroy previously available assessment information.
- Candidate information must be handled with appropriate access controls and data minimization.

## 3. Initial Repository and Architecture Review

The repository documentation identifies the following components:

| Component | Responsibility |
|---|---|
| React 18, TypeScript, Vite | User interface and client-side interactions |
| Rails API | Application endpoints and business logic |
| PostgreSQL | Persistent application and assessment data |
| Redis and Sidekiq | Background job processing |
| Gemini API | AI-assisted narrative generation |

The documented local setup requires Ruby, Node.js, PostgreSQL, Redis, application configuration, and a working Gemini API key for AI-backed functionality.

### Evidence reviewed

- `api/app/controllers/api/v1/portfolios_controller.rb`
- `api/app/services/fit_gap/engine.rb`
- `api/app/workers/fit_gap_generator_worker.rb`
- `web/src/pages/fitgap/FitGapReportPage.tsx`
- `api/README.md`
- `web/README.md`

The review so far is a static code inspection. A complete runtime assessment remains pending.

## 4. Problem and Gap Analysis

### Finding A — Existing report is deleted before replacement succeeds

**Location:** `api/app/controllers/api/v1/portfolios_controller.rb`, `regenerate_fitgap`

The controller deletes the existing `FitGapReport` and then enqueues `FitGapGeneratorWorker`. The endpoint responds with HTTP 202 after the enqueue call.

**Potential impact:** if the worker fails, the queue is unavailable, or regeneration does not finish, the previous report may no longer be available. A recruiter may lose access to the last successful comparison while waiting for a replacement.

**Initial severity:** P1 — high priority reliability risk. Final severity should be confirmed against business requirements and actual failure behavior.

**Category:** implementation reliability gap.

**Proposed acceptance criteria:**
- The last successful report remains available until a replacement is successfully persisted.
- A failed replacement does not destroy the previous successful result.
- Repeated regeneration requests do not create conflicting or ambiguous report states.
- The endpoint returns a clear accepted or error response consistent with whether the job was actually enqueued.
- Regression tests cover worker failure and retry behavior.

### Finding B — Frontend clears the current report during regeneration

**Location:** `web/src/pages/fitgap/FitGapReportPage.tsx`, `handleRegenerate`

After the regeneration request succeeds, the frontend calls `setReport(null)` and sets `generating` to `true`. The existing report is therefore removed from the component state while the replacement is pending.

The function uses `finally` to reset `regenerating`, but it does not explicitly catch the request error or present a user-facing error message.

**Potential impact:** users can lose access to the previous report during a long-running operation. If the request fails, the user may receive insufficient feedback about whether regeneration started.

**Initial severity:** P1/P2 — prioritize according to report criticality and the actual failure experience.

**Category:** implementation and interaction-state gap.

**Proposed acceptance criteria:**
- The last successful report stays visible during regeneration.
- A visible loading indicator communicates that a replacement is being prepared.
- Request failure displays a useful error and leaves the previous report available.
- Polling distinguishes pending, completed, and failed generation.
- Repeated clicks are prevented while a request is being submitted.
- The user can retry safely after a failure.

### Finding C — AI narrative failure and partial-result behavior

**Location:** `api/app/services/fit_gap/engine.rb`

The engine derives skill comparisons from vacancy requirements and portfolio skills. It also recognizes `not_assessed` skills. Gemini narrative generation is wrapped in a rescue path that logs an error and returns a fallback overall narrative with a `nil` culture narrative.

This is a useful existing fallback, but its behavior and presentation need verification.

**Potential impact:** the report may contain a partial narrative or omit culture analysis. If the UI does not clearly distinguish missing AI content from a completed analysis, users may overestimate the completeness of the result.

**Initial severity:** P2 — medium priority, subject to user impact.

**Category:** existing resilience mechanism requiring validation.

**Proposed acceptance criteria:**
- Rule-based skill comparisons remain available when Gemini is unavailable.
- Missing AI narrative is explicitly represented in the UI.
- Invalid or malformed model output is handled without exposing internal details.
- Logs contain sufficient operational context without secrets or unnecessary candidate information.
- Tests cover timeouts, malformed JSON, missing narrative fields, and provider errors.

### Finding D — Error and retry behavior needs broader validation

The worker configures Sidekiq retries and re-raises general errors. The controller enqueues work asynchronously, while the frontend polls for report availability.

This structure makes failure handling important across multiple layers. The initial review has not established whether every failure scenario is handled correctly.

**Initial severity:** P2 — validation required.

**Proposed acceptance criteria:**
- A failed job can be retried without corrupting an existing successful report.
- Repeated requests are handled safely.
- A missing portfolio or vacancy is handled explicitly.
- Frontend polling can stop or transition to a failure state when appropriate.
- Operational logs do not include API keys, tokens, or unnecessary candidate data.

## 5. Product and Engineering Constraints

The following constraints should guide the implementation:

1. **Asynchronous work:** a request being accepted is not equivalent to report generation succeeding.
2. **Data integrity:** a previous successful result should not be removed before a replacement is ready.
3. **AI dependency:** Gemini can time out, return invalid output, or be unavailable.
4. **Partial assessments:** `not_assessed` is a distinct state and should not be presented as a confirmed skill gap.
5. **Candidate privacy:** assessment data should only be accessible to authorized users.
6. **Human oversight:** generated narratives should not be presented as definitive hiring decisions.
7. **Environment:** local dependency installation has encountered a Puma native-extension build failure on Windows. Backend runtime verification is pending; this environment issue is not, by itself, proof of an application defect.

## 6. Remediation Options and Recommended Strategy

### Option A — Minimal UI-only change

Keep the existing report visible while regeneration is pending and add explicit request error handling.

**Advantages:** small frontend change and quick user-experience improvement.

**Trade-offs:** does not solve deletion of the previous report in the backend. A failed regeneration may still leave no report to display.

### Option B — Backend-safe replacement and improved UI

Preserve the current report until a replacement has been successfully generated and persisted. Update the frontend to retain the last successful result, communicate generation state, and handle failures.

**Advantages:** addresses both data availability and user experience.

**Trade-offs:** requires careful handling of concurrent jobs, report replacement, and job failure states.

### Option C — Full report-versioning workflow

Store immutable report versions and track which version is current, along with generation status and error metadata.

**Advantages:** stronger auditability and historical traceability.

**Trade-offs:** broader schema and UI changes, migration work, and additional operational complexity.

### Recommendation

Prefer Option B for the immediate reliability fix, with a path toward Option C if product requirements call for historical report comparison or formal audit trails.

The implementation should be small, reversible, and protected by regression tests. Schema changes should only be introduced if necessary and must include a safe migration and rollback strategy.

## 7. Cross-Layer Implementation Plan

### Backend

- Stop deleting the previous report before generation succeeds.
- Define explicit semantics for queued, successful, and failed regeneration.
- Ensure the report replacement is persisted safely.
- Consider duplicate and concurrent jobs.
- Preserve the existing fallback for rule-based comparisons when AI narrative generation fails.
- Validate endpoint inputs and authorization according to the application's existing access-control conventions.
- Avoid logging secrets, authentication tokens, or unnecessary candidate data.

### Frontend

- Keep the current report visible during regeneration.
- Show a clear pending state and explain that a newer version is being prepared.
- Catch request errors and provide a retry path.
- Distinguish unavailable reports from reports that are still generating.
- Present missing AI narrative and unassessed skills explicitly.
- Verify loading, empty, error, and long-text states.

### Testing

Tests should be added or updated for:

- Successful regeneration.
- Background worker failure and retry.
- Preservation of an existing report after failure.
- Duplicate or concurrent regeneration requests.
- Missing or invalid vacancy identifiers.
- Missing assessment values and `not_assessed` skills.
- Gemini timeout, malformed response, and partial narrative.
- Frontend loading, failure, retry, and report-preservation states.

These are proposed tests, not a claim that they have already been executed.

## 8. Privacy, Security, and Responsible AI

The platform handles information that can affect a candidate's employment opportunities. The implementation should therefore follow data minimization, access control, secure logging, and appropriate retention practices.

For an Indonesian deployment, review applicable requirements under Indonesia's Personal Data Protection Law (UU No. 27 Tahun 2022), including the organization's lawful processing basis, transparency, security safeguards, and data-subject rights. Legal or privacy review should confirm the specific obligations for the deployment.

AI-generated text should summarize evidence and uncertainty rather than infer unsupported personal characteristics. Recruiters should retain responsibility for hiring decisions, and the interface should not turn missing data into negative evidence.

## 9. Verification Plan and Current Limitations

### Verified through static inspection

- The regeneration controller deletes the existing report before enqueuing the worker.
- The engine uses `find_or_initialize_by` and updates the report after generating comparisons and narratives.
- The worker is configured for retries and re-raises general exceptions.
- The frontend clears its report state after a successful regeneration request.
- The engine contains a fallback path for narrative-generation errors.
- The local development documentation identifies Rails, PostgreSQL, Redis, Sidekiq, and the frontend as required components.

### Not yet verified

- Backend boot and API behavior in a running local environment.
- Database migrations and seed execution.
- Automated backend regression tests.
- Frontend production build and test results for the current branch.
- End-to-end behavior under worker failure.
- Actual Gemini timeout and malformed-response behavior.
- Screenshot evidence and a recorded demonstration.
- A reviewed pull request and final commit history.

### Local environment limitation

A native Puma dependency build failed during Windows setup. This currently blocks complete backend runtime verification. The failure should be documented as a setup limitation until resolved; it must not be reported as a successful backend test or used to claim application behavior that has not been observed.

## 10. Evidence and Submission Checklist

- [x] Implement the targeted regeneration reliability improvement.
- [x] Add frontend regeneration-request error handling and preserve the existing report in React state.
- [x] Run the frontend production build successfully after the changes.
- [x] Run `git diff --check` successfully.
- [ ] Add regression tests for successful and failed regeneration.
- [ ] Run backend tests and verify runtime behavior.
- [ ] Verify background-job failure, polling timeout, concurrent regeneration, and authorization behavior.
- [ ] Capture screenshots of the report, loading state, and error/retry state.
- [ ] Review the final changes and create the pull request.
- [ ] Record a 3–5 minute demonstration and provide an accessible video link.
- [ ] Confirm no secrets, tokens, or real candidate personal data appear in submission evidence.
- [ ] Export one polished PDF and upload it to the hiring platform.

## 11. Final Status

The repository has a targeted reliability improvement for fit/gap report regeneration. The backend controller no longer deliberately deletes the existing report before enqueueing regeneration, and the frontend preserves the displayed report when the regeneration request fails. The frontend also displays a dismissible error message when that request fails.

The frontend production build passes, and `git diff --check` reports no whitespace errors. Backend tests, end-to-end runtime behavior, screenshots, pull-request review, and the demonstration video have not yet been completed or verified. This report must not be treated as evidence of a fully tested release.


## 12. Implementation Progress and Verification Update

**Update date:** 27 October 2026

### Changes implemented

1. **Backend regeneration reliability:** Removed the controller statement that deleted the existing fit/gap report before the background regeneration job was enqueued. The previous report is no longer deliberately deleted at the start of regeneration.
2. **Frontend report preservation:** Removed the statement that cleared the existing report from React state after a regeneration request was accepted.
3. **Frontend error handling:** Added a dedicated regeneration-error state, request failure handling, and a dismissible error message. The regeneration action also checks whether regeneration is already in progress.

### Verification evidence

- `npm.cmd run build` — passed after the frontend changes. TypeScript compilation and the Vite production build completed successfully.
- Latest recorded build: 1,841 modules transformed; build completed in 2.60 seconds.
- `git diff --check` — passed with no reported whitespace errors.
- Backend automated tests and runtime behavior remain unverified because the local Puma dependency installation failed on Windows.
- The repository's `api/spec` directory did not return any files during inspection. This does not establish that no other test infrastructure exists elsewhere in the repository.

### Remaining limitations

- Add regression coverage for failed regeneration and preservation of an existing report.
- Verify how the frontend handles background-job failure and polling timeouts.
- Review concurrent regeneration requests and authorization behavior.
- Capture screenshots and record the required demonstration.
- Review the final changes, commit them, and create the pull request before submitting the final PDF.

**Current status:** A targeted reliability improvement has been implemented, and the frontend production build passes. End-to-end behavior and backend test coverage have not yet been verified.
