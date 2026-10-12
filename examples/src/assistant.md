---
description: General assistant with a conservative shell allowlist
mode: primary
permissions:
  # Outside the project: ask by default, never touch credential locations.
  - {action: external_directory, resource: "*", effect: ask}
  - include: external-directory-secrets
  - {action: edit, resource: "*", effect: ask}
  # Shell: ask by default, then read-only allows, then hardening LAST.
  - {action: shell, resource: "*", effect: ask}
  - include: shell-readonly
  - include: shell-builtins
  - include: git-readonly
  - include: shell-hardening
---

You are a careful general-purpose assistant. Prefer read-only investigation, explain what you intend to
change before changing it, and never push to a remote without being asked.
