---
name: pr-code-review
description: Code review a GitHub pull request
disable-model-invocation: true
allowed-tools:
  - Bash(gh issue view:*)
  - Bash(gh search:*)
  - Bash(gh issue list:*)
  - Bash(gh pr comment:*)
  - Bash(gh pr diff:*)
  - Bash(gh pr view:*)
  - Bash(gh pr list:*)
  - Bash(gh api:*)
  - Bash(gh pr checks:*)
  - Bash(inspect pr:*)
  - Bash(git log:*)
  - Bash(git blame:*)
  - Bash(git diff:*)
  - Bash(git rev-parse:*)
  - Bash(git show:*)
---

Provide a code review for the given pull request.

When invoked, always run — draft PRs and re-reviews of PRs that already have a prior review from this skill are both expected and encouraged. The only disqualifier is a PR that is closed or merged.

Follow these steps precisely:

Model policy: wherever a step says "Haiku agent", use the **latest Haiku** model; wherever a step says "Sonnet agent", use the **latest Sonnet** model; wherever a step says "Fable agent" or "Astra agent", use the **latest Fable** or **latest Astra** model respectively.

Parallelism: when workflow tooling (e.g. a `workflow` tool with `parallel()`) is available, prefer it for the parallel fan-out steps (the review agents in step 5, the per-finding confidence scoring in step 7, and the Fable/Astra verification in step 9) instead of dispatching individual Agent calls. Otherwise, launch the agents as multiple Agent calls in a single message so they run concurrently.

Output routing: findings are posted as PR comments by default (steps 11–14). If the user explicitly asked for a different output when starting the review (e.g. "report back to me, don't comment on the PR"), honor that instead and skip the posting steps.

1. Use a Haiku agent to check whether the PR is closed or merged. If so, stop. Do NOT skip on draft status or on the presence of a prior review.

2. Use another Haiku agent to list the file paths (not contents) of relevant agent instruction files: root `AGENTS.md` and/or `CLAUDE.md` files (if present), plus any nested `AGENTS.md` or `CLAUDE.md` files in directories touched by the PR.

3. Use a Haiku agent to fetch the PR and return: a summary of the change, the head sha, the repo owner, and the repo name. Then, alongside the initial diff retrieval, run hotspot triage: use a Haiku agent to run the `inspect` CLI against the remote PR:
   ```bash
   inspect pr --remote <owner>/<repo> --format json --context <number>
   ```
   Parse the entity-level results into a **hotspot summary**: for each entity, its name, file, line range, risk level/score, and flags (blast radius, dependency/dependent impact, public API, cross-file). If `inspect` is unavailable, unauthenticated, fails, or reports no hotspots, record the hotspot summary as unavailable/clean and continue — this is not a code finding.

4. **Fetch prior findings from earlier runs of this skill.** Use `gh api user --jq .login` to get the authenticated user, then list issue comments and PR review comments on this PR authored by that user. Identify prior skill output by these HTML markers embedded in each comment body:
   - Summary comment: `<!-- pr-code-review:summary -->`
   - Finding comment: `<!-- pr-code-review:finding:{category}:{file}:{line_start}-{line_end} -->`
   Build a map `prior_findings[category][file][line_range]` → `{comment_id, comment_type (review|issue), thread_last_author}` for later matching.

