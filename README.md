# opencode-permission-sets

Reusable permission rule sets for [OpenCode](https://opencode.ai) agents, plus a tiny build tool that
composes them. Define your shell allowlist once, include it from as many agents as you like.

> **Status:** personal baseline, shared as-is. Read every rule before you use it. This is not a sandbox:
> permission rules match command text and cannot understand every shell construct. See
> [Disclaimer](#disclaimer).

## Why

OpenCode agents carry their permissions inline, so a useful shell allowlist (dozens of read-only allows plus
deny rules for the dangerous cases) gets copied into every agent. When you tighten a rule you must remember
every copy, and a copy that misses it silently keeps the hole open. This repo keeps each group of rules in one
file and splices them into agents at build time. It is a stopgap until OpenCode supports this natively
(tracked in [anomalyco/opencode#54893](https://github.com/anomalyco/opencode/issues/54893)).

## How it works

An agent **source** is a Markdown file whose frontmatter contains `include` lines:

```yaml
---
description: General assistant
mode: primary
permissions:
  - {action: shell, resource: "*", effect: ask}
  - include: shell-readonly
  - include: git-readonly
  - include: shell-hardening      # last: denies must follow the allows
---
You are a helpful assistant.
```

`build.py` replaces each `include` with the rules from `permissions/<name>.yml`, **in place**. OpenCode uses
"last matching rule wins", so order is the policy; the build never reorders anything.

```sh
python3 build.py agents/src agents            # write agents/*.md
python3 build.py --check agents/src agents    # exit 1 if generated files are stale
python3 build.py --path ./private-sets agents/src agents   # private sets searched first
```

- Standard library only. If PyYAML is installed, output is validated (every rule needs `action`, `resource`,
  `effect`).
- `--path` is repeatable; earlier paths win, so a private directory can add to or override these sets.
- Output starts with a `GENERATED` comment and `>>> include` / `<<< include` markers around each fragment.

## Sets

Rules use the V2 list format. `shell` rules are raw command text with `*` globs.

| Set | Rules | What it does |
|-----|------:|--------------|
| `shell-readonly` | 66 | `cat`, `ls`, `grep`, `find`, `head`, `jq`, `diff`, `stat`, ... Inspection only. |
| `shell-builtins` | 6 | `cd`, `test`, `read`, `command -v`: pieces of compound commands. |
| `shell-file-inspection` | 5 | `pdftotext`, `pdfinfo`, `unzip -l`, `zipinfo`, `cmp`. |
| `shell-fs-create` | 4 | `mkdir`, `touch`. Not read-only, but non-destructive. |
| `git-readonly` | 53 | `status`, `diff`, `log`, `show`, plumbing, config reads, `ls-remote`, plus `git -C <path>` forms. |
| `git-sync` | 6 | `fetch`, `pull`, `clone`, `gh repo clone`. Moves data in; never pushes. |
| `git-local-write` | 20 | `add`, `commit`, `init`, `mv`, `branch`, `tag`, `remote`, `archive`. See the note below. |
| `github-readonly` | 2 | `gh repo list`, `gh repo view`. `gh api` is deliberately absent. |
| `opencode-readonly` | 1 | `opencode api get *`. |
| `external-directory-secrets` | 18 | Denies reads of `~/.ssh`, `~/.aws`, `~/.gnupg`, shell rc files, `.env`, tokens, ... |
| `shell-hardening` | 27 | Denies `rm -r/-f`, `sudo`, `dd`, `mkfs`, `curl\|sh`, `sed -i`, `tee`, `find -exec/-delete`, redirects. **Include last.** |

Ordering rules of thumb:

1. Start with the agent's default (`shell * ask`).
2. Include the allow sets.
3. Include `shell-hardening` **last** so its denies override earlier allows (for example `find *` is allowed
   by `shell-readonly` and `find * -exec*` is denied by hardening).
4. Put `external_directory * allow` (or `ask`) **before** `external-directory-secrets`.

### `git push` is intentionally absent

No set contains a rule for `git push`, allow or deny, and none should. Every push should prompt. If you add
one, you are removing that safety.

### Known loose ends

- `git-local-write` allows `git branch *`, `git tag *` and `git remote *`, which also match their delete and
  modify forms. Drop them (or move them to a separate set) for a stricter baseline.
- `git archive *` can write files via `--output`.
- `git pull` can change the working tree.
- Review whether you want `git config --get*` readable at all: it can print values from your global config.

## Private sets

Machine- or account-specific rules (a particular desktop's diagnostics, a named backup remote) do not belong
here. Keep them in your own repo, point `--path` at it, and `include` them from your private agent source.
The same goes for anything that names your skills or tools.

## License

[GPL-3.0-or-later](LICENSE). This is a strong copyleft license: if you distribute these sets, or a modified
version of them, you must do so under the same license with source available. Using them privately, including
building agent files for your own machines, creates no obligation to publish anything. Generated agent files
embed rules from these sets, so the same applies if you distribute those generated files.

## Disclaimer

Permission rules are a convenience layer, not a security boundary. They match command text, and compound
commands, redirects, substitutions and aliases can behave differently from how a rule reads. Treat these sets
as a starting point, audit them against how your agent actually behaves, and keep secrets out of any directory
the agent can reach. Review changes to these files like changes to a firewall.

## Contributing

Rule changes widen or narrow what an agent can do without asking, so every change is reviewed with that in mind.
Prefer small PRs that state which command is being allowed or denied and why.
