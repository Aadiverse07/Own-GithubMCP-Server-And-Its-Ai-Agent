# TOOLS_SPEC.md — v1 MCP Toolset (16 tools)

**Status:** Day 1 specification (v1.1, after the consistency review; see `DAY1_REVIEW.md`). No implementation code.
**Companion to:** `ARCHITECTURE.md` (envelope, error taxonomy, allowlist, limits).
**Legend:** ⚠ = I am unsure of this PyGithub/GitHub detail; verify against the pinned version before implementing.

---

## 0. Conventions (apply to every tool)

### 0.1 Naming and typing
- Tool names are `verb_noun` snake_case using only `get`, `list`, `create`, `update`, `add`.
- `repo` is always the string `"owner/name"`. `issue_number` and `pr_number` are integers ≥ 1.
- Timestamps are ISO-8601 UTC strings. Users are represented by login string only.
- Example values in this document are illustrative. The authoritative fixture data (issue numbers, PR number 5, branches, files) lives in `EVALUATION_SPEC.md` §19 to §24.

### 0.2 Response envelope

```json
{ "success": true,  "data": { }, "error": null }
{ "success": false, "data": null, "error": { "type": "not_found", "message": "..." } }
```

`error.type` ∈ `auth | not_found | validation | rate_limit | unknown`.

### 0.3 Shared parameters and limits

| Item | Rule |
|------|------|
| `repo` validation | Regex `^[A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+$` **and** must be in `GITHUB_ALLOWED_REPOS`. |
| `limit` (all list tools) | integer, default 10, min 1, max 30. Handler fetches `limit + 1` items to set `truncated`. |
| List output shape | `{ "items": [...], "count": <int>, "truncated": <bool> }` |
| Text truncation | Issue/PR bodies cut at 4,000 chars; comments at 1,000; file content at 20,000. Truncation sets a sibling flag (`body_truncated`, `truncated`). |
| Write-safety | Every write tool is on the repo allowlist and is subject to the confirmation policy in §8. |

### 0.4 Common failures (C1–C6) — apply to every repo-scoped tool, not repeated below

| ID | Condition | Error type |
|----|-----------|-----------|
| C1 | `repo` malformed, or not in the allowlist | `validation` |
| C2 | Repo does not exist or is not visible to the token (404) | `not_found` |
| C3 | Bad/expired/revoked token (401) | `auth` |
| C4 | Token lacks permission (403, no rate-limit signal) | `auth` |
| C5 | Primary or secondary rate limit | `rate_limit` |
| C6 | GitHub 5xx, network error, unexpected exception | `unknown` |

Per-tool tables below list only **additional** failures. A Pydantic input failure (wrong type, out of range, missing required field) is always `validation`.

### 0.5 Tool inventory

| # | Tool | R/W | Core PyGithub call |
|---|------|-----|-------------------|
| 1 | `get_repository` | read | `Github.get_repo` |
| 2 | `list_repositories` | read | `AuthenticatedUser.get_repos` |
| 3 | `list_issues` | read | `Repository.get_issues` |
| 4 | `get_issue` | read | `Repository.get_issue` |
| 5 | `create_issue` | **write** | `Repository.create_issue` |
| 6 | `update_issue` | **write** | `Issue.edit` |
| 7 | `add_issue_comment` | **write** | `Issue.create_comment` |
| 8 | `list_branches` | read | `Repository.get_branches` |
| 9 | `create_branch` | **write** | `Repository.create_git_ref` |
| 10 | `get_file` | read | `Repository.get_contents` |
| 11 | `create_file` | **write** | `Repository.create_file` |
| 12 | `update_file` | **write** | `Repository.update_file` |
| 13 | `list_commits` | read | `Repository.get_commits` |
| 14 | `list_pull_requests` | read | `Repository.get_pulls` |
| 15 | `get_pull_request` | read | `Repository.get_pull` |
| 16 | `create_pull_request` | **write** | `Repository.create_pull` |

9 read tools, 7 write tools.

### 0.6 Design decisions made in this spec (flag if you disagree)

