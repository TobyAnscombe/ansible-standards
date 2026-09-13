# Ansible / Ubuntu Project Standards

Canonical, written conventions for Ansible automation projects. This repo
exists so the standards live in one place instead of being copied (and
drifting) between individual playbook repos.

This document was reviewed against the actual repos under `DevWork/claude`
(`ubuntu-user-creation`, `ansible-control`, `ubuntu-patching`,
`ansible-proxmox-caddy`, `ansible-proxmox-arrstack`, `ansible-cyberark`,
`ansible-cyberark-ssh-provisioning`, `htb-ssh-key-rotation`,
`github-actions`) rather than written from memory alone, so it reflects
what's actually in place, not just what was originally intended.
`ubuntu-user-creation` remains the most complete worked example.
`ansible-proxmox-homepage` plus its three role repos
(`ansible-role-lxc-provision`, `ansible-role-unifi`, `ansible-role-caddy`)
are the source for the "Proxmox LXC provisioning pattern" section below —
that pattern is now confirmed against a live Proxmox/UniFi/Caddy fleet, not
just written to spec.

## Repository layout

Every project root should contain:

```
ansible.cfg
inventory.ini            # gitignored — contains real hostnames/IPs
inventory.ini.example    # committed template
playbook.yml             # or site.yml for a top-level multi-role play
requirements.yml         # Galaxy collections AND/OR roles from other repos
.ansible-lint            # profile + skip_list overrides
.pre-commit-config.yaml  # gitleaks hook
.gitleaks.toml           # gitleaks rule config
.gitignore
.github/workflows/ci.yml # thin caller into TobyAnscombe/github-actions
vars/
  *.yml                  # real values — gitignored
  *.example.yml          # committed template, if used
roles/
  <role_name>/
    tasks/...
    defaults/
    handlers/
    meta/
```

`CLAUDE.md`, `tasks/*.md`, and `.claude/` exist on disk in every project but
are **never committed** — see "Claude Code working files" below.

## Role distribution: local vs. cross-repo

Two patterns are both in active use, and the choice depends on whether a
role is single-purpose to one project or shared:

- **Vendored in-repo** (`ansible-control`, `ansible-proxmox-caddy`,
  `ansible-proxmox-arrstack`, `ubuntu-patching`): roles live under
  `roles/<name>/` in the same repo as the playbook that uses them.
- **Installed via `requirements.yml`** (`htb-ssh-key-rotation`,
  `ansible-cyberark-ssh-provisioning`): the role has its *own* repo,
  tagged with semver, and consuming playbooks pull it in:

  ```yaml
  roles:
    - name: ubuntu_manage_user
      src: https://github.com/TobyAnscombe/ubuntu-user-creation
      version: v1.1.10
  ```

  When a role is distributed this way, the consuming repo's `.gitignore`
  excludes `roles/` entirely — the role is installed at run time
  (`ansible-galaxy role install -r requirements.yml`), never vendored.

  Default to vendoring for a role that's genuinely single-project.
  Promote a role to its own repo (and start tagging releases) once a
  second project needs it — `ubuntu_manage_user` and the `cyberark_api_*`
  roles are the established examples.

## Role task-file structure

The task-file split is real and consistently applied, but the *specific*
files depend on what the role does — there is no single universal set
beyond `main.yml` (dispatch only) and `validate.yml` (pre-flight
assertions, which should be present in every role):

- **User/account management** (`ubuntu_manage_user`): `validate.yml`,
  `user.yml`, `ssh.yml`, `sudo.yml`, plus role-specific extras
  (`add_sshd.yml`, `manage_user.yml`).
- **Service install** (`caddy`, `servarr_app`, `caddy_lxc`, `unifi_dns`):
  `validate.yml`, `install.yml`, `configure.yml`, `service.yml`.
- **Maintenance/operational** (`ubuntu_patch`, `cyberark_ssh_key_rotation`):
  a smaller, task-specific set (e.g. `check_window.yml`, `patch.yml`,
  `reboot.yml`, or `discover_accounts.yml`, `rotate_key.yml`).

`main.yml`'s only job is to `include_tasks` the others in the right order.

Configuration is variable-driven: concrete values live in `vars/`
(project-level) or `defaults/`/`vars/` (role-level), not hardcoded in
tasks.

