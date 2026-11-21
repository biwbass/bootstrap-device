# CLAUDE.md

Ansible bootstrap for a homelab kubeadm cluster. See `README.md` for what
each playbook does. The GitOps side (Argo CD apps) is in
`jordanthomas761/homelab-infra`.

## This repo is public

- **No IP addresses or subnets** in files, commit messages, PR bodies, issues
  or comments. Name hosts (`gpu.k8s.internal`) or roles ("the cluster
  network", "the Proxmox host") instead. Before committing or posting, grep
  the diff and the text for `([0-9]{1,3}\.){3}[0-9]{1,3}`.
- **No secrets, tokens, emails or personal paths.** Secrets go through Ansible
  Vault (`inventory/vault/`, passed explicitly) or SealedSecrets in
  homelab-infra.
- If one slips into a pushed commit, rewrite the branch before the PR merges,
  and say so.

## Access to machines

- **Read-only by default.** Ask before anything that changes a machine or the
  cluster: Ansible runs, `kubectl apply`/`patch`/`cordon`, restarts, package
  installs.
- **Read-only tools that need no permission:**
  - `~/.local/bin/ssh-ro <cp-ro|gpio-ro|ml-ro> <cmd>`: a restricted user with
    no sudo.
  - `~/.local/bin/kubectl-ro <args>`: a view-only ServiceAccount with no
    Secrets.

  Both are set up by `playbooks/claude-readonly-access.yml`.
- **Plain `ssh`, `scp`, `kubectl` (admin kubeconfig) and `ansible-playbook`**
  need the user's OK every time, even for read-only commands. If the
  read-only path can't answer something, give the user the command to run.
- **The Proxmox host is out of reach.** Its changes (vfio binding, GRUB,
  VM 201 config, snapshots) are done by hand by the user and are not in this
  repo. Write them up as steps for the user.

## GitHub

- **Ask before filing issues or posting comments,** unless the task says
  otherwise.
- **Never merge a PR** without the user's go-ahead.
- **Branch off up-to-date `main`.** Merged PRs land on GitHub first, so run
  `git fetch origin` and work from `origin/main`.

## Worktrees

Sessions often run in `.claude/worktrees/<name>`. The shell's working
directory can reset to the main checkout between commands, which silently
puts edits on `main`.

- Use `git -C <worktree>` and absolute worktree paths for every edit.
- Check `git -C <worktree> branch --show-current` before committing.
- Never use a bare `git stash`. The stash is shared across worktrees.

## Conventions

- **Docs:** usage and operations go in README sections. Design decisions go
  in `docs/adr/NNNN-title.md`, linked from the README.
- **Idempotency:** a second `main.yml` run on a host must report `changed=0`.
  Check with `--check --diff` first.
- **Version pins** live in `inventory/group_vars/all.yml`. Bump Kubernetes one
  minor at a time (README "Kubernetes Version Upgrades").

## Known gotchas

- **Node names.** The kubelet registers under `uname -n`, not
  `hostname -f`. Cloud-init VMs on Proxmox get a short `uname -n` unless the
  FQDN drop-in is set. Node names can't change after a join.
- **The control plane runs Ubuntu 26.04:**
  - `/usr/bin/test` is uutils, and its `-r` ignores ACLs. Use bash's `[ -r ]`.
  - `sudo` is sudo-rs.
  - `ansible_managed` is only defined in the `template` module.
  - `kubectl auth can-i` needs `--subresource=` for subresources.
- **The GPU worker:**
  - **Tainted at registration** with `dedicated=gpu:NoSchedule`.
  - **No Longhorn,** deliberately: it is meant to sleep. Every GPU workload
    sets `storageClassName` explicitly, because `longhorn` is the default.
  - **Assume nothing about it staying up.** See
    `docs/adr/0001-gpu-worker-sleep-wake.md` once that merges.
