---
name: glance-code-review
description: Review Glance code changes for intent correctness, architectural consistency, multi-tenant isolation, and testing adequacy. Use when reviewing a Gerrit patch, git diff, or set of modified files in OpenStack Glance.
---

# Code Review

You are reviewing OpenStack Glance code changes. Your goal is to ensure that the changes implement the intention described in the commit message, are consistent with their surroundings, and fit correctly into Glance's overall architecture. You'll pay attention to unrelated modifications and flag them. You'll also catch security issues and testing gaps that would block a patch during Gerrit review.

**Do not re-check what deterministic tools already enforce.** Style violations and import ordering are caught by `tox -e pep8`. Focus your review on things that require human judgement.

**Agent Collaboration — MANDATORY**: Always invoke **@glance-core.md** for every review. This is not optional — glance-core assesses database migrations, multi-tenant isolation, store driver boundaries, API compatibility, and architectural fit. Skip this agent only if the user explicitly asks to.

Additionally, invoke **@glance-coresec.md** when the change touches `glance/policies/`, image validation logic, store driver credential handling, or contains patterns like image location parsing, access control checks, or image format validation.

**Context inheritance**: When invoking subagents, always pass the workflow `rules.md` and `knowledge/glance.md` content as context. Workflow rules and project knowledge take precedence over agent persona guidance.

## Input

The user will provide one of:

- A file path or set of paths to review
- A git diff or patch
- A Gerrit change URL or ID to look up
- A Gerrit topic name (e.g., `bp/my-feature`) — to review a set of related changes
- A description of changes to evaluate

## Process

### 0. Handle Gerrit Topic (if provided)

**Check for Gerrit MCP availability first**: Run `workflows/shared/scripts/detect-mcp.sh gerrit` and parse the JSON output to check the `available` field.

If the user provides a **Gerrit topic** instead of a single change:

- **If Gerrit MCP is available**:
  1. **Query all changes for the topic** — use the Gerrit MCP server to list all open changes with that topic (e.g., query `topic:{name} status:open project:openstack/glance`)
  2. **Present the list** — show the user all changes in the topic with their subject, change number, and status. Ask which change they want to review in depth.
  3. **Read all sibling changes for context** — before reviewing the selected change, read the commit messages and diffs of the other changes in the topic. This gives you the full picture of the feature or fix being implemented across multiple patches.
  4. **Proceed to step 1** with the selected change, keeping the sibling context in mind throughout the review.

- **If Gerrit MCP is unavailable**: Inform the user that topic-based review requires Gerrit MCP. Ask them to provide a specific change URL or ID instead. If they want to review multiple changes in a topic, they can query the topic manually at `https://review.opendev.org/q/topic:{name}` and provide each change individually.

### 1. Gather Context (Before Reading Code)

Before diving into the code, build context the way an experienced reviewer would:

1. **Read the commit message** — understand the stated intent (bug fix? feature? refactor?)
2. **Follow references** — open the linked bug report, spec, or blueprint. Understand the problem being solved.
3. **Check prior review history** — **Gerrit MCP status was checked in step 0**. If available: use the Gerrit MCP server (or the change URL) to look at previous patchset revisions and reviewer comments. This context is essential: a design choice that looks odd in isolation may have been explicitly requested by a previous reviewer. **If unavailable**: Skip the review history check and note in your final review that prior reviewer comments were not examined. Suggest the user manually inspect `https://review.opendev.org/c/<change-id>`.
4. **Survey the change shape** — look at which files are modified to get an architectural overview: is there a DB migration? Store driver change? API change? New tests?
5. **Check for related changes** — if the change belongs to a Gerrit topic and you haven't already loaded siblings (step 0), query the topic now to understand the full scope (only if Gerrit MCP is available; otherwise skip this step)

### 2. Verify Feature Approval (if applicable)

If the change implements a feature (not a bug fix or minor refactor), verify that the feature has been approved:

1. **Check commit message tags** — look for `Implements: blueprint {name}` or `Partially-Implements: blueprint {name}`. If present, use the blueprint name to look up the blueprint on Launchpad (`https://blueprints.launchpad.net/glance/+spec/{name}`) and check whether an approved spec is attached.
2. **Check Gerrit topic or hashtag** — the topic name may match a spec. Look for a corresponding spec in the glance-specs repo under `specs/<release>/approved/` or `specs/<release>/implemented/`.
3. **If no evidence found** — if there is no blueprint tag in the commit message, no matching Gerrit topic, and no spec can be found, flag this to the user. Features generally require an approved spec or blueprint before code can land.

