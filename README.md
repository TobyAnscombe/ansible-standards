# Ansible / Ubuntu Project Standards

Canonical, written conventions for Ansible automation projects targeting
Ubuntu hosts. This repo exists so the standards live in one place instead
of being copied (and drifting) between individual playbook repos.

These conventions were established in and are best illustrated by
[`ubuntu-user-creation`](https://github.com/TobyAnscombe/ubuntu-user-creation),
which remains the reference implementation. New Ansible projects should
follow this document; `ubuntu-user-creation` and other existing repos
should be brought into line with it over time rather than treated as
separate sources of truth.

## Repository layout

Every project root should contain:

```
inventory.ini
playbook.yml
requirements.yml     # Ansible Galaxy collection dependencies
roles/
  <role_name>/
    tasks/
      main.yml        # includes the files below, in order
      validate.yml    # pre-flight checks / assertions
      user.yml         # user account management
      ssh.yml          # SSH key + sshd config
      sudo.yml         # sudoers management
    vars/
      *.yml            # variable-driven config, e.g. users.yml
```

Task files are split by concern rather than written as a single monolithic
`main.yml`. `main.yml`'s only job is to `include_tasks` the others in the
right order.

Configuration is variable-driven: concrete values (which users exist,
whether a given user gets `sudo: true`, etc.) live in `vars/`, not hardcoded
in tasks. This keeps the role generic and reusable across inventories.

## Design principles

- **Idempotent by default.** Every task must be safe to re-run with no
  side effects on a host that's already converged. Prefer Ansible modules
  with built-in idempotency over `shell`/`command`; when `shell`/`command`
  is unavoidable, guard it (`creates`, `when`, a preceding check task).
- **Variable-driven configuration.** Behaviour changes by editing `vars/`,
  not by editing task logic.
- **Simplicity first.** Prefer the straightforward, readable task over a
  clever one. Minimal impact: change what needs to change, nothing else.

## Security posture

- **SSH-key-only authentication.** No password-based SSH access. Managed
  users get `password_lock` set — the account exists and can `sudo`, but
  cannot log in with a password.
- **Sudoers validated via `visudo`.** Sudoers entries are templated and
  always validated (`visudo -cf`) before being applied, to catch syntax
  errors before they lock out sudo entirely.
  - Baseline pattern: `NOPASSWD: ALL` for interactive admin users.
  - Hardened pattern (for **service accounts**, e.g. the Ansible
    connection account itself): set `shell: /usr/sbin/nologin` to block
    interactive sessions while preserving SSH key auth for Ansible, and
    scope sudoers to the specific commands that account needs rather than
    `NOPASSWD: ALL`. This limits blast radius if the account's key is ever
    compromised.
- **Secret scanning.** [`gitleaks`](https://github.com/gitleaks/gitleaks)
  runs as both a pre-commit hook (catches secrets before they're committed)
  and a GitHub Actions workflow on push/PR, uploading results as SARIF so
  findings show up in the repo's Security tab.

## Tooling

- **Ansible**, with all Galaxy collection dependencies declared explicitly
  in `requirements.yml` — no relying on collections that happen to be
  preinstalled on the control node.
- **gitleaks** for secret scanning (pre-commit + CI).
- **GitHub Actions** for CI, including SARIF upload for gitleaks findings.

## Known troubleshooting pattern

If passwordless sudo isn't working for a managed user, check the `sudo:`
flag on that user's entry in `vars/users.yml` first — it's very rarely the
sudoers template, which already contains `NOPASSWD: ALL`.

## Applying these standards to a new project

1. Scaffold the repository layout above.
2. Write `vars/` first — decide what's configurable before writing tasks
   against it.
3. Split `tasks/main.yml` into `validate.yml` / `user.yml` / `ssh.yml` /
   `sudo.yml` (or the equivalent breakdown for the role's actual
   responsibility) from the start, rather than refactoring a monolith
   later.
4. Wire up `gitleaks` pre-commit and the CI workflow before the first real
   commit lands.
5. For any account the automation itself uses to connect (as opposed to
   accounts it's provisioning), default to the hardened service-account
   pattern (`nologin` shell, scoped sudoers), not the baseline interactive
   pattern.

## Scope

This repo documents conventions only — it is intentionally not a
distributable Ansible role or collection. Individual projects implement
these conventions themselves; `ubuntu-user-creation` is the worked example
to copy from.