5. Launch 11 parallel Sonnet agents to independently review the change. Include the step 3 hotspot summary in the brief of every agent whose review examines code behavior (#2 `bug`, #3 `history`, #6 `hotspot`, #7 `removed-behavior`, #8 `call-site`, #9 `efficiency`, #10 `altitude`) so they prioritize high-risk entities first. Hotspots guide attention only — they are not findings by themselves, and agents must still review the rest of the diff. Each agent tags every finding it returns with a fixed category:
   a. Agent #1 (`claudemd`): Audit compliance with the relevant `AGENTS.md` and/or `CLAUDE.md` files. Remember these files guide agents writing code, so not every rule applies during review. Keep the `claudemd` category name for compatibility with findings from earlier skill versions.
   b. Agent #2 (`bug`): Shallow scan of the diff for obvious bugs — inverted conditions, off-by-one errors, null/nil dereferences, wrong variable used, swallowed errors. Focus on large bugs, ignore nitpicks and likely false positives.
   c. Agent #3 (`history`): Read git blame and history of the modified code and flag bugs visible only with that historical context.
   d. Agent #4 (`prior-pr`): Read prior PRs touching these files, surface comments that still apply.
   e. Agent #5 (`code-comment`): Read code comments in the modified files and flag changes that violate comment-stated guidance.
   f. Agent #6 (`hotspot`): Work from the step 3 hotspot summary (re-run the `inspect` CLI from step 3 only if that summary lacks the detail needed). Treat `inspect` as a hotspot triage tool, not as proof of a finding. Parse its entity-level results and prioritize Critical/High-risk entities, plus Medium-risk entities with meaningful blast radius, dependency/dependent impact, public API, or cross-file flags. Check each selected hotspot meticulously by reading the changed entity, its surrounding implementation, dependencies, dependents/callers, and relevant history. Report only actionable issues caused or exposed by this PR; pre-existing risk without a PR regression is not a finding. Map findings to a concrete line range in the modified diff whenever possible, rather than merely citing the entity's full-file range. Include the `inspect` risk level/score and relevant entity or dependency evidence in the reason. If `inspect` is unavailable, unauthenticated, fails, or reports no hotspots, report that pass as unavailable or clean and continue without treating the condition as a code finding.
   g. Agent #7 (`removed-behavior`): For every deleted line or block in the diff, name the invariant or contract it enforced, then find where (or prove) that contract is re-established elsewhere. Report only gaps where the invariant is NOT re-established.
   h. Agent #8 (`call-site`): For each function/method whose signature or behavior changed in the diff, grep the codebase for callers, then check whether each call site is still correct after the change. Report only call sites that are now broken or need updating.
   i. Agent #9 (`efficiency`): Identify redundant I/O or network calls, sequential work that could be parallel, and blocking operations on a startup or hot path introduced by the diff. Only report issues with real-world impact, not micro-optimizations.
   j. Agent #10 (`altitude`): Assess whether the change is made at the RIGHT abstraction level. Look for bandaids on shared infrastructure that should be fixed at the root, fixes in the wrong layer (e.g. compensating in the UI for a data model problem), or the change solving a symptom rather than the cause.
   k. Agent #11 (`reuse-simplify`): Identify new code in the diff that duplicates existing helpers, utilities, or patterns already in the codebase (propose the existing symbol to use instead), plus redundant state that could be derived, copy-paste variation that could be a shared function, and dead code introduced by the diff. Only report what a senior engineer would actually raise.

   Every finding must include: category, brief description, file path, line range in the modified diff (or `none` if not line-scoped), and reason.

6. **Dedup step — be conservative, double then triple check.** Collect all findings from all review agents. For each candidate pair where file path matches exactly AND line ranges overlap AND categories match:
   a. First check: do the descriptions describe the same underlying problem, not merely a related one?
   b. Second check: launch a Haiku agent with both findings and the PR diff, and ask verbatim: "Would fixing one of these issues fully resolve the other? Answer YES only if they are literally the same issue, not merely related. Cite the specific line(s) of the diff that support your answer."
   c. Third check: verify the agent's citation exists in the actual diff — reject the merge if the cited lines don't support it.
   Only merge when all three checks pass. When in doubt, keep both as separate findings. Duplicate comments are easier to dismiss than conflated ones that drop important context.

7. For each deduped finding, launch a parallel Haiku agent that scores confidence 0–100. Use this rubric verbatim:
   a. 0: Not confident at all. False positive that doesn't stand up to light scrutiny, or a pre-existing issue.
   b. 25: Somewhat confident. Might be a real issue, but might be a false positive. The agent wasn't able to verify. If stylistic, not explicitly called out in CLAUDE.md.
   c. 50: Moderately confident. Verified as a real issue, but may be a nitpick or rare in practice. Not very important relative to the rest of the PR.
   d. 75: Highly confident. Double-checked, likely to be hit in practice. The PR's current approach is insufficient. Important and directly impacts functionality, or directly mentioned in a relevant AGENTS.md or CLAUDE.md.
   e. 100: Absolutely certain. Double-checked and confirmed. Will happen frequently in practice. Evidence directly confirms this.

8. **Apply score tiers:**
   - **<5** → drop (explicit false positive, not worth posting).
   - **5–14** → `INFO`.
   - **15–39** → `MIGHT FIX`.
   - **40–79** → `SHOULD FIX`.
   - **80–100** → `MUST FIX`.

9. **Verification pass — latest Fable and latest Astra in parallel.** Once the scored finding list is complete, send the FULL list (each finding's category, description, file, line range, reason, and current score) plus the PR diff to two independent verifier agents launched in parallel: one Fable agent and one Astra agent. Each verifier must, for every finding, return:
   - a verdict: `CONFIRMED` (can trace the exact failure in the diff), `PLAUSIBLE` (valid concern but not certain), or `REFUTED` (wrong or already handled), with a one-line reason citing diff lines;
   - a corrected 0–100 score using the step 7 rubric, or `agree` if the original score is right.

10. **Reconcile verification results:**
    - If BOTH verifiers return `REFUTED` → drop the finding.
    - If exactly one returns `REFUTED` → keep the finding but treat that verifier's corrected score as 0 for the median below.
    - Final score = median of (original score, Fable score, Astra score), where `agree` means the original score.
    - Re-apply the step 8 tiers to the final score (a finding whose final score falls below 5 is dropped).
    These are the **final verified findings**. If the user asked for results in-chat rather than PR comments (see Output routing above), present the final verified findings now — tier, score, file:line, description, and verification verdicts — and stop here.

11. Use a Haiku agent to re-confirm the PR is still not closed/merged. If it is, stop before posting.

12. **Decide how to post each remaining finding:**
    - If the finding has a concrete line range in the diff → post as a PR review comment anchored to that line (`gh api -X POST /repos/{owner}/{repo}/pulls/{num}/comments`). Review comments thread natively, which matches the user's Resolved/Skipped reply workflow.
    - If the finding has no line range (or its lines are outside all diff hunks) but the finding's file IS in the diff → anchor as a review comment on the nearest relevant changed line in that file. Prefer a line the finding relates to (e.g., the new code that should have had a test, or the new branch that should have updated a doc comment). This keeps all findings threadable.
    - Only fall back to a top-level issue comment (`gh pr comment`) when the finding cannot be associated with any file in the diff.

13. **Match each new finding to prior findings from step 4 before posting:**
    a. Primary match: same category + exact file + line ranges overlap (tolerate ±5 lines to absorb churn).
    b. Ambiguous match: same category + same file, but ranges don't overlap → launch a Haiku agent with both descriptions and ask "Is this the same underlying issue?" Require a clear YES to match. Default to no-match.
    c. On match:
       - If the last comment in the existing thread is from the user (not this skill): post a follow-up reply that is **concrete and current** — name the specific line/behavior that still fails at head sha `<sha>`, not a restatement of the original. For review comments, reply via `POST /repos/{owner}/{repo}/pulls/{num}/comments/{comment_id}/replies`. For issue comments (no native threading), post a new issue comment that quotes and links the original comment URL.
       - If the last comment in the thread is from this skill: skip to avoid nagging. The user's turn to reply.
    d. On no match: post a fresh finding comment.

14. **Post or update the summary comment:**
    - Search existing comments for the `<!-- pr-code-review:summary -->` marker.
    - If one exists: PATCH it via `gh api -X PATCH /repos/{owner}/{repo}/issues/comments/{id}` to reflect the latest run.
    - If none exists: POST a new issue comment.
    - Even if no new findings remain, still update/post the summary (e.g., "No new issues found at <sha>").

**Finding comment body format:**

```
**[TIER — SCORE]** <one-line description>

<reason, with citation: AGENTS.md or CLAUDE.md quote + link / git blame line / referenced prior PR / code comment / `inspect` risk, entity, or dependency evidence>

<!-- pr-code-review:finding:{category}:{file}:{line_start}-{line_end} -->
```

`TIER` is one of `MUST FIX`, `SHOULD FIX`, `MIGHT FIX`, `INFO`. `SCORE` is the 0–100 confidence. For top-level issue comments (no inline anchor), include a permalink to the code using the full-sha link format below.

**Summary comment body format:**

```
### Code review

Reviewed <head_sha>. <N> findings:

- Must fix: <n>
- Should fix: <n>
- Might fix: <n>
- Info: <n>

<bulleted links to each finding comment>

Last run: <ISO timestamp>.

<!-- pr-code-review:summary -->
```

**Link format** (for any code reference inside an issue-comment body — review comments anchor themselves):

`https://github.com/{owner}/{repo}/blob/{full_sha}/{file}#L{start}-L{end}`

- Must use the full sha (no `$(git rev-parse …)`).
- Repo must match the PR's repo.
- Provide ≥1 line of context before and after the target line (e.g., target L5 → link L4–L6).

**Examples of false positives** (filter these out in step 5 before deduping):

- Pre-existing issues
- Something that looks like a bug but isn't
- Pedantic nitpicks a senior engineer wouldn't raise
- Issues a linter / typechecker / compiler would catch (assume CI runs these separately — do NOT run them here)
- General code-quality issues (test coverage, doc polish, generic security) unless a relevant AGENTS.md or CLAUDE.md explicitly calls them out
- Issues flagged in AGENTS.md or CLAUDE.md but explicitly silenced in the code (lint-ignore etc.)
- Functional changes likely intentional or part of the stated scope
- Real issues on lines the PR didn't modify

**Notes:**

- Do not run builds, typechecks, or tests. CI covers those.
- Use `inspect pr` only for hotspot triage; do not substitute its risk scores for manual verification.
- If `inspect` is unavailable or cannot authenticate, continue with the other review passes and do not post an availability failure as a finding.
- Use `gh` (not web fetch) for all GitHub I/O.
- Make a todo list first.
- Keep comment bodies brief, no emojis, always cite and link.
- Verify every code link renders before posting: full sha, correct repo, `#L{start}-L{end}`.
- For steps 6 (dedup) and 13 (cross-run matching), bias toward keeping things separate when uncertain.
- The Fable/Astra verification pass (steps 9–10) is mandatory — never post or present findings that have not been through it.
