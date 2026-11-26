# AI Review Benchmark — Experiment Report

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

Each benchmark case is reviewed through its corresponding application-only branch.

The benchmark implementation, experiment notes and grading information are kept outside the reviewer-facing application changes.

The reviewers are evaluated using the same case changes under the code-only context condition.

Potential shared learning, repository indexing and access to earlier review discussions should be documented where those settings are visible.

**Isolation verification — pending:** Record provider indexing, shared-learning settings and access limitations.

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
2. **Old cursor compatibility:** The cursor meaning changes from an ID to a `scheduled_at` timestamp, but the CLI still accepts the old cursor format. Verify whether old cursor values are supported or explicitly versioned.
3. **Index support:** The query filters and sorts using `scheduled_at`, but the reported migration does not appear to add a supporting index. Verify the migration and query plan before grading this as a confirmed performance issue.

### Commit References

- Base commit: `9b5ac64ab0d3ba414c5edec1e64fcdf8b3b19a1a`
- Head commit: `b42257746fa78caad922340082a2a27ed00cdeb8`
- Pull Request: #1
- Repository: `AryanUbale7/ai-review-run-01`

### Case 01 Conclusion

Proofline and CodeRabbit identified the same valid pagination issue. Qodo reported the same core issue and two additional concerns that require further verification.

This result represents one case and should not be generalized to overall reviewer performance.

---

## Case 02 — Stream the Report Export

### Review Configuration

| Attribute | Value |
|---|---|
| Case | 02 |
| Repository | `AryanUbale7/ai-review-run-01` |
| Pull Request | #2 |
| Base branch | `main` |
| Head branch | `pilot/02` |
| Context | Code-only |

### Recorded Review Attempts

| Provider | Trigger | Outcome | Findings | Valid | False positives |
|---|---|---|---:|---:|---:|
| Proofline | GitHub PR | Completed | 1 | 1 | 0 |
| CodeRabbit | GitHub PR | Completed | 1 | 1 | 0 |
| Qodo | Manual `/agentic_review` | Completed | 1 | 1 | 0 |

### Finding Assessment

The export implementation joins database values using commas without proper CSV quoting.

Report names containing commas, double quotes or newlines can therefore break the exported CSV structure. Using Python's `csv.writer` while preserving the streaming behaviour is an appropriate solution.

### Qodo Finding

- **Finding:** Exported reports break when a name has a comma or quote.
- **Severity:** High.
- **Assessment:** Valid.
- **Evidence:** Qodo identified the issue in `reports.py`, in the `write_export` implementation.
- **Reason:** The implementation manually joins values with commas and does not escape commas, double quotes or newlines. This can produce malformed CSV output.

### Proofline and CodeRabbit

Both reviewers are recorded as having identified the CSV formatting issue.

- Proofline: one finding, assessed as valid.
- CodeRabbit: one finding, assessed as valid.

### Case 02 Conclusion

All three recorded reviewers identified the CSV formatting issue. Qodo's finding is supported by the implementation shown in the review comment.

---

## Case 03 — Batch Queued Deliveries

### Review Configuration

| Attribute | Value |
|---|---|
| Case | 03 |
| Repository | `AryanUbale7/ai-review-run-01` |
| Pull Request | #3 |
| Base branch | `main` |
| Head branch | `pilot/03` |
| Context | Code-only |

### Recorded Review Attempts

| Provider | Trigger | Outcome | Findings | Valid | False positives |
|---|---|---|---:|---:|---:|
| Proofline | GitHub PR | Completed | 1 | 1 | 0 |
| CodeRabbit | GitHub PR | Completed | 1 | 1 | 0 |
| Qodo | Manual `/agentic_review` | Completed | 1 | 1 | 0 |

### Finding Assessment

The batching loop processes only complete batches of 20.

When fewer than 20 jobs remain, those jobs may be skipped. This can leave due jobs pending and cause the worker to report fewer deliveries than expected.

The loop should process the final partial batch as well.

### Qodo Finding

- **Finding:** Low-volume queues never deliver reports.
- **Severity:** High.
- **Assessment:** Valid.
- **Evidence:** Qodo identified the loop in `reports.py` around lines 78–79.
- **Reason:** `deliver_due` iterates only over complete groups of 20. A queue with fewer than 20 due jobs is not processed, and the remaining jobs can stay pending.