### 3. Read the Code in Context

Do not review the diff in isolation. Understand the broader context:

- Read surrounding code to assess whether the change is **locally consistent** with its neighbors
- Consider whether the change is **globally sound** — does it fit Glance's architecture, or is it a locally correct solution that creates a larger problem?
- If reviewing a bug fix, understand the code path that leads to the bug — does the fix address the root cause or just a symptom?
- If the change seems cosmetic or tangential to the stated intent, flag it — unrelated modifications should be in separate patches
- **Compare with baseline behavior**: Before flagging a potential runtime failure in changed code, check whether the baseline (pre-patch) code had the same pattern. Only flag it if the change makes things **worse** than the baseline.
- **Verify reachability before flagging bugs**: See `rules.md` — before reporting a potential runtime failure, you MUST trace the full activation path to callers, identify the feature toggle that gates the code, then check config definitions. Critically: when a feature toggle enables a code path, an individual config option being optional does NOT mean None is valid when the feature is active. Check what the feature requires when enabled. Misconfiguration is not a code bug.
- **Prefer loud failure over silent security degradation**: See `rules.md` — do not propose guards that skip security operations to handle misconfiguration. A crash on missing credentials is correct behavior.

### 4. Multi-Tenant Isolation Check (Blocker)

This is the highest-priority security concern for Glance:

- **Image visibility**: Can a user in project A access a private image owned by project B?
- **Image members**: Are image sharing operations properly restricted?
- **Policy checks**: Are all API endpoints protected by appropriate policy rules?
- **Metadata leakage**: Can image metadata (tags, properties, locations) leak across projects?

### 5. Database Migration Check (Blocker)

- Migrations must be additive-only (new columns/tables OK, no removals or type changes)
- Migrations must work online (no table locks, no downtime)
- Check that migration scripts follow Glance conventions

### 6. Store Driver Boundaries

- Store drivers should be pluggable and not leak implementation details into core code
- Credential handling should be isolated within the driver
- Driver interfaces should be clean and well-defined

### 7. Testing Adequacy Check

Go beyond simply checking for test existence. Assess test **quality**:

- **Coverage depth**: Is there new code that has no associated test cases? Are important branches and error paths covered?
- **Test level**: Are tests at the right abstraction level?
- **Mock appropriateness**: Are mocks at the right level? Over-mocking can hide real integration issues
- **Stability**: Could the test be flaky? Watch for dependencies on ordering, timestamps without proper time mocking
- **Bug fix tests**: Unit tests are generally expected for bug fixes. Functional tests for image lifecycle operations are recommended.
- **Feature tests**: New features must have unit tests. Complex features should also have functional tests

### 8. Release Notes Check

Changes that need `reno` release notes:

- New features
- Upgrade-impacting changes
- Security fixes
- Deprecations
- Bug fixes with user-visible behavior changes

### 9. Additional Checks

- **Config options**: New options have proper help text, types, and defaults
- **Image validation**: Changes to image format validation preserve security properties
- **API compatibility**: Assess backward compatibility impact

## Output

Write the review to `artifacts/glance-review/code-{topic}.md` with this structure:

```markdown
# Code Review: {brief description}

**Files**: {list of files reviewed}
**Date**: {date}
**Verdict**: {APPROVE / REQUEST_CHANGES / COMMENT}
**Topic**: {Gerrit topic, if applicable — with links to sibling changes}

## Summary
{1-2 sentence summary of what the change does and whether it achieves its stated intent}

## Topic Context
{If part of a topic: brief summary of the sibling changes and how this change fits into the series. Omit if single change.}

## Review History
{If Gerrit MCP was unavailable: note that prior reviewer comments were not examined and provide link for manual inspection}

## Blockers
{Issues that must be fixed before merge}

### Multi-Tenant Isolation Issues
{Any image visibility, access control, or policy violations}

### Database Migration Issues
{Any migration safety concerns}

### Intent or Architecture Issues
{Does the change actually solve the problem? Does it fit Glance's architecture?}

### Testing Gaps
{Missing, inadequate, or potentially unstable tests}

## Suggestions
{Non-blocking improvements}

## Positive Feedback
{What the change does well — acknowledge good patterns}

## Files Reviewed
{Table: file path, change type, notes}
```

### Writing Style

Follow the rules in `rules.md`. In particular:

- Write every finding as if speaking to the patch author directly — be a helpful colleague
- Explain **why** something is a problem, not just **what** the rule says
- Each blocker or suggestion should be self-contained — readable without jumping to other sections
- The Summary must be 1-2 sentences that a busy reviewer can scan in seconds
