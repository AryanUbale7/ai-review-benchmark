# AI Review Benchmark — Experiment Report

## Contents

- [1. Experiment Overview](#1-experiment-overview)
- [2. Experimental Methodology](#2-experimental-methodology)
- [3. Finding Evaluation Method](#3-finding-evaluation-method)
- [4. Case-by-Case Results](#4-case-by-case-results)
- [5. Cross-Case Comparison](#5-cross-case-comparison)
- [6. Attempt-Level Metadata](#6-attempt-level-metadata)
- [7. Limitations](#7-limitations)
- [8. Remaining Work](#8-remaining-work)
- [9. Final Status](#9-final-status)

## Executive Summary

This report records an eight-case comparison of Proofline, CodeRabbit and Qodo on the AI Review Benchmarks application. The reviewers examined the same case changes in a shared private GitHub review repository. The recorded context was code-only.

The clearest shared findings appear in Cases 01, 02, 03 and 06. Cases 04, 05, 07 and 08 had no actionable findings recorded from any of the three reviewers. Qodo reported two additional Case 01 concerns that still need independent verification. CodeRabbit's two Case 06 comments also need to be checked to determine whether they represent separate defects or overlapping descriptions.

The results are useful as a record of these specific review attempts, but they are not yet a complete or controlled provider ranking. Attempt-level metadata, provider-isolation evidence, manual validation and the separate CI/AWS deployment exercise remain outstanding.

---

## 1. Experiment Overview

### 1.1 Objective

The objective of this experiment is to compare AI-powered code reviewers by running them against the same code changes and evaluating the quality of their review findings.

The experiment uses the **AI Review Benchmarks** application. The reviewers recorded in the experiment are:

- Proofline
- CodeRabbit
- Qodo

All reviews are initiated through GitHub pull requests, except where a Qodo review was explicitly triggered with the `/agentic_review` command.

### 1.2 Experiment Environment

| Attribute | Value |
|---|---|
| Benchmark | AI Review Benchmarks |
| Review repository | `AryanUbale7/ai-review-run-01` |
| Repository visibility | Private |
| Base branch | `main` |
| Case branches | `pilot/01` to `pilot/08` |
| Context condition | Code-only |
| Primary review trigger | GitHub Pull Request |
| Reviewers | Proofline, CodeRabbit, Qodo |

### 1.3 Scope and Limitations

The experiment covers eight related changes within one application. All cases use the documented benchmark case branches and the same review repository.

The results provide evidence about these particular changes. They should not be treated as a general ranking of AI code reviewers across unrelated repositories or software systems.

---

## 2. Experimental Methodology

### 2.1 Unit of Observation

One review attempt is treated as an individual observation.

Each observation should identify:

- Provider
- Case number
- Base commit
- Head commit
- Context condition
- Repository
- Pull request
- Trigger
- Run identifier
- Timestamp
- Provider plan or model, when visible
- Review settings, when visible
- Terminal outcome
- Original review comments
- Whether the attempt was a retry

A repeated delivery of the same GitHub webhook should not be counted as a new intentional review attempt. An intentional second review should be recorded separately.

### 2.2 Input Isolation

Each benchmark case is reviewed through its corresponding application-only branch. The benchmark implementation, experiment notes and grading information are kept outside the reviewer-facing application changes.

For the recorded run, Proofline, CodeRabbit and Qodo reviewed the same eight pull requests in the shared private repository `AryanUbale7/ai-review-run-01`. This keeps the code changes consistent across reviewers, but it does **not** provide separate provider-specific review repositories. As a result, repository indexing, prior review comments or shared learning could have affected later reviews.

Provider indexing, shared-learning settings and access to earlier review discussions have not yet been independently verified. This is an isolation limitation and must remain visible in the final interpretation.

### 2.3 Deployment Context

The current comparison uses code-only reviews. No deployment snapshot is included in the recorded comparisons.

This means the experiment evaluates findings that can be identified from the code changes and the information available in the pull request.

The AWS deployment exercise and the synthetic two-site records for Cases 04–06 are separate benchmark considerations. They should not be treated as evidence supplied to these code-only reviews.

---

## 3. Finding Evaluation Method

Each reported finding should be assessed against the actual code change and its potential impact.

| Category | Meaning |
|---|---|
| Valid finding | A reproducible correctness, reliability, security or performance issue |
| False positive | A claim that is unsupported or does not represent an actual defect |
| Duplicate | A finding that describes an issue already reported by the same reviewer |
| Missed defect | A known defect that the reviewer did not identify |
| Unsupported warning | A warning that cannot be substantiated by the available evidence |
| No actionable finding | The reviewer completed the review without reporting an actionable issue |
| Incomplete attempt | The reviewer did not successfully finish the review |

A completed review with no actionable findings is not automatically a false negative. The underlying change must be assessed before making that conclusion.

Original review comments should be retained alongside the assessment.

---

# 4. Case-by-Case Results

## Case 01 — Page Report History by Scheduled Time

### Review Configuration

| Attribute | Value |
|---|---|
| Case | 01 |
| Repository | `AryanUbale7/ai-review-run-01` |
| Pull Request | #1 |
| Base branch | `main` |
| Head branch | `pilot/01` |
| Context | Code-only |

### Recorded Review Attempts

| Provider | Trigger | Outcome | Findings | Valid | False positives |
|---|---|---|---:|---:|---:|
| Proofline | GitHub PR | Completed | 1 | 1 | 0 |
| CodeRabbit | GitHub PR | Completed | 1 | 1 | 0 |
| Qodo | Manual `/agentic_review` | Completed | 3 | 1 confirmed; 2 require verification | 0 confirmed |

### Finding Assessment

Both Proofline and CodeRabbit identified the pagination cursor issue.

The history query orders records by:

```sql
scheduled_at DESC, id DESC
```

However, the cursor uses only `scheduled_at`.

If multiple deliveries share the same `scheduled_at` value at a page boundary, the next page can skip records. A correct keyset cursor should include both `scheduled_at` and `id` to maintain consistent pagination.

### Proofline and CodeRabbit — Shared Finding

- **Issue:** History pages can silently skip entries that share a timestamp.
- **Assessment:** Valid.
- **Reason:** The query sorts by both `scheduled_at` and `id`, but the cursor filters only by `scheduled_at`. Rows sharing the boundary timestamp can be omitted from the next page.

### Qodo Review

Qodo reported three findings:

| # | Finding | Priority | Category | Assessment |
|---:|---|---|---|---|
| 1 | History pages silently skip entries that share a timestamp | High | Correctness | Valid |
| 2 | Pre-upgrade history cursors return wrong pages | Medium | Correctness | Likely valid; verify backward compatibility |
| 3 | History query has no index for its new sort | Low | Performance | Requires migration/index verification |

**Assessment notes**

1. **Timestamp tie pagination:** The cursor filters only by `scheduled_at`, while results are sorted by `scheduled_at DESC, id DESC`. Entries sharing the boundary timestamp can be skipped.
2. **Old cursor compatibility:** The cursor meaning changes from a row ID (`id < ?`) to a `scheduled_at` timestamp (`scheduled_at < ?`), while the CLI still accepts an integer cursor without versioning. A cursor issued by the old implementation may therefore be compared against Unix-second timestamps and return an empty or incorrect page. This is a plausible compatibility issue; verify the intended backward-compatibility contract before final grading.
3. **Index support:** The query filters and sorts using `scheduled_at`, but Qodo reports that the migration does not add a supporting index for the new sort. Verify the actual migration and SQLite query plan before grading this as a confirmed performance issue.

### Commit References

- Base commit: `9b5ac64ab0d3ba414c5edec1e64fcdf8b3b19a1a`
- Head commit: `b42257746fa78caad922340082a2a27ed00cdeb8`
- Pull Request: #1
- Repository: `AryanUbale7/ai-review-run-01`

### Case 01 Conclusion

Proofline and CodeRabbit identified the same valid pagination issue. Qodo reported the same core issue and two additional concerns that require further verification.

This result represents one case and should not be generalized to overall reviewer performance.

---

### Case 02 — Stream report export

| Reviewer | Findings | Assessment |
|---|---:|---|
| Proofline | 1 | Valid |
| CodeRabbit | 1 | Valid |
| Qodo | 1 | Valid |

**Shared finding:** All three reviewers identified the CSV export issue: report values are joined manually using commas without proper CSV quoting or escaping.

**Qodo finding:** `write_export` uses `",".join(map(str, row))`, which can corrupt exported CSV data when a report value contains commas, double quotes, or newline characters.

**Evaluation status:** One valid finding recorded for each reviewer. Using Python's `csv.writer` would provide proper CSV escaping while preserving the streaming approach.

---

### Case 03 — Batch queued deliveries

| Reviewer | Findings | Assessment |
|---|---:|---|
| Proofline | 1 | Valid |
| CodeRabbit | 1 | Valid |
| Qodo | 1 | Valid |

**Shared finding:** The batching loop processes only complete batches of 20 jobs. Any remaining jobs in a partial batch are skipped.

**Qodo finding:** `deliver_due` iterates only over complete groups of 20. Queues containing fewer than 20 jobs are never processed, and the remaining jobs in larger queues can remain pending until enough new jobs arrive.

**Evaluation status:** One valid finding recorded for each reviewer. The batching logic should process the final partial batch as well.
---

## Case 04 — Retire the Legacy Delivery Column

### Review Configuration

| Attribute | Value |
|---|---|
| Case | 04 |
| Repository | `AryanUbale7/ai-review-run-01` |
| Pull Request | #4 |
| Base branch | `main` |
| Head branch | `pilot/04` |
| Context | Code-only |

### Recorded Review Attempts

| Provider | Trigger | Outcome | Findings | Valid | False positives |
|---|---|---|---:|---:|---:|
| Proofline | GitHub PR | Completed | 0 | 0 | 0 |
| CodeRabbit | GitHub PR | Completed | 0 | 0 | 0 |
| Qodo | Manual `/agentic_review` | Completed | 0 | 0 | 0 |

### Review Observations

- Proofline performed a Deep Review because the change involved a database migration.
- CodeRabbit completed its review without generating actionable comments.
- Qodo reported no material issues.
- Both recorded GitHub checks passed.

### Case 04 Conclusion

None of the three reviewers reported an actionable finding. This is recorded as a no-finding result, not as proof that the migration is defect-free.

---

## Case 05 — Increase Export Batch Size

### Review Configuration

| Attribute | Value |
|---|---|
| Case | 05 |
| Repository | `AryanUbale7/ai-review-run-01` |
| Pull Request | #5 |
| Base branch | `main` |
| Head branch | `pilot/05` |
| Context | Code-only |

### Change Description

Export batch size increased from 25 to 50 rows.

### Recorded Review Attempts

| Provider | Trigger | Outcome | Findings | Valid | False positives |
|---|---|---|---:|---:|---:|
| Proofline | GitHub PR | Completed | 0 | 0 | 0 |
| CodeRabbit | GitHub PR | Completed | 0 | 0 | 0 |
| Qodo | Manual `/agentic_review` | Completed | 0 | 0 | 0 |

### Review Observations

**Proofline**
- Review completed successfully.
- No new findings.

**CodeRabbit**
- Review completed successfully.
- No actionable comments generated.

**Qodo**
- Review completed successfully.
- Bugs: 0.
- Rule violations: 0.
- Requirement gaps: 0.
- Actionable findings: 0.

**CI status**
- Both recorded GitHub checks passed.

### Manual Assessment

The change increases the number of rows processed per export batch from 25 to 50.

Neither Proofline nor CodeRabbit reported an actionable issue. Qodo also reported no material issues requiring review.

Further correctness and performance validation should be performed separately before treating the change as fully verified.

### Case 05 Conclusion

All three reviewers completed without reporting actionable findings. Manual performance validation remains separate.
---

## Case 06 — Use the New Delivery Queue

### Review Configuration

| Attribute | Value |
|---|---|
| Case | 06 |
| Repository | `AryanUbale7/ai-review-run-01` |
| Pull Request | #6 |
| Base branch | `main` |
| Head branch | `pilot/06` |
| Context | Code-only |

### Change Description

Reports are routed through the `report-dispatch` delivery queue instead of the previous `reports` queue.

### Recorded Review Attempts

| Provider | Trigger | Outcome | Findings/comments recorded |
|---|---|---|---:|
| Proofline | GitHub PR | Completed | 1 |
| CodeRabbit | GitHub PR | Completed | 2 |
| Qodo | Manual `/agentic_review` | Completed | 2 |

### Proofline Finding

- **Issue:** New reports remain pending after worker runs.
- **Assessment:** Valid.
- **Reason:** The scheduling code writes jobs to `report-dispatch`, while the documented worker invocation still uses `--queue reports`. The worker selects only jobs matching its configured queue, so newly scheduled reports can remain pending.

### CodeRabbit Findings

CodeRabbit reported two actionable comments concerning:

1. Preserving pending jobs on the old queue during migration.
2. Coordinating the new queue with existing workers.

These comments concern related migration and data-integrity risks. They should be checked against the original comments before being counted as two independent defects.

### Qodo Findings

Qodo reported two findings:

| # | Finding | Severity | Assessment |
|---:|---|---|---|
| 1 | New reports remain pending after worker runs | High | Valid |
| 2 | Existing reports remain on the old queue | Medium | Valid |

**Assessment notes**

1. **New reports remain pending:** The scheduler writes to `report-dispatch`, while the documented worker command still processes `reports`.
2. **Existing reports remain on the old queue:** Previously queued deliveries are not migrated when the queue name changes. Once the worker switches to `report-dispatch`, its exact-match query cannot deliver those existing rows unless the old queue is also drained or the rows are migrated.

### Case 06 Conclusion

All three reviewers identified risks associated with the queue migration. The findings highlight the importance of coordinating producer configuration, worker configuration and existing pending jobs during a queue transition.

---

## Case 07 — Index Pending Deliveries

### Review Configuration

| Attribute | Value |
|---|---|
| Case | 07 |
| Repository | `AryanUbale7/ai-review-run-01` |
| Pull Request | #7 |
| Base branch | `main` |
| Head branch | `pilot/07` |
| Context | Code-only |

### Recorded Review Attempts

| Provider | Trigger | Outcome | Findings | Valid | False positives |
|---|---|---|---:|---:|---:|
| Proofline | GitHub PR | Completed (Deep Review) | 0 | 0 | 0 |
| CodeRabbit | GitHub PR | Completed | 0 | 0 | 0 |
| Qodo | Manual `/agentic_review` | Completed | 0 | 0 | 0 |

### Review Assessment

All three reviewers completed their reviews without reporting any actionable findings.

Proofline triggered a Deep Review because the change involved a database migration. The detailed Proofline review was inspected and confirmed to contain zero findings.

No false positives or actionable findings were recorded for this case.

### CI Status

Both recorded GitHub checks passed successfully.

### Case 07 Conclusion

All three reviewers completed the Case 07 review with zero reported findings. The original Proofline review was verified, resolving the previously recorded discrepancy.

This case is included in the cross-case comparison with zero findings for all three providers.

---

## Case 08 — Include a Total in the Activity Summary

### Review Configuration

| Attribute | Value |
|---|---|
| Case | 08 |
| Repository | `AryanUbale7/ai-review-run-01` |
| Pull Request | #8 |
| Base branch | `main` |
| Head branch | `pilot/08` |
| Context | Code-only |

### Recorded Review Attempts

| Provider | Trigger | Outcome | Findings | Valid | False positives |
|---|---|---|---:|---:|---:|
| Proofline | GitHub PR | Completed | 0 | 0 | 0 |
| CodeRabbit | GitHub PR | Completed | 0 | 0 | 0 |
| Qodo | Manual `/agentic_review` | Completed | 0 | 0 | 0 |

### Finding Assessment

Neither Proofline nor CodeRabbit generated an actionable finding.

The change adds total counts for delivered and pending deliveries to the activity summary.

Qodo also completed the review without reporting bugs, rule violations or requirement gaps.

### CI Status

Both recorded GitHub checks passed successfully.

### Case 08 Conclusion

All three reviewers completed without reporting actionable issues. This is a no-finding result and does not independently establish that the implementation is defect-free. Manual validation is still pending.

---

# 5. Cross-Case Comparison

The following table consolidates the recorded outcomes across all eight benchmark cases. The Case 07 Proofline discrepancy has been resolved by inspecting the original review.

| Case | Proofline | CodeRabbit | Qodo |
|---|---|---|---|
| 01 | 1 finding | 1 finding | 3 findings (1 confirmed; 2 require verification) |
| 02 | 1 finding | 1 finding | 1 valid finding |
| 03 | 1 finding | 1 finding | 1 valid finding |
| 04 | No actionable findings | No actionable findings | No actionable findings |
| 05 | No actionable findings | No actionable findings | No actionable findings |
| 06 | 1 finding | 2 comments | 2 valid findings |
| 07 | No actionable findings (0) | No actionable findings (0) | No actionable findings (0) |
| 08 | No actionable findings | No actionable findings | No actionable findings |

## 5.1 Preliminary Interpretation

- All three reviewers are recorded as identifying issues in Cases 01, 02, 03 and 06.
- All three reviewers completed Cases 04, 05, 07 and 08 without reporting actionable findings.
- Qodo reported two additional Case 01 concerns: old-cursor compatibility and index support. Both require independent verification before they are counted as confirmed valid findings.
- Case 06 demonstrates the importance of checking both new queue routing and existing queued records.
- Similar comments may describe the same underlying defect; comments should not be treated as separate defects without checking their substance.

These observations are preliminary. Finding-level validity, duplicate handling, provider isolation and missed-defect assessments must be completed before calculating comparative performance rates.

---

# 6. Attempt-Level Metadata

The current experiment summary records provider outcomes, but the following metadata still needs to be collected for reproducibility.

| Metadata | Status |
|---|---|
| Base and head commit for every case | Partially recorded |
| Unique attempt/run identifiers | TODO |
| Review timestamps | TODO |
| Provider plan or model, if visible | TODO |
| Provider review settings | TODO |
| Original review comments | TODO |
| Retry identification | TODO |
| Repository indexing and shared-learning controls | TODO |

## 6.1 Suggested Attempt Identifiers

Assign stable local identifiers to each intentional review attempt:

- `C01-PROOFLINE-01`
- `C01-CODERABBIT-01`
- `C01-QODO-01`
- `C02-PROOFLINE-01`
- `C02-CODERABBIT-01`
- `C02-QODO-01`

Continue the same naming pattern for the remaining cases.

These identifiers are experiment-level labels and do not replace the original provider run identifiers.

---

# 7. Limitations

1. The experiment currently covers eight related changes in one application.
2. All recorded comparisons use the code-only context condition.
3. Deployment snapshot comparisons have not yet been completed.
4. Original review comments and attempt-level metadata are not yet fully recorded.
5. Some findings still require independent manual validation.
6. Provider model and plan information may not be visible.
7. All three providers used the same review repository, so provider-level repository isolation was not established; possible cross-review context or indexing effects remain unverified.
8. The current results should not be interpreted as a universal ranking of AI reviewers.
9. Similar comments from different reviewers may describe the same underlying defect and should not automatically be counted as separate defects.
10. AWS deployment was not performed because AWS account access and deployment credentials were unavailable. The successful GitHub Actions CI run only verifies packaging and local Lambda handler smoke testing; it does not demonstrate a live AWS deployment.
---

# 8. Remaining Work

- [ ] Complete independent manual validation of the recorded findings for Cases 01–06.
- [ ] Independently validate the additional Qodo findings for Case 01.
- [ ] Confirm whether CodeRabbit's two Case 06 comments represent separate defects or overlapping descriptions.
- [ ] Perform manual validation for Case 08.
- [ ] Record commit references for all cases.
- [ ] Record attempt IDs and timestamps.
- [ ] Preserve original review comments and review links.
- [ ] Verify retry and webhook handling.
- [ ] Document provider settings and isolation limitations.
- [ ] Expand the reviewer comparison if additional providers are available.
- [x] Execute the GitHub Actions CI workflow, including application packaging and Lambda handler smoke testing.
- [ ] Complete the AWS deployment, live verification and stack deletion exercise.
- [ ] Prepare the final reproducible report.
- [ ] Record the narrated walkthrough.

---

# 9. Final Status

The initial reviewer comparison has been executed across all eight benchmark cases using Proofline, CodeRabbit and Qodo.

The experiment has captured reviewer findings, no-finding reviews and differences in reported issues. The current results are preliminary.

The Case 07 Proofline count discrepancy has been resolved by inspecting the original review, which showed zero findings. The report remains preliminary until the outstanding manual assessments, attempt-level metadata, provider-isolation documentation and deployment evidence have been addressed.

The GitHub Actions CI workflow completed successfully on commit `807d48c`. The workflow packaged the application and successfully executed the Lambda handler smoke test, producing a local smoke-test result with one delivered report and zero pending deliveries.

The live AWS deployment, verification and stack deletion steps remain incomplete because AWS account access and the required deployment configuration were unavailable. No live AWS deployment is claimed.