## Proxmox LXC provisioning pattern

Confirmed end-to-end against the live fleet via `ansible-proxmox-homepage`
(2026-09-13): create an LXC, give it a stable IP, point DNS at it, deploy
the app, register it with Caddy. Three cross-repo roles cover the
reusable parts — `ansible-role-lxc-provision`, `ansible-role-unifi`,
`ansible-role-caddy` — pulled in via `requirements.yml` like
`ubuntu_manage_user`. A consuming repo (e.g. `ansible-proxmox-homepage`)
supplies only its own `vars/<service>.yml` and an app-specific role.

Playbook shape (three plays, one repo per app):

```yaml
- hosts: proxmox
  roles: [lxc_provision]
  post_tasks:
    - include_role: {name: unifi}   # fixed IP, when: lxc_ip == 'dhcp'
    - include_role: {name: unifi}   # DNS record, always

- hosts: <app>                       # populated in-memory by lxc_provision's add_host
  roles: [<app>]

- hosts: caddy_lxc
  tasks:
    - include_role: {name: caddy, tasks_from: register_service}
```

**VMID/IP discovery** (`lxc_provision`): look up an existing container by
hostname first — `pct list | awk -v h="<hostname>" 'NR>1 && $NF==h
{print $1}'` — `Name` is confirmed the last column on real `pct list`
output regardless of whether `Lock` is populated, so `$NF` is safe. Only
allocate a fresh VMID (`pvesh get /cluster/nextid --output-format json`,
which returns a JSON-quoted string like `"100"` — strip the quotes) when
nothing matched. **This existing-hostname check is also the idempotency
seam and a hazard**: if a run fails after `pct create` but before
`pct start` completes, the *next* run's `pct list` match makes it treat
the container as already existing and **silently skip create, bind
mounts, start, and the SSH wait** — adopting a half-built container
rather than fixing it. Don't blindly re-run after a mid-`create.yml`
failure; check `pct status <vmid>` and consider `pct destroy` first.

**Networking convention**: default new LXCs to `ip=dhcp` on `net0` (the
fleet norm) and set a UniFi fixed-IP reservation afterward for a stable
address — reserve a static `pct`-level IP only for the one LXC everything
else's DNS must point at unconditionally (the Caddy LXC). Discover a
DHCP-assigned address with `pct exec <vmid> -- hostname -I`, retried
(DHCP takes a few seconds after `pct start`), then `add_host` it into a
group for the app's own play.

**UniFi has two unrelated APIs — do not conflate their site identifiers.**
This is the one mistake actually made building this pattern, so it's
recorded here to not be rediscovered the hard way:

- *Network Integration API* (`/proxy/network/integration/v1/...`,
  `X-API-Key` header) — used for DNS records. Site-scoped by
  `unifi_site_id`, the site's **UUID**, e.g.
  `.../sites/{{ unifi_site_id }}/dns/policies`.
- *Legacy/local Network API* (`/proxy/network/api/s/{site}/...`,
  session-cookie + `X-CSRF-Token` from `POST /api/auth/login`) — the only
  surface that exposes client fixed-IP/DHCP-reservation assignment; the
  Integration API doesn't have it at all. `{site}` here is the site's
  short **`internalReference`** (e.g. `default`) — passing the
  Integration API's UUID 401s with `api.err.NoSiteContext`. Resolve it
  once via `GET .../integration/v1/sites` (matching `id` to
  `unifi_site_id`, reading `.internalReference`) before making any legacy
  call; `ansible-role-unifi`'s `legacy_login.yml` does this.

**Registering with an already-running Caddy LXC**: a standalone
`tasks_from: register_service` entrypoint (bypassing the role's normal
`install`/`configure`) that only templates one `conf.d/<service>.caddy`
snippet and reloads — the pattern any app repo should use to attach
itself to a shared, independently-managed Caddy instance. Requires the
target Caddy's live `Caddyfile` to already `import conf.d/*.caddy`;
verify this once per Caddy LXC, not per consuming app.

**Verifying a live run, not just its exit code**: check `pct list`
directly on Proxmox; for UniFi, query the controller's own record for the
client rather than trusting the task's `changed`/`ok` status alone
(`GET .../stat/sta`, matched by MAC, checking `use_fixedip`/`fixed_ip`
in the response — the same fields the UniFi UI's client-details panel
reads) and confirm DNS resolution (`dig <name> @<router-ip>`); for the
app itself, an actual HTTP request against its public hostname, not just
`docker compose up` succeeding.

