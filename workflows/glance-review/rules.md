# Glance Review Workflow Rules

This document contains rules and guidelines for the Glance Review workflow agent.

## MCP Server Integration

### Gerrit MCP Availability

The Glance Review workflow supports both **Gerrit MCP** and **REST API fallback** modes:

- **With Gerrit MCP**: Full integration - fetch change history, metadata, post reviews programmatically
- **Without Gerrit MCP**: REST API fallback - post reviews using HTTP basic authentication

**At workflow startup**, MCP availability is automatically detected. The agent will report the status and adapt accordingly.

### When Gerrit MCP is Unavailable

If Gerrit MCP is not available or connection fails:

1. **Review History** (`/glance-code-review` skill):
   - Gerrit review history check is skipped
   - Agent notes in review output that history was not checked
   - Suggests manual inspection at `https://review.opendev.org/c/<change-id>`

2. **Future: Review Posting** (when `/glance-gerrit-comment` skill is added):
   - Will automatically fall back to Gerrit REST API
   - Prompts user for HTTP credentials (username and password)
   - Credentials never stored — prompted each time, cleared after use

### Error Handling

**Clear Error Messages**: Distinguish between:
- "Gerrit MCP unavailable" (use REST API fallback)
- "REST API authentication failed" (invalid credentials)
- "Network error" (cannot reach review.opendev.org)
- "Operation failed" (other issues)

**Remediation Steps**: Every error message includes specific next steps:
- Network errors → check VPN, proxy, firewall
- Authentication errors → verify Gerrit credentials, check account status
- MCP errors → configure MCP server or continue with REST API

### User Cancellation

Users can cancel at any prompt by typing 'cancel':
- During HTTP credential prompting
- During retry confirmation
- Operation halts cleanly, no partial state

### Security

**HTTP Basic Authentication** (when implemented):
- Credentials prompted via secure input (password hidden)
- Transmitted over HTTPS only
- Cleared from memory immediately after use
- Never logged or stored in artifacts

**No Credential Storage**:
- No gitcookies, API tokens, or passwords persisted
- Every REST API operation prompts for fresh credentials
- Credentials valid only for single session

## Review Principles

### Verify reachability before flagging bugs

Before reporting a potential runtime failure (e.g., `None` passed where a path is expected, missing attribute, type mismatch), you MUST verify that the failing scenario is actually reachable under valid configuration. Do not assume a value can be `None` just because the local code doesn't guard against it. Perform these checks **before** classifying anything as a bug:

1. **Trace the full activation path** — find where the class is instantiated or the function is called. Read the caller code to understand what values are actually passed and under what conditions. Do not stop at the immediate caller; follow the chain until you reach the entry point (e.g., a config-driven factory, a service startup path). Identify which config option or condition **gates** this code path — i.e., what must be true for this code to execute at all.
2. **Check config option definitions AND their interdependencies** — if the code path depends on a configuration option, look up the option's definition (type, default, required/optional) and its documentation. Critically, **connect the activation path from step 1 to the config check**: when a feature toggle enables a code path, check what the feature's documentation says is required when it is enabled. An individual config option being technically optional does NOT mean `None` is a valid input when the feature that depends on it is active. The valid configuration space is the intersection of all options that matter when the feature is enabled, not each option checked in isolation.
3. **Distinguish misconfiguration from code bugs** — if the problematic input can only occur when the operator has set up an inconsistent or incomplete configuration (e.g., enabling a feature but not providing its required credentials), it is **not a code bug in the patch**. At most, suggest improving config validation at service startup as a separate improvement. Do not block the patch for it.

### Prefer loud failure over silent security degradation

Never suggest a guard or conditional that skips a security operation (credential validation, signature verification, access checks, image validation) to "fix" a potential crash on bad input. A `TypeError` or `FileNotFoundError` on operator misconfiguration is **always better** than silently degrading a security property. If code crashes because a required security credential is missing, that is correct behavior — flag it at most as a config-validation improvement opportunity, not as a bug in the patch. Do not propose `if value:` guards around security operations unless you have confirmed that the unguarded path is reachable under **valid** configuration.

## Writing Style

Follow these guidelines when generating review artifacts:

### Be Specific

- Cite file paths and line numbers for every finding
- Reference exact Glance conventions or documentation sections
- Provide concrete examples of both the problem and the fix

### Distinguish Severity

- **Blockers**: Multi-tenant isolation violations, database migration issues, missing required tests, architectural misfit
- **Suggestions**: Performance optimizations, edge case handling, better error messages
- **Nits**: Minor naming or formatting preferences

### Reference In-Tree Documentation

- Link to `doc/source/contributor/` for contribution guidelines
- Reference Glance's in-tree docs rather than duplicating rules
- If docs are incomplete, suggest improving them upstream

### Be Constructive

- Suggest fixes, not just problems
- Explain **why** something is an issue, not just **what** the rule says
- Assume good intent - the author is trying to improve Glance

### Keep Summaries Scannable

- Top-level summary must be 1-2 sentences
- Busy reviewers should understand the verdict in seconds
- Details belong in separate sections, not the summary