| # | Decision | Reason |
|---|----------|--------|
| D1 | `update_file` does **not** take a blob `sha`; the handler looks it up itself. | PyGithub's `update_file` requires the current sha, but forcing an 8B model to fetch and pass it adds a failure mode unrelated to tool selection. |
| D2 | `get_issue` has an `include_comments` flag (returns the **first** 10 comments; GitHub returns comments oldest first). | No `list_issue_comments` tool exists in v1; otherwise comments are unreadable. |
| D3 | `add_issue_comment` also accepts **pull request numbers** (PR conversation comments are issue comments in GitHub's model). | No `add_pr_comment` tool in v1. Documented in the description so selection stays correct. |
| D4 | `get_issue`, `update_issue` **reject PR numbers** with `validation` and a message naming `get_pull_request`. | GitHub's issues endpoint also returns PRs. Silent acceptance would blur the issue/PR distinction the benchmark tests. |
| D5 | `list_issues` filters out pull requests. | Same reason; the issues API mixes both. |
| D6 | `list_repositories` is limited to the authenticated account (no `owner` parameter). | Avoids user-vs-organization API ambiguity and keeps the schema flat. |
| D7 | Closing/reopening an issue is done via `update_issue(state=...)`; there is no `close_issue`. | Naming rule (only get/list/create/update/add). |
| D8 | `get_file` also lists directories. | No `list_files` tool in v1. |
| D9 | `list_repositories` output is filtered to `GITHUB_ALLOWED_REPOS`. | Keeps the allowlist (`ARCHITECTURE.md` A6) true for every tool, regardless of what the token can see. |

### 0.7 Known capability gaps (useful as negative benchmark tasks)
No tool exists for: merging or closing/editing a pull request, reading PR diffs or files, listing issue comments separately, deleting anything, searching code/issues, managing labels, forks, releases, or reviews. A well-behaved agent should say it cannot do these rather than force-fit a tool.

---

## 1. Repository tools

### 1.1 `get_repository` — READ

**Description (LLM-facing):**
> Returns metadata for ONE repository: description, default branch, visibility, language, and star/fork/open-issue counts. Use when the user asks about a specific repo's details or its default branch. Do NOT use to list repositories (use `list_repositories`) or to read file contents (use `get_file`).

**Parameters**

| Name | Type | Req | Default | Validation | Example |
|------|------|-----|---------|-----------|---------|
| `repo` | string | required | — | `owner/name`, allowlisted | `"testuser/mcp-fixture-repo"` |

**PyGithub:** `Github.get_repo(full_name)`; attributes `full_name`, `description`, `private`, `default_branch`, `language`, `stargazers_count`, `forks_count`, `open_issues_count`, `archived`, `html_url`, `created_at`, `pushed_at`. ⚠ `open_issues_count` includes open PRs (GitHub behavior) — document, don't "fix".

**Output**
```json
{ "full_name": "testuser/mcp-fixture-repo", "description": "Fixture repo", "private": false,
  "default_branch": "main", "language": "Python", "stars": 0, "forks": 0,
  "open_issues_and_prs": 7, "archived": false, "url": "https://github.com/testuser/mcp-fixture-repo",
  "created_at": "2026-09-01T10:00:00Z", "pushed_at": "2026-10-04T09:30:00Z" }
```

**Failures:** C1–C6 only.

**Confusability:** `list_repositories` (singular detail vs. plural browse; "tell me about my repos" is genuinely ambiguous). `list_branches` / `get_file` when the user asks "what's the default branch?" (answer lives here, not in `list_branches`).

---

### 1.2 `list_repositories` — READ

**Description:**
> Lists repositories owned by the authenticated account (name, description, visibility, last update). Use when the user asks which repos exist or wants to browse them. Do NOT use when the user names one specific repo and wants its details (use `get_repository`).

**Parameters**

| Name | Type | Req | Default | Validation | Example |
|------|------|-----|---------|-----------|---------|
| `visibility` | enum | optional | `"all"` | `all \| public \| private` | `"public"` |
| `sort` | enum | optional | `"updated"` | `created \| updated \| pushed \| full_name` | `"pushed"` |
| `limit` | integer | optional | 10 | 1–30 | `5` |

(No `repo` parameter. This is the only tool without one.)

**PyGithub:** `Github.get_user().get_repos(visibility=..., sort=...)` → `PaginatedList`; slice or `itertools.islice` to `limit + 1`. ⚠ The `visibility`/`affiliation` keyword names on `AuthenticatedUser.get_repos` vs. the older `type` keyword need verification (GitHub's API disallows mixing them).

**Output**
```json
{ "items": [ { "full_name": "testuser/mcp-fixture-repo", "description": "Fixture repo",
               "private": false, "updated_at": "2026-10-04T09:30:00Z" } ],
  "count": 1, "truncated": false }
```

**Failures:** C3–C6 only (C1/C2 not applicable). Results are filtered to `GITHUB_ALLOWED_REPOS` (D9). ⚠ How `AuthenticatedUser.get_repos` behaves with a fine-grained token scoped to one repository is unverified; check in the Day 2 smoke test.

**Confusability:** `get_repository` (see above). Low risk with `list_branches` ("list my … ") due to the shared verb.

---

## 2. Issue tools

### 2.1 `list_issues` — READ

**Description:**
> Lists issues in a repository, filtered by state, label, or assignee, newest first. Excludes pull requests. Use when the user wants several issues or asks which/what issues exist. Do NOT use when the user gives one issue number (use `get_issue`) or asks about pull requests (use `list_pull_requests`).

**Parameters**

| Name | Type | Req | Default | Validation | Example |
|------|------|-----|---------|-----------|---------|
| `repo` | string | required | — | see §0.3 | `"testuser/mcp-fixture-repo"` |
| `state` | enum | optional | `"open"` | `open \| closed \| all` | `"closed"` |
| `labels` | array[string] | optional | none | ≤ 10 items, each 1–50 chars | `["bug"]` |
| `assignee` | string | optional | none | GitHub login pattern | `"testuser"` |
| `limit` | integer | optional | 10 | 1–30 | `10` |

**PyGithub:** `repo.get_issues(state=, labels=, assignee=, sort="created", direction="desc")`; each item's `.pull_request` is non-`None` for PRs → skip those while iterating. ⚠ Whether `labels` accepts plain strings vs. `Label` objects varies by version. ⚠ Because PRs are filtered after fetching, a page may contain fewer than `limit` issues; the handler must keep iterating until `limit + 1` issues are collected or the list ends (cap total scanned items, e.g., 100).

**Output**
```json
{ "items": [ { "number": 12, "title": "Login fails", "state": "open", "author": "testuser",
               "labels": ["bug"], "comments": 2, "created_at": "2026-10-01T08:00:00Z",
               "url": "https://github.com/testuser/mcp-fixture-repo/issues/12" } ],
  "count": 1, "truncated": false }
```
(No bodies in list output.)

**Failures (extra):** invalid `state`/`limit` → `validation`; assignee/label that matches nothing → empty list, **not** an error.

**Confusability:** `get_issue` (HIGH — "show issue 12" vs. "show open issues"); `list_pull_requests` (HIGH — "open items", "what's open").

---

### 2.2 `get_issue` — READ

**Description:**
> Returns the full details of ONE issue by number, optionally with its first 10 comments. Use when the user names an issue number or asks to read, show, or check a single issue. Do NOT use to browse or filter many issues (use `list_issues`), to change an issue (use `update_issue`), or for pull requests (use `get_pull_request`).

**Parameters**

| Name | Type | Req | Default | Validation | Example |
|------|------|-----|---------|-----------|---------|
| `repo` | string | required | — | §0.3 | `"testuser/mcp-fixture-repo"` |
| `issue_number` | integer | required | — | ≥ 1 | `12` |
| `include_comments` | boolean | optional | `false` | — | `true` |

**PyGithub:** `repo.get_issue(number)`; when `include_comments` is true, iterate `issue.get_comments()` (oldest first) and stop after 10. ⚠ Fields: `title`, `body`, `state`, `user.login`, `labels`, `assignees`, `comments`, `created_at`, `updated_at`, `closed_at`, `html_url`. The description and this behavior both say "first 10 comments".

**Output**
```json
{ "number": 12, "title": "Login fails", "body": "Steps to reproduce...", "body_truncated": false,
  "state": "open", "author": "testuser", "labels": ["bug"], "assignees": [],
  "comments_count": 2, "created_at": "2026-10-01T08:00:00Z", "updated_at": "2026-10-02T11:00:00Z",
  "closed_at": null, "url": "https://github.com/testuser/mcp-fixture-repo/issues/12",
  "comments": [ { "author": "testuser", "body": "Reproduced.", "created_at": "2026-10-02T11:00:00Z" } ] }
```
(`comments` present only when `include_comments=true`.)

**Failures (extra)**

| Condition | Type |
|-----------|------|
| Issue number does not exist (404) | `not_found` |
| Number belongs to a pull request (D4) | `validation` (message: "#N is a pull request; use get_pull_request") |

**Confusability:** `list_issues` (HIGH), `update_issue` (HIGH — read vs. write on the same object: "check if issue 4 is closed" vs. "close issue 4"), `get_pull_request` (HIGH — bare "#7"), `add_issue_comment` (LOW — "what did people say on issue 4" should be `get_issue` with `include_comments`).

---

### 2.3 `create_issue` — WRITE

**Description:**
> Creates a NEW issue with a title and optional body, labels, and assignees. Use when the user wants to report, file, or open a new issue. Do NOT use to change an existing issue (use `update_issue`) or to reply on an existing issue (use `add_issue_comment`).

**Parameters**

| Name | Type | Req | Default | Validation | Example |
|------|------|-----|---------|-----------|---------|
| `repo` | string | required | — | §0.3 | `"testuser/mcp-fixture-repo"` |
| `title` | string | required | — | 1–256 chars, not blank (⚠ GitHub's exact limit unverified) | `"Add dark mode"` |
| `body` | string | optional | `""` | ≤ 65,536 chars (⚠ verify) | `"Users want a dark theme."` |
| `labels` | array[string] | optional | none | ≤ 10 items, 1–50 chars each | `["enhancement"]` |
| `assignees` | array[string] | optional | none | ≤ 10 logins | `["testuser"]` |

**PyGithub:** `repo.create_issue(title=, body=, labels=, assignees=)`. Omit unset arguments rather than passing `None` (PyGithub uses a `NotSet` sentinel). ⚠ Behavior when a label does not exist (GitHub may auto-create it for users with push access) and when an assignee is not a collaborator (may be silently dropped or rejected) — verify with a live test and document the result.

**Output**
```json
{ "number": 13, "title": "Add dark mode", "state": "open", "labels": ["enhancement"],
  "assignees": [], "url": "https://github.com/testuser/mcp-fixture-repo/issues/13",
  "created_at": "2026-10-05T07:00:00Z" }
```

**Failures (extra)**

| Condition | Type |
|-----------|------|
| Invalid assignee/label rejected by GitHub (422) | `validation` |
| Issues disabled on the repo (⚠ likely 410 Gone) | `unknown` (per taxonomy: not 401/403/404/422) |

**Confusability:** `update_issue` (MED — "edit/create"), `add_issue_comment` (MED — "add a note about X" with no issue number reads like a new issue), `create_pull_request` / `create_file` (LOW, `create_` prefix).

---

### 2.4 `update_issue` — WRITE

**Description:**
> Edits an EXISTING issue: change its title, body, labels, assignees, or open/closed state. Use to rename, edit, relabel, assign, close, or reopen an issue. Do NOT use to post a reply (use `add_issue_comment`), to create a new issue (use `create_issue`), or on pull requests.

**Parameters**

| Name | Type | Req | Default | Validation | Example |
|------|------|-----|---------|-----------|---------|
| `repo` | string | required | — | §0.3 | `"testuser/mcp-fixture-repo"` |
| `issue_number` | integer | required | — | ≥ 1 | `12` |
| `title` | string | optional | unchanged | 1–256 chars | `"Login fails on Safari"` |
| `body` | string | optional | unchanged | ≤ 65,536 chars | `"Updated steps..."` |
| `state` | enum | optional | unchanged | `open \| closed` | `"closed"` |
| `labels` | array[string] | optional | unchanged | ≤ 10 items; **replaces the full label set** | `["bug","wontfix"]` |
| `assignees` | array[string] | optional | unchanged | ≤ 10 logins; **replaces** the assignee set | `[]` |

Rule: at least one of `title`, `body`, `state`, `labels`, `assignees` must be supplied, else `validation`.

**PyGithub:** `repo.get_issue(n)` then `issue.edit(title=, body=, state=, labels=, assignees=)`; only pass fields the caller supplied. ⚠ `state_reason` (completed / not_planned) exists in recent PyGithub but is **not exposed in v1**.

**Output**
```json
{ "number": 12, "title": "Login fails", "state": "closed", "labels": ["bug"], "assignees": [],
  "changed_fields": ["state"], "url": "https://github.com/testuser/mcp-fixture-repo/issues/12",
  "updated_at": "2026-10-05T07:10:00Z" }
```
(`changed_fields` is computed by the handler by comparing before/after; not a GitHub field.)

**Failures (extra)**

| Condition | Type |
|-----------|------|
| No updatable field supplied | `validation` |
| Issue number does not exist | `not_found` |
| Number is a pull request (D4) | `validation` |
| Invalid label/assignee rejected (422) | `validation` |

**Confusability:** `get_issue` (HIGH), `add_issue_comment` (MED — "add a note to issue 4" could be a body edit; "comment and close" is two steps), `create_issue` (MED), `get_pull_request` / PR-edit requests (LOW — no PR-update tool exists; an agent may wrongly try `update_issue` for "close PR 5"), `update_file` (LOW, shared `update_` prefix).

---

### 2.5 `add_issue_comment` — WRITE

**Description:**
> Posts a new comment on an existing issue, or on a pull request's conversation. Use when the user wants to reply, comment, or leave a note on an issue or PR. Do NOT use to edit or close the issue itself (use `update_issue`) or to create a new issue (use `create_issue`).

**Parameters**

| Name | Type | Req | Default | Validation | Example |
|------|------|-----|---------|-----------|---------|
| `repo` | string | required | — | §0.3 | `"testuser/mcp-fixture-repo"` |
| `issue_number` | integer | required | — | ≥ 1 (issue **or** PR number, D3) | `12` |
| `body` | string | required | — | 1–65,536 chars, not blank | `"Fixed in the latest commit."` |

**PyGithub:** `repo.get_issue(n)` then `issue.create_comment(body)`. ⚠ Confirm `get_issue` works for PR numbers (GitHub's issue endpoint does return PRs, so it should).

**Output**
```json
{ "comment_id": 4455667788, "issue_number": 12,
  "url": "https://github.com/testuser/mcp-fixture-repo/issues/12#issuecomment-4455667788",
  "created_at": "2026-10-05T07:15:00Z" }
```

**Failures (extra):** number does not exist → `not_found`; issue locked (⚠ likely 403 or 422 depending on permissions) → `auth` or `validation` respectively (verify and fix mapping on live test); blank body → `validation`.

**Confusability:** `update_issue` (MED), `create_issue` (MED), `get_issue` (LOW — "what comments are on issue 4").

---

## 3. Branch tools

### 3.1 `list_branches` — READ

**Description:**
> Lists the branch names in a repository and whether each is protected. Use when the user asks which branches exist. Do NOT use to create a branch (use `create_branch`) or to see commit history (use `list_commits`).

**Parameters**

| Name | Type | Req | Default | Validation | Example |
|------|------|-----|---------|-----------|---------|
| `repo` | string | required | — | §0.3 | `"testuser/mcp-fixture-repo"` |
| `limit` | integer | optional | 10 | 1–30 | `20` |

**PyGithub:** `repo.get_branches()` → `PaginatedList[Branch]`; `branch.name`, `branch.protected`, `branch.commit.sha`.

**Output**
```json
{ "items": [ { "name": "main", "protected": false, "head_sha": "a1b2c3d" },
             { "name": "feature/readme", "protected": false, "head_sha": "d4e5f6a" } ],
  "count": 2, "truncated": false }
```

**Failures (extra):** empty repository (⚠ GitHub may return 404 or an empty list) → `not_found` or empty result; verify.

**Confusability:** `create_branch` (MED — shared noun), `list_commits` (MED — "what's on main", "latest on branch X"), `get_repository` (LOW — "default branch").

---

### 3.2 `create_branch` — WRITE

**Description:**
> Creates a new branch from an existing branch (default: the repo's default branch). Use when the user wants to make, start, or cut a new branch. Do NOT use to list branches (use `list_branches`) or to add or change files (use `create_file` or `update_file`).

**Parameters**

| Name | Type | Req | Default | Validation | Example |
|------|------|-----|---------|-----------|---------|
| `repo` | string | required | — | §0.3 | `"testuser/mcp-fixture-repo"` |
| `branch_name` | string | required | — | 1–255 chars; regex `^[A-Za-z0-9][A-Za-z0-9._/-]*$`; no `..`, no trailing `/` or `.lock` | `"feature/dark-mode"` |
| `from_branch` | string | optional | repo default branch | same rules as `branch_name` | `"main"` |

**PyGithub:** source SHA from `repo.get_branch(from_branch).commit.sha`; then `repo.create_git_ref(ref="refs/heads/<branch_name>", sha=<sha>)`. ⚠ The validation regex is a conservative subset of git's ref rules, not a full implementation.

**Output**
```json
{ "branch": "feature/dark-mode", "from_branch": "main", "sha": "a1b2c3d4e5f6",
  "url": "https://github.com/testuser/mcp-fixture-repo/tree/feature/dark-mode" }
```

**Failures (extra)**

| Condition | Type |
|-----------|------|
| `from_branch` does not exist | `not_found` |
| Branch name already exists (422 "Reference already exists") | `validation` |
| Invalid branch name | `validation` |
| Empty repo with no commits (⚠ likely 404/409) | `not_found` / `unknown`; verify |

**Confusability:** `list_branches` (MED), `create_file` (MED — `create_file` also has a `branch` parameter, so "create a branch with a file" pulls the model toward `create_file`), `create_pull_request` (MED — branch-then-PR workflows).

---

## 4. File tools

### 4.1 `get_file` — READ

**Description:**
> Reads the text content of ONE file at a path on a given branch, or lists the entries if the path is a directory. Use to read, show, open, or browse files. Do NOT use to change a file (use `update_file` or `create_file`) or to see who changed it (use `list_commits`).

**Parameters**

| Name | Type | Req | Default | Validation | Example |
|------|------|-----|---------|-----------|---------|
| `repo` | string | required | — | §0.3 | `"testuser/mcp-fixture-repo"` |
| `path` | string | required | — | 1–1000 chars; no leading `/`, no `..`; `""`-root listing expressed as `"."` | `"README.md"` |
| `branch` | string | optional | repo default branch | branch-name rules | `"main"` |

**PyGithub:** `repo.get_contents(path, ref=branch)` → a `ContentFile` for a file or a `list[ContentFile]` for a directory. File text: `content_file.decoded_content.decode("utf-8")`. ⚠ Files over ~1 MB return no inline content via this endpoint (blob API needed) — v1 returns a `validation` error "file too large" instead. ⚠ Binary files fail UTF-8 decoding → `validation` "binary file not supported".

**Output (file)**
```json
{ "type": "file", "path": "README.md", "branch": "main", "size": 120, "sha": "e69de29",
  "content": "# Demo repo\n...", "truncated": false }
```
**Output (directory)**
```json
{ "type": "dir", "path": "src", "branch": "main",
  "entries": [ { "name": "app.py", "type": "file", "size": 840 }, { "name": "utils", "type": "dir", "size": 0 } ],
  "truncated": false }
```

**Failures (extra)**

| Condition | Type |
|-----------|------|
| Path or branch does not exist (404) | `not_found` |
| Binary file / file over size cap | `validation` |
| Path is a submodule or symlink (⚠ PyGithub behavior unverified) | `unknown` |

**Confusability:** `update_file` (MED — "edit README" begins with a read), `list_commits` (MED — "history of README.md" has a `path` filter there), `get_repository` (LOW — "describe the repo").

---

### 4.2 `create_file` — WRITE

**Description:**
> Creates a NEW file with the given content and commits it to a branch. Fails if the path already exists. Use when the user wants to add a file that does not exist yet. Do NOT use to change an existing file (use `update_file`) or to create a branch (use `create_branch`).

**Parameters**

| Name | Type | Req | Default | Validation | Example |
|------|------|-----|---------|-----------|---------|
| `repo` | string | required | — | §0.3 | `"testuser/mcp-fixture-repo"` |
| `path` | string | required | — | as `get_file` | `"docs/notes.md"` |
| `content` | string | required | — | UTF-8 text, 0–100,000 chars (our cap) | `"# Notes\n"` |
| `commit_message` | string | required | — | 1–500 chars | `"Add notes file"` |
| `branch` | string | optional | repo default branch | branch-name rules; must exist | `"feature/dark-mode"` |

**PyGithub:** `repo.create_file(path, message, content, branch=)` → returns a dict with `"content"` (ContentFile) and `"commit"` (Commit). ⚠ Confirm that `content` is passed as plain `str` (PyGithub encodes it) and the exact return keys.

**Output**
```json
{ "path": "docs/notes.md", "branch": "main", "commit_sha": "9f8e7d6", "file_sha": "c0ffee1",
  "url": "https://github.com/testuser/mcp-fixture-repo/blob/main/docs/notes.md" }
```

**Failures (extra)**

| Condition | Type |
|-----------|------|
| Path already exists (422, "sha wasn't supplied") | `validation` (message: "File exists; use update_file") |
| Branch does not exist | `not_found` |
| Branch protected / push blocked (403) | `auth` |
| Invalid path | `validation` |

**Confusability:** `update_file` (HIGH — "add a line to README" vs. "add a README"), `create_branch` (MED — `branch` parameter lure, `create_` prefix), `create_issue` (LOW — "create a TODO" ambiguous between file and issue).

---

### 4.3 `update_file` — WRITE

**Description:**
> Replaces the ENTIRE content of an EXISTING file and commits it. Fails if the file does not exist. Use when the user wants to edit, modify, or overwrite a file. Do NOT use for a new file (use `create_file`). To change part of a file, read it first with `get_file` and send the full new content.

**Parameters**

| Name | Type | Req | Default | Validation | Example |
|------|------|-----|---------|-----------|---------|
| `repo` | string | required | — | §0.3 | `"testuser/mcp-fixture-repo"` |
| `path` | string | required | — | as `get_file` | `"README.md"` |
| `content` | string | required | — | UTF-8 text, 0–100,000 chars; **full replacement** | `"# Demo repo v2\n"` |
| `commit_message` | string | required | — | 1–500 chars | `"Update README title"` |
| `branch` | string | optional | repo default branch | must exist | `"main"` |

**PyGithub:** handler first calls `repo.get_contents(path, ref=branch)` to obtain `.sha` (D1), then `repo.update_file(path, message, content, sha, branch=)`. ⚠ Same return-shape caveat as `create_file`. ⚠ A race between the sha lookup and the write yields a 409 conflict → `unknown` per the taxonomy; acceptable in a single-user setup.

**Output**
```json
{ "path": "README.md", "branch": "main", "commit_sha": "1a2b3c4", "file_sha": "f00ba47",
  "url": "https://github.com/testuser/mcp-fixture-repo/blob/main/README.md" }
```

**Failures (extra)**

| Condition | Type |
|-----------|------|
| File does not exist | `not_found` (message: "File not found; use create_file") |
| Path is a directory | `validation` |
| Branch does not exist | `not_found` |
| Push blocked by protection (403) | `auth` |
| 409 conflict | `unknown` |

**Confusability:** `create_file` (HIGH), `get_file` (MED — a small model may call `update_file` blindly with partial content and destroy the file; mitigated by the description and the confirmation policy), `update_issue` (LOW — shared prefix).

---

## 5. Commit and pull request tools

### 5.1 `list_commits` — READ

**Description:**
> Lists recent commits (SHA, message, author, date) on a branch, optionally only those that touched one file path. Use when the user asks for history, recent changes, or who changed something. Do NOT use to read file content (use `get_file`) or to list branches (use `list_branches`).

**Parameters**

| Name | Type | Req | Default | Validation | Example |
|------|------|-----|---------|-----------|---------|
| `repo` | string | required | — | §0.3 | `"testuser/mcp-fixture-repo"` |
| `branch` | string | optional | repo default branch | branch-name rules | `"main"` |
| `path` | string | optional | none | as `get_file` | `"README.md"` |
| `author` | string | optional | none | GitHub login | `"testuser"` |
| `limit` | integer | optional | 10 | 1–30 | `5` |

**PyGithub:** `repo.get_commits(sha=<branch>, path=, author=)` → `PaginatedList[Commit]`. Fields: `commit.sha`, `commit.commit.message`, `commit.commit.author.name`, `commit.commit.author.date`, `commit.html_url`. ⚠ `commit.author` (linked GitHub user) can be `None` for unlinked emails — use the git author name instead.

**Output**
```json
{ "items": [ { "sha": "1a2b3c4", "message": "Update README title", "author": "Fixture User",
               "date": "2026-10-05T07:20:00Z",
               "url": "https://github.com/testuser/mcp-fixture-repo/commit/1a2b3c4" } ],
  "count": 1, "truncated": false }
```
Only the first line of the commit message is returned.

**Failures (extra)**

| Condition | Type |
|-----------|------|
| Branch does not exist | `not_found` (⚠ could surface as 404 or 422; verify) |
| Empty repository (⚠ GitHub returns 409 "Git Repository is empty") | `unknown` per taxonomy |

**Confusability:** `list_branches` (MED), `get_file` (MED — "what's in README" vs. "what changed in README"), `list_pull_requests` (LOW — "recent changes").

---

### 5.2 `list_pull_requests` — READ

**Description:**
> Lists pull requests in a repository, filtered by state or by base/head branch, newest first. Use when the user wants several PRs or asks which PRs are open. Do NOT use when given one PR number (use `get_pull_request`) or for issues (use `list_issues`).

**Parameters**

| Name | Type | Req | Default | Validation | Example |
|------|------|-----|---------|-----------|---------|
| `repo` | string | required | — | §0.3 | `"testuser/mcp-fixture-repo"` |
| `state` | enum | optional | `"open"` | `open \| closed \| all` | `"all"` |
| `base` | string | optional | none | branch-name rules | `"main"` |
| `head` | string | optional | none | branch name; handler converts to `"owner:branch"` | `"feature/dark-mode"` |
| `limit` | integer | optional | 10 | 1–30 | `10` |

**PyGithub:** `repo.get_pulls(state=, sort="created", direction="desc", base=, head=)`. ⚠ GitHub's `head` filter requires `user:branch` format; the handler prefixes the repo owner when no colon is present (same-repo PRs only in v1). ⚠ `PullRequest.merged` and the change-count fields may trigger an extra API call per item when accessed on list results — use `merged_at` (present in list payloads) and omit counts.

**Output**
```json
{ "items": [ { "number": 7, "title": "Add dark mode", "state": "open", "draft": false,
               "author": "testuser", "head": "feature/dark-mode", "base": "main",
               "merged": false, "created_at": "2026-10-04T12:00:00Z",
               "url": "https://github.com/testuser/mcp-fixture-repo/pull/7" } ],
  "count": 1, "truncated": false }
```
(`merged` derived from `merged_at is not None`.)

**Failures (extra):** invalid `state` → `validation`; branch filter matching nothing → empty list, not an error.

**Confusability:** `get_pull_request` (HIGH), `list_issues` (HIGH — "open items"), `list_commits` (LOW).

---

### 5.3 `get_pull_request` — READ

**Description:**
> Returns the details of ONE pull request by number: title, description, state, branches, merge status, and change counts. Use when the user names a PR number or asks about one specific PR. Do NOT use to browse many PRs (use `list_pull_requests`) or for issues (use `get_issue`).

**Parameters**

| Name | Type | Req | Default | Validation | Example |
|------|------|-----|---------|-----------|---------|
| `repo` | string | required | — | §0.3 | `"testuser/mcp-fixture-repo"` |
| `pr_number` | integer | required | — | ≥ 1 | `7` |

**PyGithub:** `repo.get_pull(number)`; fields `title`, `body`, `state`, `draft`, `merged`, `mergeable`, `user.login`, `head.ref`, `base.ref`, `commits`, `additions`, `deletions`, `changed_files`, `created_at`, `updated_at`, `merged_at`, `html_url`. ⚠ `mergeable` may be `None` while GitHub computes it; return `null` and do not retry.

**Output**
```json
{ "number": 7, "title": "Add dark mode", "body": "Implements theme toggle.", "body_truncated": false,
  "state": "open", "draft": false, "merged": false, "mergeable": true,
  "author": "testuser", "head": "feature/dark-mode", "base": "main",
  "commits": 3, "additions": 40, "deletions": 5, "changed_files": 2,
  "created_at": "2026-10-04T12:00:00Z", "updated_at": "2026-10-05T06:00:00Z", "merged_at": null,
  "url": "https://github.com/testuser/mcp-fixture-repo/pull/7" }
```

**Failures (extra):** PR number does not exist (404) → `not_found`. If the number is actually an issue, GitHub's pulls endpoint returns 404 → `not_found` (message should say "not found as a pull request; it may be an issue — try get_issue").

**Confusability:** `get_issue` (HIGH — bare "#7"; this is the core "issue vs. PR number" trap), `list_pull_requests` (HIGH), `create_pull_request` (MED — "the PR for branch X" could be lookup or creation).

---

### 5.4 `create_pull_request` — WRITE

**Description:**
> Opens a new pull request that proposes merging a head branch into a base branch. Use when the user wants to open or create a PR. Do NOT use to create the branch itself (use `create_branch`) or to merge a PR (not supported).

**Parameters**

| Name | Type | Req | Default | Validation | Example |
|------|------|-----|---------|-----------|---------|
| `repo` | string | required | — | §0.3 | `"testuser/mcp-fixture-repo"` |
| `title` | string | required | — | 1–256 chars | `"Add dark mode"` |
| `head` | string | required | — | existing branch name (same repo in v1) | `"feature/dark-mode"` |
| `base` | string | optional | repo default branch | existing branch name; must differ from `head` | `"main"` |
| `body` | string | optional | `""` | ≤ 65,536 chars | `"Implements theme toggle."` |
| `draft` | boolean | optional | `false` | — | `false` |

**PyGithub:** `repo.create_pull(title=, body=, head=, base=, draft=)`. Use keyword arguments only. ⚠ Older PyGithub versions accepted a deprecated positional form `create_pull(base, head, title, body)`; verify signature and the `draft` keyword for the pinned version.

**Output**
```json
{ "number": 8, "title": "Add dark mode", "state": "open", "draft": false,
  "head": "feature/dark-mode", "base": "main",
  "url": "https://github.com/testuser/mcp-fixture-repo/pull/8", "created_at": "2026-10-05T07:30:00Z" }
```

**Failures (extra)**

| Condition | Type |
|-----------|------|
| `head` branch does not exist (⚠ GitHub typically 422 "invalid head") | `validation` |
| `base` branch does not exist (⚠ 422 or 404) | `validation` / `not_found`; verify |
| No commits between head and base (422) | `validation` |
| PR already exists for this head/base (422) | `validation` |
| `head == base` | `validation` |

**Confusability:** `create_branch` (MED), `get_pull_request` (MED), `create_issue` (LOW — "open a request/ticket").

---

## 6. Confusability Matrix

Rows and columns are tools. **H** = high risk, **M** = medium, **L** = low, `·` = negligible. The matrix is symmetric: a pair is risky in both directions, though the *dominant* error direction is given in §6.2.

Column codes: **GR** get_repository · **LR** list_repositories · **LI** list_issues · **GI** get_issue · **CI** create_issue · **UI** update_issue · **AC** add_issue_comment · **LB** list_branches · **CB** create_branch · **GF** get_file · **CF** create_file · **UF** update_file · **LC** list_commits · **LP** list_pull_requests · **GP** get_pull_request · **CP** create_pull_request

| | GR | LR | LI | GI | CI | UI | AC | LB | CB | GF | CF | UF | LC | LP | GP | CP |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **GR** | — | M | · | · | · | · | · | L | · | · | · | · | · | · | · | · |
| **LR** | M | — | · | · | · | · | · | · | · | · | · | · | · | · | · | · |
| **LI** | · | · | — | **H** | · | · | · | · | · | · | · | · | · | **H** | · | · |
| **GI** | · | · | **H** | — | · | **H** | L | · | · | · | · | · | · | · | **H** | · |
| **CI** | · | · | · | · | — | M | M | · | · | · | L | · | · | · | · | L |
| **UI** | · | · | · | **H** | M | — | M | · | · | · | · | L | · | · | L | · |
| **AC** | · | · | · | L | M | M | — | · | · | · | · | · | · | · | · | · |
| **LB** | L | · | · | · | · | · | · | — | M | · | · | · | M | · | · | · |
| **CB** | · | · | · | · | · | · | · | M | — | · | M | · | · | · | · | M |
| **GF** | · | · | · | · | · | · | · | · | · | — | · | M | M | · | · | · |
| **CF** | · | · | · | · | L | · | · | · | M | · | — | **H** | · | · | · | · |
| **UF** | · | · | · | · | · | L | · | · | · | M | **H** | — | · | · | · | · |
| **LC** | · | · | · | · | · | · | · | M | · | M | · | · | — | L | · | · |
| **LP** | · | · | **H** | · | · | · | · | · | · | · | · | · | L | — | **H** | · |
| **GP** | · | · | · | **H** | · | L | · | · | · | · | · | · | · | **H** | — | M |
| **CP** | · | · | · | · | L | · | · | · | M | · | · | · | · | · | M | — |

### 6.1 Riskiest pairs and what drives them

| Rank | Pair | Why a small model confuses them | Mitigation (see §7) |
|------|------|--------------------------------|--------------------|
| 1 | `get_issue` ↔ `get_pull_request` | Bare references like "#7", "item 7"; GitHub itself blurs issues and PRs. | Descriptions name each other; system prompt rule: "if the user says 'issue' use issue tools, 'PR/pull request' use PR tools; if unclear, ask." Bare-`#N` requests are covered by the probe set (`EVALUATION_SPEC.md` §45), because no single tool is unambiguously correct for them. |
| 2 | `get_issue` ↔ `list_issues` (and `get_pull_request` ↔ `list_pull_requests`) | Singular vs. plural; "show issue 12" vs. "show issues". | Lead each description with "ONE" / "several"; require number in the `get_` description's trigger. |
| 3 | `get_issue` ↔ `update_issue` | Same object, read vs. write; "check if issue 4 is closed" vs. "close issue 4". | Mark read-only/edits explicitly in the first clause; list trigger verbs (close, reopen, rename, assign). |
| 4 | `create_file` ↔ `update_file` | "Add X to README" is update if README exists, create if not; model usually can't know. | Descriptions state the failure mode of each ("Fails if the path already exists/does not exist") and error messages cross-reference the sibling. Benchmark both existing/non-existing path variants. |
| 5 | `list_issues` ↔ `list_pull_requests` | "Open items", "what's open", "anything pending". | Each description names the sibling; D5 filters PRs out of `list_issues`. |
| 6 | `add_issue_comment` ↔ `update_issue` / `create_issue` | "Add a note", "leave feedback", "mention X on issue 4". | Trigger words: comment/reply/note → `add_issue_comment`; edit/close/relabel → `update_issue`. |
| 7 | `create_branch` ↔ `create_file` ↔ `create_pull_request` | Shared `create_` prefix and a `branch` parameter on `create_file`; branch → file → PR workflow chains. | Keep `branch` parameter descriptions explicit ("existing branch to commit to; does not create it"). Treat the **first** call in multi-step tasks as the scored one. |
| 8 | `get_file` ↔ `list_commits` | "What changed in README" vs. "show README". | Descriptions mention "content" vs. "history". |

### 6.2 Dominant error direction (hypotheses to test)
- Model reads → writes: `get_issue` request answered with `update_issue` when the verb "check/mark" is ambiguous.
- Singular → plural: `get_*` prompts answered with `list_*` when the user omits a number.
- Write → write: `update_file` when the file does not exist (or `create_file` when it does) — this is a *selection* error that the server turns into a recoverable `validation` error.

### 6.3 Out-of-toolset requests that attract wrong tools
"Close PR 5" → `update_issue` · "Delete branch X" → `create_branch`/`list_branches` · "Merge PR 3" → `create_pull_request`/`get_pull_request` · "Search issues for 'login'" → `list_issues`. These should be scored as correct only if the agent declines or explains the limitation. They run as the probe set in `EVALUATION_SPEC.md` §45, outside the 50 scored tasks.

---

## 7. Five Rules for Descriptions That Small (7–8B) Models Can Use

1. **Lead with one plain verb-plus-object sentence in the user's own words.** The first ~12 words decide selection for small models. Use the same nouns as the tool name ("issue", "pull request", "file") and make the *first words* differ between siblings (e.g., "Lists…" vs. "Returns…ONE…"); avoid shared boilerplate openings.
2. **Put the contrast inside the description, with exact sibling names.** "Do NOT use … (use `list_issues`)" beats abstract wording, because the model can copy the name. Include it only for the 1–3 genuinely confusable siblings from §6; a long list of exclusions dilutes the signal.
3. **State cardinality and mutation in capitals or fixed phrases.** "ONE", "NEW", "EXISTING", "ENTIRE", "Excludes pull requests", "Fails if the path already exists" — these small, consistent markers distinguish get/list and create/update. Say explicitly whether the tool writes anything.
4. **Include the trigger words users actually say.** Mention synonyms in the "Use when" clause (close, reopen, rename → `update_issue`; reply, comment, note → `add_issue_comment`; read, show, open → `get_file`). Small models match surface tokens more than concepts.
5. **Keep descriptions short and parameters flat and self-describing.** Target ≤ 60 words per description; put formats and an example in each parameter description (`repo`: `"owner/name", e.g. "octocat/hello"`), use enums instead of free text, keep optional parameters to a minimum, and state defaults. Treat every wording change as a **versioned experiment** (`description_version`) so the benchmark can attribute accuracy changes to it.

---

## 8. Confirmation Policy (interactive mode)

All 9 read tools run without confirmation. The 7 write tools are tiered by **reversibility within the v1 toolset** (which contains no delete tools), **visibility to others** (notifications, public comments), and **blast radius**.

| Tier | Tools | Require confirmation? | Why |
|------|-------|----------------------|-----|
| **1 — High** | `update_file`, `create_file`, `update_issue` | **Always** | `update_file` overwrites the whole file from model-generated content, and a truncated or hallucinated body would silently destroy it (recoverable only via git history). `create_file` commits to a branch — possibly the default branch — and cannot be undone by any v1 tool. `update_issue` can close issues and **replaces** whole label/assignee sets. |
| **2 — Medium** | `create_pull_request`, `add_issue_comment`, `create_issue` | **Yes** (default) | Each is visible to others and can trigger notifications; none can be deleted or edited by the toolset. Low technical risk but irreversible within v1. |
| **3 — Low** | `create_branch` | **Optional** (recommend yes in interactive mode, auto-approve in benchmark mode) | Creates only a pointer, harmless and easy to remove manually, but cannot be removed by any v1 tool, so it can accumulate clutter. |

**Confirmation prompt contents:** tool name, resolved repo, **resolved branch** (including the defaulted one), and an argument preview — for file writes, path, commit message, content length, and the first ~20 lines (ideally a diff against current content for `update_file`).

**Benchmark mode:** auto-approve all writes, relying on the allowlist, the throwaway account, and a fixture reset between tasks/runs (per `ARCHITECTURE.md` §7). A `--dry-run` flag stubs writes so selection can be measured with no side effects.

---

## 9. Decisions and Open Questions

### Resolved in review

| # | Question | Decision |
|---|----------|----------|
| 1 | Allowlist and `list_repositories` | Filter output to `GITHUB_ALLOWED_REPOS` (D9). |
| 3 | `get_issue` comments: first or latest | First 10 (D2). The description already says "first". |
| 5 | Unsupported-request tasks | Run as a separate probe set (`EVALUATION_SPEC.md` §45), not inside the 50. |
| 6 | Multi-step scoring for "update the README to say X" | If the user supplies the full new content, the correct call is `update_file` directly (single-step). Read-first is expected only when the request explicitly says to read first (task T41). |

### Still open

2. **D3:** is `add_issue_comment` accepting PR numbers acceptable? (No benchmark task depends on it, so this can wait.)
4. **Default branch writes:** should `create_file` / `update_file` default to the repo's default branch (current spec), or require an explicit `branch`? Current spec keeps the default to avoid an extra required argument for a small model.
7. **Live verification list:** a short Day 2 live smoke test of every ⚠ item against the throwaway account before the schemas are frozen.
