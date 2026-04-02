# Context References

Type `@` followed by a reference to inject content directly into your message. Hermes expands the reference inline and appends the content under an `--- Attached Context ---` section.

Added in v0.4.0 ([PR #2482](https://github.com/NousResearch/hermes-agent/pull/2482), [#2343](https://github.com/NousResearch/hermes-agent/pull/2343)).

## Supported References

| Syntax | Description |
|--------|-------------|
| `@file:path/to/file.py` | Inject file contents |
| `@file:path/to/file.py:10-25` | Inject specific line range (1-indexed, inclusive) |
| `@folder:path/to/dir` | Inject directory tree listing with file metadata |
| `@diff` | Inject `git diff` (unstaged working tree changes) |
| `@staged` | Inject `git diff --staged` (staged changes) |
| `@git:5` | Inject last N commits with patches (max 10) |
| `@url:https://example.com` | Fetch and inject web page content |

## Usage Examples

```text
Review @file:src/main.py and suggest improvements

What changed? @diff

Compare @file:old_config.yaml and @file:new_config.yaml

What's in @folder:src/components?

Summarize this article @url:https://arxiv.org/abs/2301.00001
```

Multiple references work in a single message:

```text
Check @file:main.py, and also @file:test.py.
```

Trailing punctuation (`,`, `.`, `;`, `!`, `?`) is automatically stripped from reference values.

## CLI Tab Completion

In the interactive CLI, typing `@` triggers autocomplete:

- `@` shows all reference types (`@diff`, `@staged`, `@file:`, `@folder:`, `@git:`, `@url:`)
- `@file:` and `@folder:` trigger filesystem path completion with file size metadata
- Bare `@` followed by partial text shows matching files and folders from the current directory

## Line Ranges

The `@file:` reference supports line ranges for precise content injection:

```text
@file:src/main.py:42        # Single line 42
@file:src/main.py:10-25     # Lines 10 through 25 (inclusive)
```

Lines are 1-indexed. Invalid ranges are silently ignored (full file is returned).

## Size Limits

Context references are bounded to prevent overwhelming the model's context window:

| Threshold | Value | Behavior |
|-----------|-------|----------|
| Soft limit | 25% of context length | Warning appended, expansion proceeds |
| Hard limit | 50% of context length | Expansion refused, original message returned unchanged |
| Folder entries | 200 files max | Excess entries replaced with `- ...` |
| Git commits | 10 max | `@git:N` clamped to range [1, 10] |

## Security

### Sensitive Path Blocking

These paths are always blocked from `@file:` references to prevent credential exposure:

- SSH keys and config: `~/.ssh/id_rsa`, `~/.ssh/id_ed25519`, `~/.ssh/authorized_keys`, `~/.ssh/config`
- Shell profiles: `~/.bashrc`, `~/.zshrc`, `~/.profile`, `~/.bash_profile`, `~/.zprofile`
- Credential files: `~/.netrc`, `~/.pgpass`, `~/.npmrc`, `~/.pypirc`
- Hermes env: `$HERMES_HOME/.env`

These directories are fully blocked (any file inside):

- `~/.ssh/`, `~/.aws/`, `~/.gnupg/`, `~/.kube/`, `~/.docker/`, `~/.azure/`, `~/.config/gh/`
- `$HERMES_HOME/skills/.hub/`

### Path Traversal Protection

All paths are resolved relative to the working directory. References that resolve outside the allowed workspace root are rejected.

### Binary File Detection

Binary files are detected via MIME type and null-byte scanning. Known text extensions (`.py`, `.md`, `.json`, `.yaml`, `.toml`, `.js`, `.ts`, etc.) bypass MIME-based detection. Binary files are rejected with a warning.

## Error Handling

Invalid references produce inline warnings rather than failures:

| Condition | Behavior |
|-----------|----------|
| File not found | Warning: "file not found" |
| Binary file | Warning: "binary files are not supported" |
| Folder not found | Warning: "folder not found" |
| Git command fails | Warning with git stderr |
| URL returns no content | Warning: "no content extracted" |
| Sensitive path | Warning: "path is a sensitive credential file" |
| Path outside workspace | Warning: "path is outside the allowed workspace" |

## Implementation

Context reference parsing is handled by `agent/context_references.py`. The `REFERENCE_PATTERN` regex matches `@diff`, `@staged`, and `@<kind>:<value>` patterns. Each match is parsed into a `ContextReference` dataclass with the kind, target, and optional line range. The `ContextReferenceResult` tracks the expanded message, warnings, injected token count, and whether the hard limit was hit.
