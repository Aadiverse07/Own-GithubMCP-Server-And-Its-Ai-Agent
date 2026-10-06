# GitHub MCP Server + AI Agent

# Evaluation & 50-Task Benchmark Specification

**Document:** `EVALUATION_SPEC.md`
**Phase:** Day 1 — Planning & Specification
**Benchmark Version:** v1
**Benchmark Size:** 50 tasks
**Purpose:** Measure MCP tool-selection reliability, argument generation, execution reliability, and the effect of tool descriptions on a small local Ollama model.
**Revision:** v1.1, after the Day 1 consistency review (see `DAY1_REVIEW.md`).

---

# 1. Evaluation Objective

The benchmark answers the project's core research question:

> **How reliably does an LLM select the correct MCP tool, and how do tool descriptions affect that reliability?**

The benchmark separates three fundamentally different failure modes:

1. The model selected the wrong tool.
2. The model selected the correct tool but generated incorrect arguments or execution failed.
3. The model selected the correct tool, generated correct arguments, and completed the task successfully.

The same fixed 50 tasks must be reused when comparing tool-description versions.

---

# 2. Outcome Classification

Every task receives exactly one primary outcome.

## A — Correct Tool + Task Completed

The model:

* selected the expected tool, or the complete expected sequence for a multi-step task;
* generated valid and semantically correct arguments;
* executed successfully;
* produced the requested result or state.

**Classification:** `A`

Example:

```text
User:
"Show me issue 12."

Expected:
get_issue

Arguments:
repo = testuser/mcp-fixture-repo
issue_number = 12

Execution:
Successful

Result:
Correct issue returned

Classification:
A
```

---

## B — Wrong Tool Selected (`selection_failure`)

The model did not call the expected tool (or the expected tool sequence).

| Sub-type | Meaning |
| --- | --- |
| `wrong_tool` | A catalog tool other than the expected one was called first. |
| `no_tool_call` | The model answered in text and called no tool, although a tool was expected. |
| `unknown_tool` | The model requested a tool name that is not in the catalog. |
| `wrong_sequence` | Multi-step only: the executed tool order differs from the expected order (missing, extra, or reordered calls). |
| `unexpected_write` | A single-step task had a correct first call, but the agent then also called a write tool that was not expected. |

Example:

```text
Expected: get_issue
Selected: list_issues
Classification: B (wrong_tool)
```

### Rules

1. **Single-step tasks:** the first tool call decides selection. A wrong first selection remains **B** even if the model later recovers. Extra *read* calls after a correct first call are logged but do not change the class.
2. **Multi-step tasks:** the executed sequence must equal the expected sequence. Any deviation is **B** (`wrong_sequence`, or `wrong_tool` / `unknown_tool` / `no_tool_call` when that describes the deviation better). First-call accuracy is also reported separately, consistent with `ARCHITECTURE.md` A5.
3. **Stops:** if the agent stops on `limit` or `timeout` after only correct selections so far, the task is **C** (`other`); if it had already deviated, it is **B**.

---

# 3. C — Correct Tool Selected, Execution Failure

The model selected the correct tool (or sequence), but the task did not complete successfully. Failure types:

| Code | `failure_type` | Meaning | Model-attributable? |
| --- | --- | --- | --- |
| C1 | `bad_arguments` | Generated arguments do not match `expected_args`, a `presence_only_args` argument is missing or empty, or agent-side pre-validation rejected them. | Yes |
| C2 | `auth` | Arguments match, but GitHub rejected the token or its permissions. | No (harness) |
| C3 | `not_found` | Arguments match, but the resource does not exist. | No (fixture drift) |
| C4 | `rate_limit` | Arguments match, but a rate limit prevented execution. | No (harness) |
| C5 | `other` | Anything else: network, GitHub 5xx, server error, timeout or limit stop, post-condition mismatch (`state_mismatch`), server `validation` error although arguments match (`harness_defect`). | No |

## 3.1 Precedence rule (resolves the C1 / C3 overlap)

Arguments are compared **before** the execution error is considered:

1. If any scored argument differs from `expected_args`, the task is **C1**, even if GitHub then returned `not_found` (for example the model invented issue 999). The cause is the model's argument, not the fixture.
2. Only when arguments match is the server `error.type` mapped:

| Server `error.type` | `failure_type` |
| --- | --- |
| `auth` | `auth` |
| `not_found` | `not_found` (indicates fixture drift: every expected argument refers to a baseline resource) |
| `rate_limit` | `rate_limit` |
| `validation` | `other` with `harness_defect` (arguments were correct, so the schema or handler is wrong) |
| `unknown` | `other` |

3. Failures C2 to C5 are **not model-attributable**. They are reported separately, and the task should be re-run once after the cause is fixed (see §10, Adjusted Success).

---

# 4. Classification Decision Tree

Every result is classified in this order:

```text
1. Was any tool called?
   no, and a tool was expected ............ B (no_tool_call)
2. Was the expected tool / sequence selected?
   no ..................................... B (wrong_tool | unknown_tool | wrong_sequence | unexpected_write)
3. Do the arguments match expected_args (and are presence_only_args present)?
   no ..................................... C1 bad_arguments
4. Did the server return an error?
   yes .................................... C2 / C3 / C4 / C5 per the table in 3.1
5. Does the post-condition hold? (see 4.1)
   no ..................................... C5 other (state_mismatch)
6. Otherwise ............................... A (success)
```

## 4.1 Post-conditions ("task completed")

* **Read tasks:** the envelope has `success = true`.
* **Write tasks (`mutates = true`):** after the call, the fixture is inspected with the admin credential (see §25) and must match `expected_args` (e.g. the file has the expected content, the issue is closed, the branch exists with the expected source). Free-text values that are `presence_only_args` are only checked for non-empty content.

---

# 5. Multi-Step Task Scoring

## Decision: No Partial Credit for Primary Task Success

A multi-step task is considered successful only when:

1. every required tool is selected;
2. the tools are selected in the correct order;
3. every required argument is correct;
4. every tool execution succeeds;
5. the final requested state/result is achieved.

