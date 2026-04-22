# Glance — Project Reference

Glance is OpenStack's image service for discovering, registering, and retrieving virtual machine images. It provides a REST API for uploading and querying images and supports multiple storage backends.

## Project Links

- **Repository**: https://opendev.org/openstack/glance (GitHub is a mirror only)
- **Bug tracking**: https://bugs.launchpad.net/glance
- **Code review**: Gerrit at https://review.opendev.org (not GitHub PRs)
- **Docs**: https://docs.openstack.org/glance/latest/
- **Contributor guide**: `doc/source/contributor/` in the Glance repo
- **Specs**: `openstack/glance-specs` — `specs/<release>/approved/`, `specs/<release>/implemented/`, `specs/backlog/`, `specs/abandoned/`

For directory structure, core services, storage drivers, configuration patterns, and test commands, refer to the Glance repository's in-tree documentation at `doc/source/contributor/`. Do not duplicate that information here — read it directly from the source.

## Architecture Overview

```text
┌─────────────┐
│ glance-api  │  ← REST API service (image metadata + upload/download)
└──────┬──────┘
       │
       ├─→ Database (image metadata, locations)
       ├─→ glance_store (separate library - all store drivers)
       └─→ Keystone (authentication)
```

## glance_store Library

**Critical architectural detail**: Store drivers (filesystem, Swift, Ceph, S3, HTTP, etc.) live in a **separate repository and library** called `glance_store`, not in the main Glance tree.

- **Repository**: https://opendev.org/openstack/glance_store
- **Purpose**: Pluggable storage backend abstraction layer
- **Used by**: Glance, Cinder (for image volume cache), and potentially other projects
- **Driver location**: `glance_store/_drivers/` (not `glance/store/`)

### Key Implications for Review

- **Store driver changes**: Patches to store drivers go to the `glance_store` repo, not `glance`
- **Interface changes**: Changes to the store driver interface in `glance_store` may require coordinated changes in Glance
- **Testing**: Store driver tests live in `glance_store`, but Glance should have integration tests for store interactions
- **Versioning**: `glance_store` has its own release cycle and version constraints in Glance's `requirements.txt`

When reviewing Glance patches that interact with image storage, check whether changes belong in `glance` or `glance_store`.

## Coding Conventions

### Deterministic Checks (enforced by CI)

Style violations, import ordering, and Glance-specific hacking checks are enforced by `tox -e pep8`. Do not manually re-check these during review.

### Conventions That Require Human Judgement

- **glance_store integration boundaries**: Code interacting with `glance_store` should use the defined interface, not reach into driver internals
- **Architectural fit**: Changes should be locally consistent with surrounding code and globally fit Glance's architecture
- **Test quality**: Assess coverage depth, mock appropriateness, stability
- **Cache vs. store separation**: Image cache (SQLite-backed, in `glance/image_cache/`) is separate from glance_store drivers

### REST API

- Use "image" for image resources
- URLs use hyphens; request bodies use snake_case
- Standard HTTP status codes: 201 for creation, 204 for deletion, etc.

## Security Considerations

- **Image data validation**: Uploaded images must be validated to prevent malicious content
- **Multi-tenant isolation**: Image visibility (public/private/shared) must be enforced correctly
- **Credential handling**: Store backend credentials (Swift, S3, Ceph) are configured in Glance but passed to glance_store drivers — review both sides for credential leakage

## Operations Requiring Human Review

- Any database migration
- Changes to config option defaults (defined across modules, registered in `glance/opts.py`)
- Changes to `glance/policies/` defaults
- Changes to glance_store interface or integration points
- Image format validation changes
- Image cache mechanism changes (`glance/image_cache/`)

## Commit Conventions

- Glance uses **Gerrit**, not GitHub PRs
- Release notes are mandatory for upgrade, security, or feature-impacting changes (use `reno`)
- Commit messages: reference Launchpad bug IDs with `Closes-Bug: #NNNNNN` or `Related-Bug: #NNNNNN`
