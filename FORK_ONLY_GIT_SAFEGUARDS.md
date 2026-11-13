# Fork-only Git safeguards

## Purpose

This checkout is for private prototype work. **Do not push to
`NVIDIA-NeMo/Speech` (or its historical `NVIDIA-NeMo/NeMo` URL), and do not
open an upstream pull request.** Push only to Jennifer Fox's fork:

```text
git@github.com:jennyfox-rad/NeMo-Speech.git
```

An agent must obtain explicit approval before *any* push. When asking, name the
full destination repository and branch. A fork branch push does **not** imply
permission to create a pull request, especially an upstream one.

## Apply in every fresh clone

Run these commands from the repository root. They are deliberately local to the
clone: they do not alter global Git configuration or the remote repository.

First, make sure the remotes have the intended roles. `origin` may be the
read-only upstream source, while `fork` is the only push destination.

```bash
git remote -v
git remote get-url fork
git remote get-url --push fork
```

The last two commands must print:

```text
git@github.com:jennyfox-rad/NeMo-Speech.git
```

If the `fork` remote does not exist, add it only with that exact URL:

```bash
git remote add fork git@github.com:jennyfox-rad/NeMo-Speech.git
```

Then install the local configuration guards:

```bash
git config --local remote.pushDefault fork
git config --local --unset-all remote.origin.pushurl || true
git config --local --add remote.origin.pushurl DISABLED_UPSTREAM_PUSH
```

These make an unqualified `git push` select `fork`, and make `git push origin`
fail before it can reach upstream. Fetching from `origin` continues to work.

## Install the pre-push hook

Git hooks are not committed, so this step is required in every fresh clone.
Create `.git/hooks/pre-push` with this exact content and mark it executable:

```sh
#!/bin/sh
# Prevent accidental pushes to the canonical NVIDIA-NeMo repository.

set -eu

remote_name=$1
remote_url=$2

case "$remote_name:$remote_url" in
  *NVIDIA-NeMo/*)
    printf '%s\n' "Blocked push to NVIDIA-NeMo upstream ($remote_name: $remote_url)." >&2
    printf '%s\n' "This checkout is guarded for fork-only pushes." >&2
    exit 1
    ;;
esac

exit 0
```

```bash
chmod 755 .git/hooks/pre-push
```

The hook is a second guard: it blocks an upstream URL even if someone bypasses
the remote name and runs an explicit command such as
`git push https://github.com/NVIDIA-NeMo/Speech.git ...`.

## Verify without pushing

Run all of the following after installation:

```bash
git config --local --get remote.pushDefault
git config --local --get-all remote.origin.pushurl
git remote get-url --push fork
test -x .git/hooks/pre-push
```

Expected values are respectively:

```text
fork
DISABLED_UPSTREAM_PUSH
git@github.com:jennyfox-rad/NeMo-Speech.git
```

Finally, simulate the hook directly. The first command must print `Blocked
push...` and return failure; the second must succeed silently.

```bash
if .git/hooks/pre-push origin git@github.com:NVIDIA-NeMo/NeMo.git; then
  echo "ERROR: upstream push was not blocked" >&2
  exit 1
fi
.git/hooks/pre-push fork git@github.com:jennyfox-rad/NeMo-Speech.git
```

## Before any approved push

1. Run `git status --short` and `git log --oneline origin/main..HEAD`.
2. State the exact command and destination, for example:
   `git push -u fork target-speaker-asr-prototype` to
   `jennyfox-rad/NeMo-Speech`.
3. Wait for explicit user approval.
4. Do not create a pull request unless the user separately identifies the
   target repository and base branch.

If upstream contribution is eventually intended, use a separate checkout and
remove these local safeguards there only after the user explicitly authorizes
that workflow.