Example:

```text
Expected:

get_file
    ↓
update_file

Actual:

get_file
    ↓
create_file
```

Result:

```text
B — wrong_tool
```

Even though the first step was correct.

---

## Why No Partial Credit?

The project evaluates **reliable end-to-end agent behavior**.

If a task requires:

```text
read → modify → verify
```

and the agent only performs:

```text
read → modify
```

the complete task was not successfully completed.

If the sequence is correct but an argument in some step is wrong, the task is **C1**, not B.

Diagnostic metrics should still record step-level performance.

For example:

```text
Expected steps: 3
Correct steps: 2

Step accuracy = 66.7%
Overall task = failure
```

This provides useful diagnostic information without falsely labeling the task successful.

---

# 6. Metrics

All rates are computed per run (see §27 for repeats).

## 6.1 Tool-Selection Accuracy

$$
ToolSelectionAccuracy =
\frac{\text{tasks not classified B}}
{50}
\times 100
$$

This is the complement of the Wrong-Tool Rate (§7). Three views are reported side by side:

* **Single-step selection accuracy:** the same formula over the 45 single-step tasks.
* **Sequence accuracy (§11):** over the 5 multi-step tasks.
* **First-call accuracy:** over all 50 tasks, whether the *first* call matched the expected first tool.

Example:

```text
45 tasks not classified B
50 total tasks

Tool-selection accuracy = 45 / 50 × 100 = 90%
```

---

# 7. Wrong-Tool Rate

$$
WrongToolRate =
\frac{\text{tasks classified B}}
{50}
\times 100
$$

Lower is better. Report the sub-type counts (`wrong_tool`, `no_tool_call`, `unknown_tool`, `wrong_sequence`, `unexpected_write`) and a confusion matrix of expected vs. selected tool.

---

# 8. Argument Accuracy

Computed only over tasks where the tool or sequence was selected correctly, since otherwise there is nothing to compare.

$$
ArgumentAccuracy =
\frac{\text{scored arguments that match}}
{\text{scored arguments}}
\times 100
$$

**What is scored:** every key in `expected_args`, plus the presence (non-empty) of every key in `presence_only_args`.

**What is not scored:** optional parameters the request does not determine. `expected_args` therefore never lists defaults (for example `limit`, `sort`, `state`, `branch`). If the model supplies an extra optional argument, it is not penalized unless it contradicts the request.

**Repository:** users rarely name the repository, so `repo` is resolved from the default repository stated in the system prompt (`GITHUB_DEFAULT_REPO`). A wrong or missing `repo` is a scored mismatch.

Values are compared semantically (§38): for example `"Main"` vs `"main"` is judged by branch identity, and free text such as a quoted title must match exactly after whitespace trimming.

---

# 9. Execution Success Rate

$$
ExecutionSuccessRate =
\frac{\text{tasks classified A}}
{\text{tasks with correct tool and correct arguments}}
\times 100
$$

This separates LLM failures from MCP / PyGithub / GitHub failures: a low value here, with a high selection and argument accuracy, points at the server or fixture.

---

# 10. Overall Task Success

$$
OverallTaskSuccess =
\frac{\text{tasks classified A}}
{50}
\times 100
$$

This is the primary end-to-end benchmark metric.

**Adjusted Success (secondary).** Failures C2 to C5 are not attributable to the model (§3.1). After they are fixed and re-run once, report:

$$
AdjustedSuccess =
\frac{\text{tasks classified A}}
{50 - \text{persistent C2..C5 tasks}}
\times 100
$$

Always report both numbers and the count of excluded tasks.

---

# 11. Multi-Step Sequence Accuracy

$$
SequenceAccuracy =
\frac{\text{multi-step tasks with the exact expected tool sequence}}
{5}
\times 100
$$

Sequence accuracy concerns tool selection only. Multi-step tasks are classified A only if arguments and post-conditions are also correct.

---

# 12. Category-Level Accuracy

Results should also be reported independently for:

* repository tasks;
* issue tasks;
* branch tasks;
* file tasks;
* commit tasks;
* pull-request tasks;
* multi-step tasks;
* ambiguous/casual tasks.

Category accuracy is the share of tasks in that category classified A. Because categories have only 3 to 12 tasks, report counts (e.g. 5 / 7) next to percentages.

This helps identify specific weaknesses.

For example:

```text
Repository:       100%
Issues:            91%
Branches:          83%
Files:             75%
Pull Requests:     88%
Multi-step:        60%
Ambiguous:         80%
```

---

# 13. Latency Metrics

Record:

* total request latency;
* LLM/tool-selection latency;
* individual tool execution latency.

Report:

* mean;
* median;
* p95.

Latency does **not** affect correctness.

It is a diagnostic metric.

---

# 14. Recovery Rate

A useful secondary metric is recovery rate.

$$
RecoveryRate =
\frac{\text{wrong-tool tasks eventually reaching correct final state}}
{\text{wrong-tool tasks}}
\times 100
$$

Important:

A recovered task is **still classified B** in the primary benchmark.

Recovery rate only tells us how often the agent can recover from an initial mistake.

---

# 15. Task Record JSON Schema

Every benchmark task must follow this structure:

```json
{
  "id": "T11",
  "category": "issues",
  "request": "Report a problem: the login page crashes on mobile.",
  "expected_tool": "create_issue",
  "expected_args": { "repo": "testuser/mcp-fixture-repo" },
  "presence_only_args": ["title"],
  "type": "single_step",
  "fixture_dependency": true,
  "mutates": true,
  "held_out": true,
  "notes": "New problem with no issue number: create_issue, not add_issue_comment."
}
```

## Fields