**Debugging without materializing secrets**: never decrypt a vault or
print an API key to check it's right. Let Ansible use it internally —
write a throwaway task (delete it afterward) that makes the real API call
and `debug`s only the non-secret response fields you need (site
`internalReference`, a client's `use_fixedip`, an HTTP status code). This
is how the `NoSiteContext` root cause above was found, without ever
exposing `unifi_api_key` or the legacy account's password.

**Iterating on a `requirements.yml`-installed role during development**:
don't loop through `ansible-galaxy role install --force` for every edit —
it re-clones from the pinned commit on GitHub, so nothing changes until
you've committed *and pushed*. Instead edit the role's own sibling repo
checkout directly and `rsync -a --exclude=.git --exclude=.ansible` it
into the consumer's `roles/<name>/` for fast iteration. Once green:
commit and push the role repo, bump `requirements.yml`'s pinned commit to
that SHA, then do one clean `ansible-galaxy role install --force` and a
final full run — that run is what actually proves the pinned state
works, not the iterated-on copy.

## Security posture

- **SSH-key-only authentication.** No password-based SSH access.
  Interactive managed users get `password_lock` set — the account exists
  and can `sudo`, but cannot log in with a password.
- **Two-tier account model** (`ubuntu_manage_user`'s established pattern —
  apply it to any role that provisions accounts):
  - *Interactive users*: require `ssh_public_key`; password login always
    locked.
  - *Service accounts*: `system: true`, `nologin` shell, no SSH key; the
    `ssh.yml`-equivalent task is skipped entirely for these.
- **Sudoers validated via `visudo`.** Sudoers entries are templated and
  always validated (`visudo -cf`) before being applied. `sudo: true`
  writes `/etc/sudoers.d/<name>`; `sudo: false` (the default) actively
  *removes* any existing drop-in on every run — sudo access is never
  silently left in place from a previous run.
  - Baseline: `NOPASSWD: ALL` for interactive admin users.
  - Hardened (agreed direction for the Ansible **connection account**
    itself — the account Ansible uses to log in — not yet fully enforced
    in `ansible-control`, which currently still allows a login shell):
    `nologin`/no-interactive-shell plus sudoers scoped to specific
    commands rather than `NOPASSWD: ALL`, to limit blast radius if that
    account's key is ever compromised. Worth closing that gap.
- **Secrets never committed.** Real inventory (`inventory.ini`), real
  account lists (`vars/users.yml`, `vars/service_accounts.yml`), and vault
  password files (`.vault_pass`, `.vault_password`, `vault_password_file`)
  are gitignored; only `*.example` templates are committed. Ansible Vault
  encrypts any secret values that must be stored (see
  `ansible-proxmox-caddy/group_vars/all/vault.yml` for the pattern).
- **Secret scanning.** `gitleaks` runs both as a pre-commit hook (against
  a per-repo `.gitleaks.toml`, with `--redact`) and as CI via the shared
  `gitleaks.yml` reusable workflow, uploading SARIF to the repo's Security
  tab.

## CI: centralized, not duplicated

CI is **not** hand-written per repo. `TobyAnscombe/github-actions` holds
reusable `workflow_call` workflows, and every project's own
`.github/workflows/ci.yml` is a short caller:

```yaml
jobs:
  secret-scan:
    uses: TobyAnscombe/github-actions/.github/workflows/gitleaks.yml@main
  ansible-lint:
    uses: TobyAnscombe/github-actions/.github/workflows/ansible-ci.yml@main
  claude-file-check:
    uses: TobyAnscombe/github-actions/.github/workflows/claude-file-check.yml@main
  protect-main:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    needs: [secret-scan, ansible-lint, claude-file-check]
    uses: TobyAnscombe/github-actions/.github/workflows/branch-protection.yml@main
    secrets:
      token: ${{ secrets.BRANCH_PROTECTION_TOKEN }}
```

New shared CI logic goes into `github-actions`, versioned on `main`, not
copy-pasted into each project. Current reusable workflows: `gitleaks.yml`,
`ansible-ci.yml` (runs `ansible-lint`, installing `requirements.yml` first
if present), `claude-file-check.yml`, `branch-protection.yml`, plus
`terraform-ci.yml` and `powershell-lint.yml` for non-Ansible stacks.

`ansible-lint` runs with a per-repo `.ansible-lint` (typically
`profile: basic`, plus a `skip_list` for justified exceptions — comment
*why* next to each skip) and a pinned version installed via pip in CI.

## Claude Code working files: local only, CI-enforced

`CLAUDE.md`, `tasks/*.md` (including `tasks/lessons.md`), and `.claude/`
are working files for AI-assisted development, not project deliverables.
They exist in every repo's working copy but are **always gitignored**,
and `claude-file-check.yml` fails CI if any of them are ever accidentally
tracked. When scaffolding a new project, add these three lines to
`.gitignore` from the start:

```
CLAUDE.md
tasks/*.md
.claude/
```

## Idempotency & dry-run safety

- Every task must be safe to re-run without side effects, **and** safe to
  run under `ansible-playbook --check`. `ubuntu_manage_user`'s `ssh.yml`
  passes an explicit `path:` to `ansible.posix.authorized_key` (derived
  from a variable rather than looked up on the live target) specifically
  because the dynamic lookup fails in `--check` mode when the account
  doesn't exist yet — prefer variables over runtime facts wherever a task
  needs to work correctly in check mode.
- Idempotency includes actively correcting drift, not just avoiding
  errors on re-run — the sudoers removal behaviour above is the model:
  if the desired state doesn't include something, the role removes it,
  it doesn't just skip adding it.

## Applying these standards to a new project

1. Scaffold the repository layout above, including the CI caller
   workflow and the `.ansible-lint` / `.pre-commit-config.yaml` /
   `.gitleaks.toml` trio.
2. Decide vendored-role vs. own-repo-role up front based on whether this
   is genuinely single-project.
3. Write `vars/` (and their `.gitignore` entries) before writing tasks
   against them.
4. Split `tasks/main.yml` from the start using whichever task-file set
   matches the role's *kind* (user-management / service-install /
   maintenance) — don't force the user-management five-file set onto a
   role that's actually installing a service.