### Case 03 Conclusion

Proofline, CodeRabbit and Qodo are all recorded as identifying the final partial-batch processing issue. The finding is a functional correctness issue because due jobs can remain pending.

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

### Case 05 — Increase export batch size

- Reviewer: Qodo
- Pull Request: #5
- Bugs reported: 0
- Rule violations: 0
- Requirement gaps: 0
- Valid findings: 0
- False positives: 0
- Result: Qodo reported no material issues.
- Notes: No findings were reported for this case.
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
| Proofline | GitHub PR | Completed (Deep Review) | 0 in case details; summary conflict | Pending verification | — |
| CodeRabbit | GitHub PR | Completed | 0 | 0 | 0 |
| Qodo | Manual `/agentic_review` | Completed | 0 | 0 | 0 |

### Review Assessment

Neither CodeRabbit nor Qodo reported an actionable finding.

Proofline triggered a Deep Review because the change involved a database migration. The case-specific notes record zero findings, but the earlier cross-case summary records one Proofline finding for Case 07.

**Data consistency issue:** Verify the original Proofline review before finalizing the Case 07 count. Do not use this case in comparative calculations until the discrepancy is resolved.

### CI Status

Both recorded GitHub checks passed successfully.

### Case 07 Conclusion

CodeRabbit and Qodo completed without actionable findings. Proofline's final finding count requires verification against the original review.

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

The following table consolidates the currently recorded outcomes. Case 07 contains an unresolved Proofline count discrepancy and is marked accordingly.

| Case | Proofline | CodeRabbit | Qodo |
|---|---|---|---|
| 01 | 1 finding | 1 finding | 3 findings (1 confirmed; 2 require verification) |
| 02 | 1 finding | 1 finding | 1 valid finding |
| 03 | 1 finding | 1 finding | 1 valid finding |
| 04 | No actionable findings | No actionable findings | No actionable findings |
| 05 | No actionable findings | No actionable findings | No actionable findings |
| 06 | 1 finding | 2 comments | 2 valid findings |
| 07 | **Conflicting record — verify** | No actionable findings | No actionable findings |
| 08 | No actionable findings | No actionable findings | No actionable findings |

## 5.1 Preliminary Interpretation

- All three reviewers are recorded as identifying issues in Cases 01, 02, 03 and 06.
- Case 07 shows a difference in recorded outcomes, but the Proofline count must be reconciled before drawing a conclusion.
- All three reviewers completed without actionable findings in Cases 04, 05 and 08.
- Qodo reported additional findings in Case 01 that require independent verification.
- Case 06 demonstrates the importance of checking both new queue routing and existing queued records.

These observations are preliminary. Finding-level validity, duplicate handling and missed-defect assessments must be completed before calculating comparative performance rates.

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
7. Case 07 contains a conflicting Proofline finding count that must be resolved.
8. The current results should not be interpreted as a universal ranking of AI reviewers.
9. Similar comments from different reviewers may describe the same underlying defect and should not automatically be counted as separate defects.

---

# 8. Remaining Work

- [ ] Inspect and grade the original Proofline finding for Case 07.
- [ ] Verify the original findings for Cases 01–06.
- [ ] Independently validate the additional Qodo findings for Case 01.
- [ ] Confirm whether CodeRabbit's two Case 06 comments represent separate defects or overlapping descriptions.
- [ ] Perform manual validation for Case 08.
- [ ] Record commit references for all cases.
- [ ] Record attempt IDs and timestamps.
- [ ] Preserve original review comments and review links.
- [ ] Verify retry and webhook handling.
- [ ] Document provider settings and isolation limitations.
- [ ] Expand the reviewer comparison if additional providers are available.
- [ ] Complete the CI and AWS deployment exercise.
- [ ] Prepare the final reproducible report.
- [ ] Record the narrated walkthrough.

---

# 9. Final Status

The initial reviewer comparison has been executed across all eight benchmark cases using Proofline, CodeRabbit and Qodo.

The experiment has captured reviewer findings, no-finding reviews and differences in reported issues. The current results are preliminary.

The report should be treated as complete only after the outstanding manual assessments, Case 07 discrepancy, attempt-level metadata and deployment evidence have been addressed.