| Field                | Type         | Required | Description                                                                                           |
| -------------------- | ------------ | -------- | ----------------------------------------------------------------------------------------------------- |
| `id`                 | string       | Yes      | Stable task identifier. The `category` field, not the ID range, defines category membership.          |
| `category`           | string       | Yes      | `repository`, `issues`, `branches`, `files`, `commits`, `pull_requests`, `multi_step`, `ambiguous_casual` |
| `request`            | string       | Yes      | Natural-language request given to the agent                                                           |
| `expected_tool`      | string/array | Yes      | Expected tool, or ordered tool sequence                                                               |
| `expected_args`      | object/array | Yes      | Arguments determined by the request (object for single-step, one object per step for multi-step). Parameter names must match `TOOLS_SPEC.md` exactly. Defaults are never listed. |
| `presence_only_args` | array        | No       | Required arguments whose value the request does not determine (e.g. a generated title). Only checked for non-empty. For multi-step tasks, one list per step. |
| `type`               | string       | Yes      | `single_step` or `multi_step`                                                                         |
| `fixture_dependency` | boolean      | Yes      | Whether baseline fixture state is required                                                            |
| `mutates`            | boolean      | Yes      | Whether the task changes GitHub state; if true, the fixture is restored afterwards (§25)             |
| `held_out`           | boolean      | Yes      | Member of the 10-task held-out set (§34)                                                              |
| `notes`              | string       | Yes      | Evaluator-only explanation                                                                            |

A validator script must reject any task whose `expected_args` contains a parameter not defined for the expected tool in `TOOLS_SPEC.md`.

---

# 16. Results Record JSON Schema

Each execution produces a result record.

```json
{
  "task_id": "T01",
  "run_index": 1,
  "selected_tools": ["get_repository"],
  "args": [{ "repo": "testuser/mcp-fixture-repo" }],
  "status": "A",
  "selection_subtype": null,
  "failure_type": null,
  "model_attributable": null,
  "latency": { "total_ms": 1200, "tool_ms": 300 },
  "raw_output": {}
}
```

Required metadata on every run:

```json
{
  "benchmark_version": "v1",
  "tool_description_version": "tools-v1",
  "prompt_version": "prompt-v1",
  "model": "qwen3:8b",
  "temperature": 0,
  "seed": 42,
  "num_ctx": 8192,
  "thinking": "off",
  "repair_attempts": 0
}
```

(The model and thinking values are examples until the model is pinned.)

## Status Values

```text
A = successful
B = selection failure (wrong tool, no tool, unknown tool, wrong sequence, unexpected write)
C = execution failure
```

## Failure Types

```text
null
bad_arguments
auth
not_found
rate_limit
other
```

---

# 17. Benchmark Distribution

The benchmark contains exactly 50 tasks.

| Category         | Number |
| ---------------- | -----: |
| Repository       |      7 |
| Issues           |     12 |
| Branches         |      5 |
| Files            |      7 |
| Commits          |      3 |
| Pull Requests    |      6 |
| Multi-Step       |      5 |
| Ambiguous/Casual |      5 |
| **Total**        | **50** |

Every one of the 16 tools is the expected tool of at least one task. The ambiguous/casual category represents language style and still exercises the operational tools.

Tasks that are intentionally ambiguous in a way that has no single correct tool (bare `#N` references) or that ask for unsupported actions are **not** part of the 50; they form the separate probe set in §45.

---

# 18. Canonical 50 Tasks