5. For any account the automation itself connects with, default to the
   hardened service-account pattern (no login shell, scoped sudoers), not
   the baseline interactive pattern.
6. Every role must ship a `validate.yml` and must tolerate `--check` mode.

## Known gaps to reconcile (as of this review)

These are inconsistencies found across the existing repos, not new rules
— listed here so they don't get silently copied into the next project:

- `ansible-control`'s bootstrap role still allows a normal login shell for
  the Ansible connection account; the hardened `nologin` pattern hasn't
  been applied there yet.
- `ubuntu-patching`'s `ubuntu_patch` role has no `validate.yml`.
- Top-level playbook naming is split between `playbook.yml` (majority) and
  `site.yml` (`ubuntu-patching`, `htb-ssh-key-rotation`) — pick one going
  forward; `playbook.yml` is the more common choice today.
- `renovate.json` (automated dependency-update PRs) is only in
  `ansible-cyberark-ssh-provisioning` and one role under
  `htb-ssh-key-rotation` — not yet a repo-wide standard, but worth
  considering for all `requirements.yml`-based repos given how easy it is
  to let pinned role/collection versions go stale.
- `ansible-control` has a stray literal directory named `{.github` at its
  root (left over from an mkdir that didn't get brace-expanded) — a
  one-off cleanup item, unrelated to these standards.
- `ansible-role-lxc-provision`, `ansible-role-unifi`, and
  `ansible-role-caddy` pin in `requirements.yml` by raw commit SHA
  (`version: ef40080b3...`) rather than a semver tag like
  `ubuntu_manage_user`'s `v1.1.10`. Acceptable while these are still
  under active, single-consumer development, but they should move to
  tagged releases once a second project consumes them or they stabilize —
  consistent with the "promote to own repo" guidance above.

## Scope

This repo documents conventions only — it is intentionally not a
distributable Ansible role or collection. Individual projects implement
these conventions themselves; `ubuntu-user-creation` is the worked example
to copy from, and `TobyAnscombe/github-actions` is the shared CI layer
every project should call into rather than duplicate.
