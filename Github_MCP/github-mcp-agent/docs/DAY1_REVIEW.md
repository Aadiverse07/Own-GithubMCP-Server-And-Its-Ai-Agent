# Day 1 Consistency Review

Reviewed: `ARCHITECTURE.md`, `TOOLS_SPEC.md`, `EVALUATION_SPEC.md` (as uploaded). Corrected versions are v1.1.

## Errors found and fixed

| # | Severity | Problem | Fix |
|---|----------|---------|-----|
| 1 | High | **24 of 50 tasks used arguments that do not exist in the tool schemas**: `direction`, `sort`, `creator` (list tools), `branch` instead of `branch_name` (4 create_branch tasks), `message` instead of `commit_message` (5 file tasks), `maintainer_can_modify` (T39). | All tasks regenerated; a validator confirms every argument exists for its tool and every required parameter is covered. |
| 2 | High | **Issue and PR numbers share one sequence in GitHub.** The fixture had issues 1-4 *and* PR #1, which is impossible. Tasks T35, T36, T44 pointed at the wrong number. | Fixture seeded issues first, so the baseline PR is **#5**. Tasks updated. |
| 3 | High | **T39 could never succeed.** It created a PR from `feature/login` into `main`, but PR #1 already existed for that pair (GitHub returns 422). | T39 now opens `develop` into `main`. |
| 4 | High | **The PR itself was unseedable.** `feature/login` had no commit ahead of `main`, and GitHub rejects PRs with no commits between head and base. | Fixture now defines a commit graph: `develop` and `feature/login` are each one commit ahead of `main`. |
| 5 | High | **`list_commits` was a tool with zero tasks.** | 3 commit tasks added (T08, T22, T34); all 16 tools now covered. |
| 6 | Medium | **T17 was a no-op.** It closed issue 2, which is already closed in the fixture. | Now closes issue 4. |
| 7 | High | **Reset only ran between whole runs**, but 16 tasks change state later tasks read (README content, issue 1 title, open PR list). `ARCHITECTURE.md` said "between tasks or runs" while the eval spec said "between runs". | Tasks carry a `mutates` flag; a targeted restore runs after each mutating task. Both documents now agree. |
| 8 | Medium | **Reset was not feasible with the stated credentials.** The agent token is fine-grained and repo-scoped, yet reset must force-push and delete. Delete-and-recreate would also restart numbering. | Separate admin credential for reset only; restore in place; limits of the REST API (issues and PRs cannot be deleted) documented. |
| 9 | Medium | **Classification gaps:** no outcome for "model answered without calling a tool" or "hallucinated tool name"; overlap between C1 `bad_arguments` and C3 `not_found`; the decision tree never checked arguments. | B sub-types added; explicit precedence (arguments compared before the execution error); new decision tree; post-conditions for write tasks. |
| 10 | Medium | **Metric definitions contradicted each other.** "Eligible tasks" vs. 50 in the example; argument accuracy counted optional defaults; the first-call rule in `ARCHITECTURE.md` conflicted with the "any deviation" rule for multi-step. | Metrics rewritten with explicit denominators; defaults never listed in `expected_args`; first-call accuracy reported alongside sequence accuracy. |
| 11 | Medium | **Held-out set (10 tasks) was required but never identified.** | Fixed IDs (T06, T11, T15, T17, T22, T26, T31, T39, T44, T49) and rules for model selection on the 40 development tasks only. |
| 12 | Medium | **Priority risk uncovered.** `TOOLS_SPEC` ranks `get_issue` vs. `get_pull_request` (bare `#N`) as risk #1 and recommends negative tasks, but the benchmark had none. | 5-task probe set outside the 50 (§45), because those cases have no single correct tool. |
| 13 | Low | `TOOLS_SPEC`: D2 said "latest 10 comments" while the description said "first 10", with a leftover inline "Spec correction" note. | Unified to "first 10". |
| 14 | Low | `TOOLS_SPEC`: write-safety row pointed at §6 (confusability matrix) instead of §8 (confirmation policy). | Fixed. |
| 15 | Low | `TOOLS_SPEC`: `list_repositories` vs. allowlist left open, which affected T05-T07. | Output filtered to the allowlist (D9). |
| 16 | Low | `ARCHITECTURE.md`: stale `create_repository` caveat (no such tool); open questions already answered elsewhere; config table missing `DESCRIPTION_VERSION`, `PROMPT_VERSION`, repair and thinking settings. | Cleaned up; resolved questions moved to a decisions table. |
| 17 | Low | Several tasks were weak or leaky: T33 said "already exists"; T43's wording allowed `get_repository` alone to satisfy it; T08/T22/T34 were redundant. | Reworded or replaced. |
| 18 | Low | Example values used `fixture-user/demo-repo` while tasks used `testuser/mcp-fixture-repo`. | One placeholder everywhere. |

## Judgment calls (change if you disagree)

* Primary benchmark runs with **zero repair attempts**; one-repair runs are a separate condition.
* **3 repeats** per condition, with a rule of thumb that differences of three tasks or fewer are noise.
* Bare-`#N` and unsupported-action requests live in a separate probe set rather than the scored 50.
* Task IDs stay stable, so category membership comes from the `category` field, not the ID range.

## Still open (need the real account / Day 2)

1. `qwen3:8b` thinking mode on or off, and `num_ctx` (4096 vs. 8192).
2. Which token type and permissions the reset script needs on the throwaway account.
3. How `list_repositories` behaves with a fine-grained token scoped to one repository.
4. Every ⚠ item in `TOOLS_SPEC.md` (a live smoke test is planned for Day 2).
5. Replace the `testuser` placeholder once the fake account exists.