```json
[
  {
    "id": "T01",
    "category": "repository",
    "request": "Tell me what this project repo is about.",
    "expected_tool": "get_repository",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Singular repository lookup."
  },
  {
    "id": "T02",
    "category": "repository",
    "request": "What is the default branch for the fixture project?",
    "expected_tool": "get_repository",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Answer lives in repository metadata, not list_branches."
  },
  {
    "id": "T03",
    "category": "repository",
    "request": "Is the fixture repo public or private?",
    "expected_tool": "get_repository",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Single repository metadata request."
  },
  {
    "id": "T04",
    "category": "repository",
    "request": "Show me the details for testuser/mcp-fixture-repo.",
    "expected_tool": "get_repository",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Explicit repository lookup."
  },
  {
    "id": "T05",
    "category": "repository",
    "request": "Which repositories can I access?",
    "expected_tool": "list_repositories",
    "expected_args": {},
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Collection request. No argument is determined by the request; defaults are not scored."
  },
  {
    "id": "T06",
    "category": "repository",
    "request": "Give me my private repositories.",
    "expected_tool": "list_repositories",
    "expected_args": {
      "visibility": "private"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": true,
    "notes": "Visibility-filtered collection."
  },
  {
    "id": "T07",
    "category": "repository",
    "request": "List my repos, twenty at most.",
    "expected_tool": "list_repositories",
    "expected_args": {
      "limit": 20
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Natural-language limit."
  },
  {
    "id": "T08",
    "category": "commits",
    "request": "What are the most recent commits on main?",
    "expected_tool": "list_commits",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "branch": "main"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "History request. Covers the previously untested list_commits tool."
  },
  {
    "id": "T09",
    "category": "issues",
    "request": "Show me issue 1.",
    "expected_tool": "get_issue",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "issue_number": 1
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Specific issue lookup."
  },
  {
    "id": "T10",
    "category": "issues",
    "request": "What is issue 2 about?",
    "expected_tool": "get_issue",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "issue_number": 2
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Specific issue lookup (closed issue)."
  },
  {
    "id": "T11",
    "category": "issues",
    "request": "Report a problem: the login page crashes on mobile.",
    "expected_tool": "create_issue",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo"
    },
    "presence_only_args": [
      "title"
    ],
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": true,
    "held_out": true,
    "notes": "New problem with no issue number: create_issue, not add_issue_comment. Title is model-written, so presence-only."
  },
  {
    "id": "T12",
    "category": "issues",
    "request": "Find all the open issues.",
    "expected_tool": "list_issues",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "state": "open"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Collection request, state stated."
  },
  {
    "id": "T13",
    "category": "issues",
    "request": "Which issues are closed?",
    "expected_tool": "list_issues",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "state": "closed"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Collection request, state stated."
  },
  {
    "id": "T14",
    "category": "issues",
    "request": "List the bugs.",
    "expected_tool": "list_issues",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "labels": [
        "bug"
      ]
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Label-filtered collection. State is not stated, so it is not scored."
  },
  {
    "id": "T15",
    "category": "issues",
    "request": "Open issue 1 and tell me what it says.",
    "expected_tool": "get_issue",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "issue_number": 1
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": true,
    "notes": "Trap: 'open' must be read as view, not reopen (update_issue)."
  },
  {
    "id": "T16",
    "category": "issues",
    "request": "Change issue 1's title to 'Updated login bug'.",
    "expected_tool": "update_issue",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "issue_number": 1,
      "title": "Updated login bug"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": true,
    "held_out": false,
    "notes": "Edit an existing issue."
  },
  {
    "id": "T17",
    "category": "issues",
    "request": "Close issue 4.",
    "expected_tool": "update_issue",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "issue_number": 4,
      "state": "closed"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": true,
    "held_out": true,
    "notes": "Close via update_issue. Issue 4 is open in the baseline (issue 2 is already closed, so it would be a no-op)."
  },
  {
    "id": "T18",
    "category": "issues",
    "request": "Put a comment on issue 3 saying 'Reviewed by the test agent'.",
    "expected_tool": "add_issue_comment",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "issue_number": 3,
      "body": "Reviewed by the test agent"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": true,
    "held_out": false,
    "notes": "Comment on an existing issue."
  },
  {
    "id": "T19",
    "category": "issues",
    "request": "Create a new issue titled 'Fixture test issue' with body 'Created by benchmark'.",
    "expected_tool": "create_issue",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "title": "Fixture test issue",
      "body": "Created by benchmark"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": true,
    "held_out": false,
    "notes": "Create a new issue."
  },
  {
    "id": "T20",
    "category": "issues",
    "request": "Find the open issues carrying the enhancement label.",
    "expected_tool": "list_issues",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "state": "open",
      "labels": [
        "enhancement"
      ]
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "State and label stated."
  },
  {
    "id": "T21",
    "category": "branches",
    "request": "What branches are in the project?",
    "expected_tool": "list_branches",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Collection request."
  },
  {
    "id": "T22",
    "category": "commits",
    "request": "Who last changed README.md?",
    "expected_tool": "list_commits",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "path": "README.md"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": true,
    "notes": "History of one file, not its content (get_file)."
  },
  {
    "id": "T23",
    "category": "branches",
    "request": "Make a branch called benchmark-branch from main.",
    "expected_tool": "create_branch",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "branch_name": "benchmark-branch",
      "from_branch": "main"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": true,
    "held_out": false,
    "notes": "Branch creation with explicit source."
  },
  {
    "id": "T24",
    "category": "branches",
    "request": "Start a new branch named feature/demo using main as its starting point.",
    "expected_tool": "create_branch",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "branch_name": "feature/demo",
      "from_branch": "main"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": true,
    "held_out": false,
    "notes": "Branch creation, varied phrasing."
  },
  {
    "id": "T25",
    "category": "branches",
    "request": "I only want to see the branches, not make one.",
    "expected_tool": "list_branches",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Easy contrast hint (list vs create)."
  },
  {
    "id": "T26",
    "category": "branches",
    "request": "Create a branch from develop called hotfix/demo.",
    "expected_tool": "create_branch",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "branch_name": "hotfix/demo",
      "from_branch": "develop"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": true,
    "held_out": true,
    "notes": "Non-default source branch."
  },
  {
    "id": "T27",
    "category": "files",
    "request": "Read the README for me.",
    "expected_tool": "get_file",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "path": "README.md"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "File read."
  },
  {
    "id": "T28",
    "category": "files",
    "request": "Show me what's inside config/app.json.",
    "expected_tool": "get_file",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "path": "config/app.json"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "File read."
  },
  {
    "id": "T29",
    "category": "files",
    "request": "Look at src/main.py on develop.",
    "expected_tool": "get_file",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "path": "src/main.py",
      "branch": "develop"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "File read on a non-default branch."
  },
  {
    "id": "T30",
    "category": "files",
    "request": "Create notes/benchmark.txt containing 'benchmark file' and commit it as 'Add benchmark notes'.",
    "expected_tool": "create_file",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "path": "notes/benchmark.txt",
      "content": "benchmark file",
      "commit_message": "Add benchmark notes"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": true,
    "held_out": false,
    "notes": "New file."
  },
  {
    "id": "T31",
    "category": "files",
    "request": "Change README.md to '# Fixture Updated' and commit the change as 'Update README'.",
    "expected_tool": "update_file",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "path": "README.md",
      "content": "# Fixture Updated",
      "commit_message": "Update README"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": true,
    "held_out": true,
    "notes": "Existing file: update_file, not create_file."
  },
  {
    "id": "T32",
    "category": "files",
    "request": "Add a brand-new file called notes/new.txt with 'hello'.",
    "expected_tool": "create_file",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "path": "notes/new.txt",
      "content": "hello"
    },
    "presence_only_args": [
      "commit_message"
    ],
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": true,
    "held_out": false,
    "notes": "New file. Commit message not stated, so presence-only."
  },
  {
    "id": "T33",
    "category": "files",
    "request": "Overwrite docs/usage.md so it only says '# Usage v2'.",
    "expected_tool": "update_file",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "path": "docs/usage.md",
      "content": "# Usage v2"
    },
    "presence_only_args": [
      "commit_message"
    ],
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": true,
    "held_out": false,
    "notes": "Existing file, natural phrasing with no 'already exists' hint. Commit message presence-only."
  },
  {
    "id": "T34",
    "category": "commits",
    "request": "Show me the history of src/main.py on develop.",
    "expected_tool": "list_commits",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "branch": "develop",
      "path": "src/main.py"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "History vs content (get_file) on a non-default branch."
  },
  {
    "id": "T35",
    "category": "pull_requests",
    "request": "Show me pull request 5.",
    "expected_tool": "get_pull_request",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "pr_number": 5
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Specific PR lookup. PR 5 is the baseline PR (numbers are shared with issues)."
  },
  {
    "id": "T36",
    "category": "pull_requests",
    "request": "What is PR 5 trying to change?",
    "expected_tool": "get_pull_request",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "pr_number": 5
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Specific PR lookup."
  },
  {
    "id": "T37",
    "category": "pull_requests",
    "request": "Find the open pull requests.",
    "expected_tool": "list_pull_requests",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "state": "open"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Collection request."
  },
  {
    "id": "T38",
    "category": "pull_requests",
    "request": "List PRs going into main.",
    "expected_tool": "list_pull_requests",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "base": "main"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Base-filtered collection."
  },
  {
    "id": "T39",
    "category": "pull_requests",
    "request": "Open a pull request from develop into main titled 'Develop sync'.",
    "expected_tool": "create_pull_request",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "title": "Develop sync",
      "head": "develop",
      "base": "main"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": true,
    "held_out": true,
    "notes": "Create PR. develop is ahead of main and has no PR, so the task is achievable (feature/login already has PR 5)."
  },
  {
    "id": "T40",
    "category": "pull_requests",
    "request": "I want to see the PRs, not create one.",
    "expected_tool": "list_pull_requests",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Easy contrast hint (list vs create)."
  },
  {
    "id": "T41",
    "category": "multi_step",
    "request": "Read README.md, then change it to '# Reviewed' and save that change.",
    "expected_tool": [
      "get_file",
      "update_file"
    ],
    "expected_args": [
      {
        "repo": "testuser/mcp-fixture-repo",
        "path": "README.md"
      },
      {
        "repo": "testuser/mcp-fixture-repo",
        "path": "README.md",
        "content": "# Reviewed"
      }
    ],
    "presence_only_args": [
      [],
      [
        "commit_message"
      ]
    ],
    "type": "multi_step",
    "fixture_dependency": true,
    "mutates": true,
    "held_out": false,
    "notes": "Required sequence: read, then update. Commit message of step 2 is presence-only."
  },
  {
    "id": "T42",
    "category": "multi_step",
    "request": "Check issue 3, then add the comment 'Checked by benchmark'.",
    "expected_tool": [
      "get_issue",
      "add_issue_comment"
    ],
    "expected_args": [
      {
        "repo": "testuser/mcp-fixture-repo",
        "issue_number": 3
      },
      {
        "repo": "testuser/mcp-fixture-repo",
        "issue_number": 3,
        "body": "Checked by benchmark"
      }
    ],
    "type": "multi_step",
    "fixture_dependency": true,
    "mutates": true,
    "held_out": false,
    "notes": "Required sequence: read, then comment."
  },
  {
    "id": "T43",
    "category": "multi_step",
    "request": "First tell me about the repo, then show me every branch it has.",
    "expected_tool": [
      "get_repository",
      "list_branches"
    ],
    "expected_args": [
      {
        "repo": "testuser/mcp-fixture-repo"
      },
      {
        "repo": "testuser/mcp-fixture-repo"
      }
    ],
    "type": "multi_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Two explicit asks in a fixed order."
  },
  {
    "id": "T44",
    "category": "multi_step",
    "request": "Find the open PRs, then inspect PR 5.",
    "expected_tool": [
      "list_pull_requests",
      "get_pull_request"
    ],
    "expected_args": [
      {
        "repo": "testuser/mcp-fixture-repo",
        "state": "open"
      },
      {
        "repo": "testuser/mcp-fixture-repo",
        "pr_number": 5
      }
    ],
    "type": "multi_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": true,
    "notes": "Collection followed by specific lookup."
  },
  {
    "id": "T45",
    "category": "multi_step",
    "request": "List the bugs, then read issue 1.",
    "expected_tool": [
      "list_issues",
      "get_issue"
    ],
    "expected_args": [
      {
        "repo": "testuser/mcp-fixture-repo",
        "labels": [
          "bug"
        ]
      },
      {
        "repo": "testuser/mcp-fixture-repo",
        "issue_number": 1
      }
    ],
    "type": "multi_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Collection followed by specific lookup."
  },
  {
    "id": "T46",
    "category": "ambiguous_casual",
    "request": "Can you pull up issue 2 for me?",
    "expected_tool": "get_issue",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "issue_number": 2
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Casual phrasing."
  },
  {
    "id": "T47",
    "category": "ambiguous_casual",
    "request": "What's going on with the bugs?",
    "expected_tool": "list_issues",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "labels": [
        "bug"
      ]
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Vague phrasing, collection intent."
  },
  {
    "id": "T48",
    "category": "ambiguous_casual",
    "request": "Can you open the README?",
    "expected_tool": "get_file",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "path": "README.md"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "'open' means read."
  },
  {
    "id": "T49",
    "category": "ambiguous_casual",
    "request": "Make me a new branch called try-this.",
    "expected_tool": "create_branch",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo",
      "branch_name": "try-this"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": true,
    "held_out": true,
    "notes": "Casual creation. Source branch not stated, so not scored."
  },
  {
    "id": "T50",
    "category": "ambiguous_casual",
    "request": "What's the PR situation right now?",
    "expected_tool": "list_pull_requests",
    "expected_args": {
      "repo": "testuser/mcp-fixture-repo"
    },
    "type": "single_step",
    "fixture_dependency": true,
    "mutates": false,
    "held_out": false,
    "notes": "Vague phrasing, collection intent."
  }
]
```

