---
name: vault-librarian
description: Interact with opencode-vault — an AI-managed, git-tracked markdown knowledge vault with sync, search, and deduplication workflows.
---

# Vault Librarian Skill

## Overview

This skill enables agents to interact with an **opencode-vault** -- an AI-managed, git-tracked knowledge vault stored as markdown files. The vault is designed for fast retrieval, atomic updates, and multi-machine synchronization.

**Default vault location**: `~/opencode-vault`

> You MUST read the vault's `README.md` and `INDEX.md` before performing any operations. These files describe the vault's specific structure, folder conventions, and tag taxonomy. This skill only describes *how* to interact with any opencode-vault -- the vault itself tells you *what's in it*.

## Getting Started

1. Determine the vault path. Default is `~/opencode-vault`. If the user specifies a different path, use that.
2. Run the git sync workflow (see below) before doing anything.
3. Read `README.md` at the vault root to understand this vault's folder structure, conventions, and any vault-specific rules.
4. Read `INDEX.md` to get the full catalog of every note -- titles, tags, types, and one-line summaries.

## Git Sync Workflow

The vault may be used across multiple machines. Always sync before and after changes.

### Before Any Operation

```bash
# 1. Check if a remote is configured
git remote -v

# 2. If a remote exists, detect the current branch and pull
BRANCH=$(git branch --show-current)
git pull --rebase origin "$BRANCH"
```

- If **no remote is configured**: skip pull/push for the entire session, but warn the user that changes are local-only.
- If **rebase encounters conflicts**:
  - `INDEX.md` conflicts: accept the incoming version (`git checkout --theirs INDEX.md`), then regenerate/update the index after your changes are complete.
  - Note conflicts: keep both versions. Rename your conflicting version with a `-conflict-YYYY-MM-DD` suffix and flag it for the user to review. Then `git add` the resolved files and `git rebase --continue`.

### After Every Commit

```bash
BRANCH=$(git branch --show-current)
git push origin "$BRANCH"
```

- If push fails because the remote has new commits: `git pull --rebase origin "$BRANCH"`, resolve any conflicts as above, then retry push.
- If push still fails: **warn the user**. Do NOT force push. Ever.

### Principles

- Pull-edit-commit-push as a tight atomic sequence. Minimize the window between pull and push.
- **Never force push.**
- **Never skip the pull step** when a remote is configured.

## Workflows

### Searching / Retrieving Information

1. Read `INDEX.md`. It contains a table of every note organized by folder, with tags, type, and a summary.
2. Scan the index for relevant notes by matching tags, titles, or summary keywords.
3. Read specific notes only after identifying them from the index.
4. If the index doesn't have what you need, use grep to search note contents directly within the vault directory.
5. Return findings to the user with file paths and relevant excerpts.

### Adding a New Note

1. **Check for duplicates**: search `INDEX.md` for notes with similar titles, tags, or topics. If a related note exists, prefer updating it instead of creating a new one.
2. **Determine the correct folder**: read the vault's existing folder structure from `INDEX.md` and `README.md`. Place the note in the most appropriate existing folder. Do not create new top-level folders without user approval.
3. **Create the note** with the required frontmatter (see Frontmatter Schema below).
4. Write content in standard markdown. Use `[[wikilinks]]` to reference other notes in the vault.
5. **Update `INDEX.md`**: add a row to the appropriate folder's table with the note's title (as a wikilink), tags, type, and a 5-15 word summary.
6. **Commit and push**:
   ```
   vault: add <note-title>
   ```

### Updating an Existing Note

1. Read the current note content.
2. Make changes. Update the `modified` field in frontmatter to today's date.
3. If tags or summary changed, update the corresponding row in `INDEX.md`.
4. **Commit and push**:
   ```
   vault: update <note-title>
   ```

### Deduplication

Before adding new content, always check for overlap:

1. Search `INDEX.md` for notes with similar tags, titles, or summaries.
2. If a related note exists, prefer updating it over creating a new one.
3. If two notes cover the same topic, merge them into one and delete the other. Update `INDEX.md` accordingly (remove the deleted note's row, update the surviving note's row).
4. **Commit and push**:
   ```
   vault: merge <note-a> into <note-b>
   ```

### Deleting a Note

1. Remove the file.
2. Remove the corresponding row from `INDEX.md`.
3. Check other notes for broken `[[wikilinks]]` to the deleted note and remove or update them.
4. **Commit and push**:
   ```
   vault: delete <note-title>
   ```

## Git Commit Rules

- **Every change** to the vault MUST be committed immediately after the change is made.
- Use descriptive commit messages prefixed with `vault:`:
  - `vault: add <title>` -- new note
  - `vault: update <title>` -- modified note
  - `vault: merge <a> into <b>` -- deduplication
  - `vault: delete <title>` -- removed note
  - `vault: update index` -- index-only changes
  - `vault: reorganize <description>` -- structural changes
- **Never commit secrets, credentials, or API keys.**
- All git commands use the vault root as the working directory.

## Frontmatter Schema

Every note MUST have these fields in YAML frontmatter:

```yaml
---
title: <Note Title>
tags: [<tag1>, <tag2>]
created: <YYYY-MM-DD or "unknown">
modified: <YYYY-MM-DD>
type: <note|daily|template|gateway|project|excalidraw|canvas>
---
```

| Field      | Type   | Description                                              |
|------------|--------|----------------------------------------------------------|
| `title`    | string | Human-readable title                                     |
| `tags`     | list   | 2-5 tags. Reuse existing tags from INDEX.md first.       |
| `created`  | string | ISO date or `"unknown"` if not determinable              |
| `modified` | string | ISO date, updated on every edit                          |
| `type`     | string | Note category (see values above)                         |

**Tag discipline**: before inventing a new tag, check what tags already exist in `INDEX.md`. Reuse existing tags for consistency. Only create new tags when no existing tag fits.

## Important Rules

- **NEVER** create or store secrets, credentials, or API keys in the vault.
- **NEVER** modify Excalidraw files (`.excalidraw.md`) -- they contain plugin-managed drawing data.
- **ALWAYS** update `INDEX.md` when adding, removing, or significantly modifying notes.
- **ALWAYS** sync (pull before, push after) when a remote is configured.
- Use `[[wikilinks]]` for cross-references between notes.
- Keep notes **atomic** -- one topic per note when possible.
- **Prefer updating** existing notes over creating new ones to avoid duplication.
- Do not create new top-level folders without user approval.