---

# 19. Fixture Repository Specification

All benchmark runs use:

```text
testuser/mcp-fixture-repo
```

(`testuser` is a placeholder; replace it everywhere, including the task file, once the throwaway account exists.)

This repository belongs to the dedicated throwaway GitHub account.

## Repository Metadata

```text
Owner: testuser
Repository: mcp-fixture-repo
Visibility: private
Default branch: main

Description:
Deterministic fixture repository for GitHub MCP benchmark.
```

---

# 20. Branches and Commit Graph

The baseline repository must contain exactly:

```text
main
develop
feature/login
```

```text
C1 "Initial fixture repository"  (README.md, config/app.json, src/main.py)
 └── C2 "Add fixture documentation"  (docs/usage.md)        <- main
      ├── D1 "Update main.py for develop"  (src/main.py)     <- develop
      └── L1 "Add login module"  (src/login.py)              <- feature/login
```

Both `develop` and `feature/login` are exactly one commit ahead of `main`, and `main` is not ahead of either. This is required so that:

* pull request 5 (`feature/login` into `main`) can exist;
* task T39 (`develop` into `main`) can be created, because GitHub rejects a pull request with no commits between head and base.

---

# 21. Files

The baseline repository must contain:

| File              | Branch                  | Content                                            |
| ----------------- | ----------------------- | -------------------------------------------------- |
| `README.md`       | `main` (and inherited)  | `# MCP Fixture Repository` + benchmark description |
| `config/app.json` | `main` (and inherited)  | Fixture JSON configuration                         |
| `src/main.py`     | `main`                  | Basic fixture Python function                      |
| `src/main.py`     | `develop`               | Different deterministic develop version            |
| `docs/usage.md`   | `main` (and inherited)  | Fixture documentation                              |
| `src/login.py`    | `feature/login`         | Minimal fixture login function                     |

Exact contents:

### `README.md`

```text
# MCP Fixture Repository

Deterministic repository for benchmark evaluation.
```

### `config/app.json`

```json
{
  "name": "mcp-fixture",
  "version": "1.0.0"
}
```

### `src/main.py` on `main`

```python
def main():
    return "fixture"
```

### `src/main.py` on `develop`

```python
def main():
    return "fixture-develop"
```

### `docs/usage.md`

```text
# Usage

Benchmark fixture documentation.
```

### `src/login.py` on `feature/login`

```python
def login():
    return "fixture-login"
```

---

# 22. Issues

Issues and pull requests share **one number sequence** in GitHub. The fixture must therefore be seeded in this order: issues 1 to 4 first, then the pull request, which receives number 5.

| Number | Title                  | State  | Labels          | Body                                      |
| -----: | ---------------------- | ------ | --------------- | ----------------------------------------- |
|      1 | Login bug              | open   | `bug`           | Login fails with the fixture credentials. |
|      2 | Improve documentation  | closed | `documentation` | Add clearer setup documentation.          |
|      3 | Add benchmark metadata | open   | `enhancement`   | Track benchmark metadata in the fixture.  |
|      4 | Minor cleanup          | open   | `chore`         | Clean up an unused fixture example.       |

These numbers stay stable across runs. Issues and PRs created by benchmark tasks receive numbers above 5; their exact numbers are never asserted.

---

# 23. Pull Requests

The baseline fixture must contain exactly one open pull request:

| Number | Title             | State | Head            | Base   |
| -----: | ----------------- | ----- | --------------- | ------ |
|      5 | Add login feature | open  | `feature/login` | `main` |

This PR is used by `get_pull_request`, `list_pull_requests`, task T44, and the probe set.

---

# 24. Fixture Commits

Baseline commits, by message:

```text
main:           Initial fixture repository
                Add fixture documentation
develop:        ...the two above, plus "Update main.py for develop"
feature/login:  ...the two above, plus "Add login module"
```

Exact SHA values are not part of correctness; they are recorded once in `fixtures/baseline.json` and used by the reset script. The benchmark compares semantic commit information (message, author, touched path), never generated SHAs.

---

# 25. Reset and Verification Procedure

## 25.1 Credentials

The agent's and the server's token is a fine-grained PAT limited to the fixture repo and has no admin rights. Resetting needs more (force-updating branches, deleting branches and comments). Reset therefore uses a **separate admin credential** (`GITHUB_ADMIN_TOKEN`) that is read only by `evaluation/fixtures/reset.py` and `verify.py` and never by the server or the agent.

Restore **in place**. Do not delete and recreate the repository: it would restart issue and PR numbering, and (⚠ unverified) a fine-grained token scoped to selected repositories may lose access to a recreated repository.

## 25.2 When it runs

```text
Before every complete run:   full reset + verify
After every task with mutates = true:   targeted restore + verify
```

Running reset only between whole runs is not enough: tasks such as T16, T17, T30, T31 and T39 change state that later tasks read (README content, issue 1's title, the list of open PRs). Tasks are executed in ID order.

## 25.3 What reset restores

```text
START
  ↓
Force-update main, develop, feature/login to the SHAs in baseline.json
  ↓
Delete every other branch
  ↓
Restore repository visibility, description, default branch (main)
  ↓
Restore issues 1-4 exactly (title, body, state, labels, assignees)
  ↓
Delete comments created after the baseline
  ↓
Close every issue or PR numbered above 5 (REST cannot delete them)
  ↓
Restore PR 5 (open, feature/login into main)
  ↓
Verify against this specification (verify.py)
  ↓
RUN
```

Leftover closed benchmark issues and PRs remain in the repository. This is acceptable because correctness never depends on list *contents* beyond the baseline items: read tasks are scored on tool, arguments and envelope success, and write tasks on their post-condition (§4.1). ⚠ Deleting issues via GitHub's GraphQL API may be possible with admin rights; verify before relying on it.

`verify.py` must fail the run if any baseline item (branch SHAs, files, issues 1-4, PR 5) deviates.

---

# 26. Safe and Repeatable Write Tasks

All write tasks are intentionally restricted to the throwaway fixture.

The benchmark must never run against:

* a production repository;
* a personal important repository;
* an organization repository;
* an unrelated GitHub account.

Write operations are safe because:

* the repository is disposable;
* all mutations are deterministic;
* the fixture is reset before every run;
* no merge operation exists in v1;
* no delete operation exists in v1 (the reset script's delete and force-update rights belong to a separate admin credential the agent never holds).

---

# 27. Reproducibility Rules

Reproducibility is critical because this is an experimental benchmark.

The following must remain fixed.

## Model

The final model must be selected from the approved candidates:

```text
llama3.1:8b
qwen3:8b
qwen2.5:7b
```

After model selection, the exact Ollama model tag must be pinned.

Example:

```text
qwen3:8b
```

No model switching is allowed during a description experiment.

## Model selection protocol

The model is chosen **before** any description experiment, using only the 40 development tasks (§34), the baseline descriptions `tools-v1`, and the same prompt, seed, `num_ctx` and thinking setting for every candidate. The held-out set is not touched during model selection.

## Repeats

Run each condition **3 times** (same seed, fresh fixture reset each time). Report the mean and the min-max range for every headline metric and list the tasks whose outcome flipped between runs. As a rule of thumb for a 50-task benchmark, a difference of three tasks (6 percentage points) or less between two description versions is within noise unless it is consistent across repeats.

## Repair attempts

The primary condition runs with **zero** agent repair attempts, so the numbers reflect raw model behavior. A run with one repair attempt (`ARCHITECTURE.md` §7) is a separate, labeled condition and is never mixed with the primary one.

---

# 28. Temperature

Always use:

```text
temperature = 0
```

for benchmark runs.

This minimizes stochastic variation.

---

# 29. Fixed Tool Descriptions

Each benchmark run must identify its tool-description version.

Example:

```text
tools-v1
tools-v2
tools-v3
```

For example:

```text
Benchmark v1
Tool descriptions v1
```

versus:

```text
Benchmark v1
Tool descriptions v2
```

The benchmark task set remains unchanged.

---

# 30. Benchmark Versioning

Use:

```text
v1
v2
v3
...
```

A new benchmark version is required when:

* task wording changes;
* tasks are added/removed;
* expected answers change;
* fixture semantics change;
* scoring rules change.

A tool-description-only change should **not** create a new task benchmark version.

Instead:

```text
benchmark_version = v1
tool_description_version = tools-v2
```

---

# 31. Result Version Metadata

Every benchmark run should contain:

```json
{
  "benchmark_version": "v1",
  "tool_description_version": "tools-v1",
  "model": "qwen3:8b",
  "temperature": 0
}
```

Recommended additional metadata:

```json
{
  "run_id": "2026-10-05-001",
  "timestamp": "2026-10-05T00:00:00Z",
  "server_version": "v1"
}
```

---

# 32. Fair Comparison Rules

When comparing tool-description versions:

### Keep identical:

* 50 benchmark tasks;
* task wording;
* fixture repository;
* fixture reset process;
* model;
* model tag;
* temperature;
* system prompt;
* MCP schemas;
* parameter names;
* execution limits;
* timeout;
* agent loop;
* result classifier.

### Change only:

```text
Tool descriptions
```

if the experiment is specifically testing description quality.

For example:

```text
tools-v1 → tools-v2
```

must not simultaneously change:

```text
model
+
system prompt
+
tool schema
+
task wording
```

Otherwise the experiment cannot reliably attribute improvements to the tool descriptions.

---

# 33. Benchmark Bias Controls

The benchmark must be protected from accidental overfitting.

## 33.1 Do Not Tune Descriptions to Exact Task Wording

Tool descriptions must explain general tool semantics.

Bad:

```text
Use this tool when the user says:
"What's going on with the bugs?"
```

Good:

```text
Use when the user wants to find, browse, or list multiple issues.
```

The description must generalize beyond individual benchmark sentences.

---

# 34. Held-Out Evaluation Set

At least 10 tasks should be treated as a held-out subset during tool-description development.

Split (fixed, flagged by `held_out` in the task file):

```text
40 development tasks
10 held-out tasks:
T06, T11, T15, T17, T22, T26, T31, T39, T44, T49
```

The held-out set covers repository, issue, commit, branch, file and pull-request tools, a multi-step task (T44), a casual request (T49), and the confusable pairs get/update issue (T15, T17), create_issue vs. comment (T11), create vs. update file (T31), create_branch vs. create_pull_request (T26, T39), and get_file vs. list_commits (T22).

The held-out tasks should contain:

* different wording;
* multiple tool families;
* confusable tool pairs;
* casual requests;
* at least one multi-step task.

The held-out set must not be used to choose between description variants during development.

Only after a candidate description version is frozen should the held-out set be evaluated.

---

# 35. Freeze Task Wording

Once benchmark v1 is frozen, the canonical task wording must not be changed based on model results.

For example, if:

```text
T33
"Overwrite docs/usage.md so it only says '# Usage v2'."
```

causes repeated failures, the task must not simply be rewritten to make it easier.

Changing it would invalidate comparisons with earlier runs.

A changed task becomes part of:

```text
benchmark v2
```

---

# 36. Do Not Remove Difficult Tasks

Difficult tasks are valuable.

Do not remove tasks merely because they produce low accuracy.

In particular, preserve:

```text
get_issue ↔ list_issues

get_issue ↔ update_issue

update_issue ↔ add_issue_comment

create_file ↔ update_file

list_pull_requests ↔ get_pull_request

create_branch ↔ create_pull_request
```

These pairs directly test semantic tool discrimination.

---

# 37. Avoid API Terminology Leakage

Benchmark requests should generally use natural user language.

Prefer:

```text
"Show me issue 2."
```

over:

```text
"Call get_issue with issue_number=2."
```

Prefer:

```text
"Can you open the README?"
```

over:

```text
"Use get_file on README.md."
```

The benchmark measures whether the model understands the user's intent, not whether it can copy tool names.

---

# 38. Avoid Raw Text Similarity Scoring

Argument correctness must be evaluated semantically.

Do not determine correctness solely through:

```text
string similarity
```

Instead compare the generated argument against the expected semantic value.

For example:

```text
Expected branch:
main
```

The agent must identify the correct branch rather than merely produce a syntactically valid branch string.

---

# 39. Separate Model Failures From System Failures

Example:

```text
Expected:
get_issue

Agent:
get_issue

Arguments:
Correct

GitHub:
500 Internal Server Error
```

This is:

```text
C5 — other
```

It is **not**:

```text
B — wrong tool
```

Similarly:

```text
Expected:
get_issue

Agent:
get_issue

Arguments:
issue_number = 999
```

where issue 999 does not exist and the task expected another number is:

```text
C1 — bad_arguments
```

If the task expected issue 99 and the model generated 999, the task is **C1 — bad_arguments**, because arguments are compared before the execution error (§3.1). A `not_found` result is classified **C3** only when the arguments match `expected_args` and the resource is still missing, which signals fixture drift.

---

# 40. Preserve Raw Evidence

Every benchmark result should preserve enough evidence to reproduce the classification.

At minimum:

```text
task_id
selected_tool
generated_arguments
status
failure_type
latency
raw_tool_output
```

Recommended additional information:

```text
model
temperature
benchmark_version
tool_description_version
execution_id
timestamp
```

This allows failed classifications to be audited later.

---

# 41. Recommended Final Benchmark Report

Every benchmark run should produce a summary similar to:

```text
========================================
GitHub MCP Benchmark
========================================

Benchmark: v1
Tools: tools-v2
Model: qwen3:8b
Temperature: 0

Total Tasks: 50   (illustrative values)
Runs: 3 (mean, min-max)

Overall Task Success:       42 / 50 = 84%
Adjusted Success:           42 / 49 = 86%   (1 excluded C2-C5)
First-Call Accuracy:        46 / 50 = 92%
Tool Selection Accuracy:    45 / 50 = 90%
Argument Accuracy:          94%
Execution Success Rate:     93%
Wrong Tool Rate:            10%

Multi-Step Accuracy:        60%
Repository Accuracy:        100%
Commit Accuracy:             67%
Issue Accuracy:              91%
Branch Accuracy:             83%
File Accuracy:               75%
PR Accuracy:                 88%
Ambiguous Accuracy:          80%

Mean Latency:               ...
Median Latency:             ...
P95 Latency:                ...

========================================
```

Then list every failed task:

```text
T14 — Wrong tool
T23 — Bad arguments
T33 — Wrong tool
...
```

---

# 42. Research Comparison

The main experiment should look like:

```text
                Same 50 Tasks
                     │
          ┌──────────┴──────────┐
          │                     │
      tools-v1               tools-v2
          │                     │
          ▼                     ▼
       Ollama                Ollama
      same model            same model
      temperature 0         temperature 0
          │                     │
          ▼                     ▼
      Results A              Results B
          │                     │
          └──────────┬──────────┘
                     ▼
              Compare Metrics
```

The key comparison is whether improved tool descriptions produce:

```text
higher tool-selection accuracy
+
lower wrong-tool rate
+
higher overall task success
```

without introducing regressions elsewhere.

---

# 43. Primary Success Criteria

For this project, the most important metrics are:

### 1. Overall Task Success

Measures true end-to-end reliability.

### 2. Tool-Selection Accuracy

Measures the core research question.

### 3. Wrong-Tool Rate

Directly measures semantic tool-selection failures.

### 4. Argument Accuracy

Measures whether the model can translate natural language into correct tool parameters.

### 5. Execution Success Rate

Separates LLM failures from MCP/GitHub implementation failures.

---

# 44. Final Benchmark Principle

The benchmark is not intended to prove that the agent can successfully execute a handful of GitHub commands.

Its purpose is to provide a **repeatable experimental framework** for measuring:

```text
Natural Language
      ↓
Intent Understanding
      ↓
Tool Selection
      ↓
Argument Generation
      ↓
MCP Execution
      ↓
GitHub Result
      ↓
Final Answer
```

The benchmark must therefore remain:

* fixed;
* reproducible;
* auditable;
* resistant to wording overfitting;
* safe to execute;
* independent of production GitHub resources.

The ultimate research comparison is:

> **How much can carefully engineered MCP tool descriptions improve reliable tool selection by small local LLMs?**

---

# 45. Probe Set (not part of the 50)

Some important behaviors have no single correct tool, so they cannot be scored by the A/B/C scheme. They run as a small separate set and are reported on their own line, never mixed into the 50-task metrics.

```json
[
  {
    "id": "P1",
    "request": "What's the status of #3?",
    "why": "Bare number: issue or PR? In this fixture #3 is an issue.",
    "pass": "Calls get_issue(3), or asks which one is meant. Calling get_pull_request(3) is a fail."
  },
  {
    "id": "P2",
    "request": "What's the status of #5?",
    "why": "Bare number: #5 is the baseline PR.",
    "pass": "Calls get_pull_request(5), or asks which one is meant. get_issue(5) is a fail (server returns a validation error naming get_pull_request)."
  },
  {
    "id": "P3",
    "request": "Merge pull request 5.",
    "why": "No merge tool in v1.",
    "pass": "Makes no write call and says merging is not supported."
  },
  {
    "id": "P4",
    "request": "Close pull request 5.",
    "why": "update_issue rejects PR numbers and there is no PR-update tool.",
    "pass": "Makes no tool call that mutates state and says this is not supported."
  },
  {
    "id": "P5",
    "request": "Delete the branch develop.",
    "why": "No delete tool in v1.",
    "pass": "Makes no tool call that mutates state and says deleting is not supported."
  }
]
```

Scoring: each probe is pass/fail by the `pass` rule. Probes P3 to P5 check that the agent declines instead of force-fitting a tool (see `TOOLS_SPEC.md` §0.7 and §6.3). P1 and P2 check the issue-versus-pull-request ambiguity ranked as the top confusion risk in `TOOLS_SPEC.md` §6.1.

---

# 46. Validation Checklist (run before freezing benchmark v1)

A validator script must confirm:

1. exactly 50 tasks; the category counts equal the §17 table;
2. every `expected_tool` exists in `TOOLS_SPEC.md`, and every `expected_args` key is a parameter of that tool;
3. every required parameter of the expected tool is either in `expected_args` or in `presence_only_args`, or has a default;
4. all 16 tools appear as an expected tool at least once;
5. exactly 10 tasks have `held_out = true`, matching §34;
6. every task that creates, edits, or closes anything has `mutates = true`;
7. every referenced fixture item (issue numbers, PR numbers, branches, paths) exists in §19 to §24;
8. the fixture passes `verify.py` before the first task.
