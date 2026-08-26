# Day 3 — Version Control (Git & GitHub)

**Phase 1: Foundations & Linux Mastery** · Day 3 of 20

| | |
|---|---|
| **Lectures** | `Day 10_GitHub and its commands` (24 slides), `Day 11_Upload files on GitHub` (16 slides) |
| **Lab host** | Ubuntu 24.04 LTS on AWS EC2 — `m7i-flex.large`, `us-east-1a` |
| **Git version** | 2.43.0 |
| **Auth method** | SSH key (Ed25519) — not a password, not a token |

Both lectures were theory again, and the second is roughly 80% a repeat of the first — the same "Setting Up Git", "How to Push", "git clone", "Summary of Git Commands", "Branching Strategy" and "PR Workflow" slides appear in both.

Days 1 and 2 were committed through the GitHub web UI: write in Notepad, drag into the browser, click Commit. Today that stopped. Everything below was run from a terminal.

**The part worth reading:** the lab hit three real failures — a paste-blocked credential prompt, a rejected push, and a mid-rebase detached HEAD. Those are in §5, §9 and §10. The lectures cover none of them, and they are what actually happens.

---

## Contents

1. [Git vs GitHub](#1--git-vs-github)
2. [Setting up identity](#2--setting-up-identity)
3. [Clone](#3--clone)
4. [The staging area](#4--the-staging-area)
5. [Failure 1 — authentication](#5--failure-1--authentication)
6. [Push](#6--push)
7. [Branching](#7--branching)
8. [Pull request and merge](#8--pull-request-and-merge)
9. [Merge conflicts](#9--merge-conflicts)
10. [Failure 2 — rejected push, and recovery](#10--failure-2--rejected-push-and-recovery)
11. [Command reference](#11--command-reference)
12. [Key takeaways](#12--key-takeaways)

---

## 1 · Git vs GitHub

*Covers deck 1 slides 5–9*

| | Git | GitHub |
|---|---|---|
| What it is | A program on your machine | A website |
| Needs internet | No | Yes |
| Stores | The `.git` folder in your project | A copy of that, remotely |
| Can you replace it | Not really | Yes — GitLab, Bitbucket, Gitea |

Git is the version control system. GitHub is one hosting service for it. Every command in this document is **git**; GitHub only appears when something has to travel over the network.

**Key terms** (deck 1 slide 7):

| Term | Meaning |
|---|---|
| Repository | A folder Git is tracking |
| Commit | A saved snapshot of the whole project |
| Push | Send local commits to GitHub |
| Pull | Fetch GitHub's commits and merge them in |
| Branch | An independent line of work |
| Merge | Combine two branches |

---

## 2 · Setting up identity

*Covers deck 1 slide 10*

```bash
git --version
git config --global user.name "Fahad Ali Seemab"
git config --global user.email "fahadaliseemab@users.noreply.github.com"
git config --global init.defaultBranch main
git config --global --list
```

**Output:**

```
git version 2.43.0
user.name=Fahad Ali Seemab
user.email=fahadaliseemab@users.noreply.github.com
init.defaultbranch=main
```

Git was already installed — Ubuntu server images ship it.

**Why the `users.noreply.github.com` address.** Commit metadata is permanent and public in a public repo. Committing with a real personal address publishes it forever to anyone who runs `git log`. GitHub issues a noreply alias that still links commits to your profile and still counts toward the contribution graph, with nothing to harvest.

The email is how GitHub attributes a commit — not the SSH key, not the username. Get it wrong and commits show up as an unknown author.

`init.defaultBranch main` avoids git's `master` default and the branch-name mismatch that follows.

---

## 3 · Clone

*Covers deck 1 slide 13*

```bash
git clone git@github.com:fahadaliseemab/devops-20-day-journey.git
cd devops-20-day-journey
ls -la
git log --oneline
git remote -v
git status
```

**Output:**

```
Cloning into 'devops-20-day-journey'...
remote: Enumerating objects: 44, done.
Receiving objects: 100% (44/44), 32.08 KiB | 6.42 MiB/s, done.
Resolving deltas: 100% (11/11), done.
```

```
drwxrwxr-x  .git
drwxrwxr-x  Day-01-Computers-Servers-Operating-Systems
drwxrwxr-x  Day-02-Networking-Security-Encryption
-rw-rw-r--  README.md
```

```
ae7b4f5 (HEAD -> main, origin/main, origin/HEAD) Update README.md
87fee27 Fix repo URL after username change, update checklist
a2e90b5 Update README.md
59adc17 Mark Day 2 complete and fix folder name
afe4f46 Create README.md
e4b1200 Update README.md
f86aaa8 improve Day 1 Linux commands
dbe3d50 docs: format Day 1 Linux commands
4025390 Day 1 Linux commands
d623a41 Create README.md
d0b9a5d docs: update DevOps journey README
95b8003 Initial commit
```

```
origin  git@github.com:fahadaliseemab/devops-20-day-journey.git (fetch)
origin  git@github.com:fahadaliseemab/devops-20-day-journey.git (push)

On branch main
Your branch is up to date with 'origin/main'.
nothing to commit, working tree clean
```

**Twelve commits.** Every "Commit changes" click in the browser over Days 1 and 2 was a real git commit — the web UI was a front-end for git the whole time. Clone retrieved all of it, not just the current files.

**`clone` is not `download`.** A ZIP gives you the files. Clone gives you the files *plus* `.git` — the complete history, every branch, and the remote wired up. That's why `git log` and `git remote -v` work immediately.

**Three pointers on one commit** is worth pausing on:

```
ae7b4f5 (HEAD -> main, origin/main, origin/HEAD)
         │        │            │
         │        │            └─ the remote's default branch
         │        └─ where GitHub's main is
         └─ where I am right now
```

All three agree, so the repo is in sync. When they drift apart, the log shows exactly how far.

---

## 4 · The staging area

*Covers deck 1 slide 11*

```bash
mkdir -p Day-03-Git-GitHub
echo "# Day 3 — Version Control (Git & GitHub)" > Day-03-Git-GitHub/README.md
git status                                        # before add
git add Day-03-Git-GitHub/README.md
git status                                        # after add
git commit -m "Day 3: start Git and GitHub notes"
git log --oneline -3
```

**Before `git add`:**

```
Untracked files:
        Day-03-Git-GitHub/

nothing added to commit but untracked files present
```

**After `git add`:**

```
Changes to be committed:
        new file:   Day-03-Git-GitHub/README.md
```

**Commit:**

```
[main 4f6990a] Day 3: start Git and GitHub notes
 1 file changed, 1 insertion(+)
 create mode 100644 Day-03-Git-GitHub/README.md
```

Nothing on disk changed between those two `git status` runs. `git add` moved the file from *untracked* to *staged*. That intermediate space is the **staging area**, and neither lecture mentions it.

```
working directory  ──git add──▶  staging area  ──git commit──▶  repository
   (your edits)                   (what's next)                  (history)
```

**Why it exists:** you fix a bug and a typo in the same session. Stage and commit them separately and you get two clean, revertible commits. Without staging you'd get one commit called "stuff".

### The commit is local only

```
4f6990a (HEAD -> main)                  ← my machine
ae7b4f5 (origin/main, origin/HEAD)      ← GitHub, one behind
87fee27 Fix repo URL after username change
```

The pointers have separated. The commit exists nowhere but this server — GitHub has no idea it happened. **Commit ≠ push.** Commit writes to `.git` locally; push is a separate network operation. Losing the machine at this moment loses the commit.

---

## 5 · Failure 1 — authentication

*Deck 1 slide 14 says: "Write your credentials of GitHub — Your email ID and Password"*

**That instruction no longer works.** GitHub removed password authentication for git operations in August 2021. Following the slide produces:

```
remote: Support for password authentication was removed.
```

The two current options are a **Personal Access Token** (used as the password) or an **SSH key**.

### What actually went wrong

Generated a token, then hit the prompt:

```
Username for 'https://github.com': fahadaliseemab
Password for 'https://fahadaliseemab@github.com':
```

The token wouldn't paste. `Ctrl+V`, right-click, `Shift+Insert`, middle-click — none of them entered anything at that prompt in this terminal. Two push attempts, both dead.

### The fix: stop needing to paste

The blocker was pasting *into* the terminal. Copying *out* of it works fine. So the direction was reversed — generate the key on the server, copy the public half out, paste it into the browser:

```bash
ssh-keygen -t ed25519 -C "ec2-devops-journey" -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub
```

```
The key fingerprint is:
SHA256:ypW4Kj4kynwLn3nYIU6epVs1BWzyPbBj8LCEUSy+OMc ec2-devops-journey

ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEnk4Wzuh7yGTItU2+sc+RLQFO/o9dYxfgMqY0m0/znm ec2-devops-journey
```

The `.pub` line went into GitHub → Settings → SSH and GPG keys → New SSH key.

```bash
ssh -T git@github.com
git remote set-url origin git@github.com:fahadaliseemab/devops-20-day-journey.git
```

```
The authenticity of host 'github.com (140.82.114.3)' can't be established.
ED25519 key fingerprint is SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes

Hi fahadaliseemab! You've successfully authenticated, but GitHub does not provide shell access.
```

**This is Day 2's asymmetric encryption doing real work.** The private key never left the server; GitHub only ever saw the public half. Nothing secret crossed the network, and there is no password to paste, leak, or expire.

**The host fingerprint prompt is the same identity check as a TLS certificate.** `SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU` matches GitHub's published ED25519 fingerprint, so `yes` was correct — but typing `yes` without checking is the SSH equivalent of `curl -k` from Day 2. Accept an attacker's fingerprint once and it's cached in `~/.ssh/known_hosts` permanently.

### Token vs SSH key

| | Personal Access Token | SSH key |
|---|---|---|
| Expires | Yes — 90 days here | No |
| Secret sent over the wire | Yes, every push | Never |
| Typed per push | Yes, unless cached | Never |
| Leaks if pasted somewhere wrong | Yes | Public half is harmless |
| Right for | CI, scripts, short-lived jobs | A machine you'll keep using |

For a long-lived server: SSH key.

---

## 6 · Push

*Covers deck 1 slide 12*

```bash
git push origin main
```

```
Enumerating objects: 5, done.
Writing objects: 100% (4/4), 405 bytes | 405.00 KiB/s, done.
To github.com:fahadaliseemab/devops-20-day-journey.git
   ae7b4f5..4f6990a  main -> main
```

`ae7b4f5..4f6990a` is a range: "the remote moved from this commit to that one." Push transmits the commits between them — not the files, the commits. Note the size: **405 bytes**, because git sent only the delta.

The pointers are back together:

```
4f6990a (HEAD -> main, origin/main)
```

Slide 12's `git remote add origin ...` wasn't needed here. `git clone` set `origin` automatically. That slide is for the other direction — a folder that exists locally first, then gets pushed to a new empty repo.

---

## 7 · Branching

*Covers deck 1 slides 15–19*

```bash
git branch                        # before
git checkout -b feature/day-03-notes
git branch                        # after
echo "Lab run on Ubuntu 24.04 EC2. Git 2.43.0, SSH key auth." >> Day-03-Git-GitHub/README.md
git status
git diff
git add .
git commit -m "Add lab environment note"
git push -u origin feature/day-03-notes
```

**Before / after:**

```
* main
Switched to a new branch 'feature/day-03-notes'
* feature/day-03-notes
  main
```

The `*` marks HEAD. Two branches now exist; you're on the new one.

### `git status` vs `git diff`

```
Changes not staged for commit:
        modified:   Day-03-Git-GitHub/README.md
```

```
diff --git a/Day-03-Git-GitHub/README.md b/Day-03-Git-GitHub/README.md
@@ -1 +1,3 @@
 # Day 3 — Version Control (Git & GitHub)
+
+Lab run on Ubuntu 24.04 EC2. Git 2.43.0, SSH key auth.
```

**Status says which file. Diff says what changed.** `@@ -1 +1,3 @@` means the file went from 1 line to 3, starting at line 1. `+` = added, `-` = removed. This is what a code reviewer reads, and it's what a Pull Request renders.

Run `git diff` before every commit. It's the last chance to catch a debug line or a pasted credential.

### Push output

```
remote: Create a pull request for 'feature/day-03-notes' on GitHub by visiting:
remote:      https://github.com/fahadaliseemab/devops-20-day-journey/pull/new/feature/day-03-notes

 * [new branch]      feature/day-03-notes -> feature/day-03-notes
branch 'feature/day-03-notes' set up to track 'origin/feature/day-03-notes'.
```

`-u` set up tracking, so plain `git push` and `git pull` work on this branch from now on without naming the remote. Without it you'd retype `origin feature/day-03-notes` every time.

### Branch naming (deck 1 slide 16)

| Pattern | Purpose |
|---|---|
| `main` | Production. Protected. Never committed to directly on a real team |
| `dev` / `develop` | Integration branch |
| `feature/...` | New work |
| `bugfix/...` | A fix targeting dev |
| `hotfix/...` | An urgent fix straight off main |
| `release/...` | Release preparation |

The prefixes aren't cosmetic — CI systems and branch protection rules match on them.

---

## 8 · Pull request and merge

*Covers deck 1 slide 20*

Opened the PR, checked **Files changed** (the same diff as §7, rendered), merged, deleted the branch.

```
Merged — fahadaliseemab merged 1 commit into main from feature/day-03-notes
merged commit fd73068 into main
deleted the feature/day-03-notes branch
```

Then brought it down locally:

```bash
git checkout main
git pull
git log --oneline --graph -4
```

```
Updating 4f6990a..fd73068
Fast-forward
 Day-03-Git-GitHub/README.md | 2 ++
 1 file changed, 2 insertions(+)
```

```
*   fd73068 (HEAD -> main, origin/main) Merge pull request #1 from fahadaliseemab/feature/day-03-notes
|\
| * f495187 Add lab environment note
|/
* 4f6990a Day 3: start Git and GitHub notes
```

`fd73068` is a **merge commit** — it has two parents, which is what `|\` and `|/` draw. GitHub's default "Create a merge commit" preserves the fact that a branch existed.

Locally the pull was a **fast-forward**: main had no commits of its own, so git slid the pointer forward with no merge needed.

### Cleanup

```bash
git remote prune origin
git branch -d feature/day-03-notes
git branch -a
```

```
 * [pruned] origin/feature/day-03-notes
Deleted branch feature/day-03-notes (was f495187).
* main
  remotes/origin/HEAD -> origin/main
  remotes/origin/main
```

Deleting a branch on GitHub doesn't remove your local copy or your cached remote-tracking ref. `git remote prune origin` drops refs to branches that no longer exist upstream; `git branch -d` removes the local one. `-d` refuses if the branch isn't merged — a safety net. `-D` forces it.

---

## 9 · Merge conflicts

*Not in either lecture. This is the part that actually causes trouble.*

Created one deliberately — two branches editing the same line:

```bash
git checkout -b test-conflict
echo "Line from branch" >> Day-03-Git-GitHub/README.md
git commit -am "Branch edit"

git checkout main
echo "Line from main" >> Day-03-Git-GitHub/README.md
git commit -am "Main edit"

git merge test-conflict
```

```
Auto-merging Day-03-Git-GitHub/README.md
CONFLICT (content): Merge conflict in Day-03-Git-GitHub/README.md
Automatic merge failed; fix conflicts and then commit the result.
```

**Git usually merges without help.** Different files, or different parts of the same file, merge silently. A conflict only happens when both sides changed *the same region*, because then there is no correct answer and git refuses to guess.

### What git writes into the file

```
# Day 3 — Version Control (Git & GitHub)
<<<<<<< HEAD
Line from main
=======

Lab run on Ubuntu 24.04 EC2. Git 2.43.0, SSH key auth.
Line from branch
>>>>>>> test-conflict
```

| Marker | Meaning |
|---|---|
| `<<<<<<< HEAD` | start of **your** version (the branch you're on) |
| `=======` | divider |
| `>>>>>>> test-conflict` | end of the **incoming** version |

`git status` during a conflict:

```
You have unmerged paths.
  (fix conflicts and run "git commit")
  (use "git merge --abort" to abort the merge)

Unmerged paths:
        both modified:   Day-03-Git-GitHub/README.md
```

**`both modified`** is the signal, and **`git merge --abort`** is the escape hatch — it returns everything to how it was before the merge started. Worth knowing before you need it.

### Resolving

Resolution means editing the file into whatever it should actually be, then telling git you're done. Keep your side, keep theirs, keep both, or write something new — git doesn't care, it just needs the markers gone.

```bash
sed -i '/^<<<<<<< /d;/^=======$/d;/^>>>>>>> /d' Day-03-Git-GitHub/README.md
git add Day-03-Git-GitHub/README.md
git commit -m "Resolve conflict: keep both lines"
git log --oneline --graph -4
```

```
*   6f37207 (HEAD -> main) Resolve conflict: keep both lines
|\
| * 40d2ca5 (test-conflict) Branch edit
* | 59c6860 Main edit
|/
```

**That's the branching diagram from slide 18, generated from real history.** The line splits at `|\`, runs in parallel through both edits, and rejoins at `|/`. Two parents, one merge commit.

**Deleting the markers is not the same as resolving.** Here it was the intended answer. In real code, blindly stripping markers produces a file that contains both versions of a function and doesn't compile. Read the conflict, decide, then delete.

---

## 10 · Failure 2 — rejected push, and recovery

Cleaned up the test lines, committed, pushed:

```bash
git commit -am "Remove conflict test lines"
git branch -D test-conflict
git push
```

```
 ! [rejected]        main -> main (fetch first)
error: failed to push some refs to 'github.com:fahadaliseemab/...'
hint: Updates were rejected because the remote contains work that you do not
hint: have locally. This is usually caused by another repository pushing to
hint: the same ref. If you want to integrate the remote changes, use
hint: 'git pull' before pushing again.
```

**This is git protecting history.** The PR merge commit `fd73068` was created *on GitHub* and hadn't been pulled. Local and remote had both moved on independently — diverged. Accepting the push would have discarded GitHub's commit, so git refused.

It is also the single most common git error in a team: someone else pushed while you were working.

### The recovery went wrong first

```bash
git pull --rebase
```

```
CONFLICT (content): Merge conflict in Day-03-Git-GitHub/README.md
error: could not apply 59c6860... Main edit
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git rebase --continue".
```

```
fatal: You are not currently on a branch.
To push the history leading to the current (detached HEAD) state now, use
    git push origin HEAD:<name-of-remote-branch>
```

`--rebase` replays local commits one at a time on top of the remote's. The first replay conflicted, so the rebase paused mid-flight — leaving a **detached HEAD**, not on any branch. `git push` then had nothing to push *to*.

Three exits from a paused rebase:

| Command | Effect |
|---|---|
| `git rebase --continue` | after resolving, carry on to the next commit |
| `git rebase --skip` | discard the conflicting commit, keep going |
| `git rebase --abort` | undo the whole rebase, back to before it started |

### The recovery

The four local commits were all conflict-test scaffolding. The content worth keeping — the lab environment note — was already on GitHub via the PR. So the honest move was to throw the local mess away:

```bash
git rebase --abort
git checkout main
git reset --hard origin/main
git log --oneline -3
git status
```

```
Your branch and 'origin/main' have diverged,
and have 4 and 1 different commits each, respectively.
HEAD is now at fd73068 Merge pull request #1 from ...

fd73068 (HEAD -> main, origin/main, origin/HEAD) Merge pull request #1
f495187 Add lab environment note
4f6990a Day 3: start Git and GitHub notes

On branch main
Your branch is up to date with 'origin/main'.
nothing to commit, working tree clean
```

Note git spelling out the divergence — *4 and 1 different commits each* — before the reset, then clean agreement after.

> **`git reset --hard` permanently destroys uncommitted work and local commits.** It was correct here because the discarded commits were disposable and everything valuable was already on the remote. Verify that's true — `git log origin/main` — before running it. There is no undo prompt.

---

## 11 · Command reference

*Deck 1 slide 14 lists six commands. These are the ones the lab actually needed.*

### Setup — once per machine

| Command | Purpose |
|---|---|
| `git --version` | Check git is installed |
| `git config --global user.name "..."` | Name on your commits |
| `git config --global user.email "..."` | Email — how GitHub attributes commits |
| `git config --global init.defaultBranch main` | Avoid the `master` default |
| `git config --global --list` | Show current config |
| `ssh-keygen -t ed25519 -C "..."` | Generate an auth keypair |
| `ssh -T git@github.com` | Test GitHub SSH auth |

### Daily

| Command | Purpose |
|---|---|
| `git clone <url>` | Copy a remote repo, history included |
| `git status` | Which files changed |
| `git diff` | What changed inside them |
| `git diff --staged` | What's staged, before committing |
| `git add <file>` | Stage one file |
| `git add .` | Stage everything |
| `git commit -m "..."` | Snapshot staged changes |
| `git commit -am "..."` | Stage tracked files and commit in one step |
| `git push` | Send commits up |
| `git pull` | Fetch and merge remote commits |
| `git log --oneline` | Compact history |
| `git log --oneline --graph` | History with branch structure |

### Branching

| Command | Purpose |
|---|---|
| `git branch` | List local branches; `*` = current |
| `git branch -a` | Include remote-tracking branches |
| `git checkout -b <name>` | Create and switch |
| `git checkout <name>` | Switch |
| `git push -u origin <name>` | Push and set tracking |
| `git merge <name>` | Merge a branch in |
| `git branch -d <name>` | Delete if merged |
| `git branch -D <name>` | Force delete |
| `git remote prune origin` | Drop refs to deleted remote branches |

### When it breaks

| Command | Purpose |
|---|---|
| `git merge --abort` | Cancel a conflicted merge |
| `git rebase --abort` | Cancel a paused rebase |
| `git rebase --continue` | Resume after resolving |
| `git restore <file>` | Discard changes to a file |
| `git restore --staged <file>` | Unstage, keep the edit |
| `git reset --hard origin/main` | Force local to match remote — **destructive** |
| `git remote -v` | Check where origin points |

---

## 12 · Key takeaways

1. **Commit ≠ push.** Commit writes to `.git` on your machine. Push is a separate network step. The log shows the gap: `HEAD -> main` ahead of `origin/main`.
2. **The staging area is a feature, not a nuisance.** It's how you split unrelated edits into separate, revertible commits.
3. **`git status` says which file; `git diff` says what changed.** Run diff before every commit.
4. **`clone` is not a download.** It brings `.git` — full history, all branches, remote wired up.
5. **The lectures' "email and password" is obsolete** since 2021. It's a token or an SSH key, and for a long-lived server an SSH key is better: nothing secret crosses the wire, and nothing expires.
6. **The SSH host-fingerprint prompt is an identity check**, exactly like a TLS certificate. Typing `yes` unread is Day 2's `curl -k`.
7. **A rejected push means diverged history.** Git is refusing to destroy commits you don't have. `git pull` first — never reach for `--force` to make the message go away.
8. **Conflicts only occur in the same region of the same file.** Git merges everything else silently.
9. **Learn the abort commands before you need them.** `git merge --abort` and `git rebase --abort` return you to safety; a paused rebase leaves you on a detached HEAD with confusing errors.
10. **A commit is safe once it's on the remote.** That's what made `git reset --hard` an acceptable choice here — and what makes it dangerous anywhere else.
11. **Deleting conflict markers is not resolving a conflict.** Read both sides and decide.
12. **The web UI was git all along.** Twelve commits from Days 1–2 came down in the clone. The browser was a wrapper.

---

## Deliverables

- [x] Watched `Day 10_GitHub and its commands`
- [x] Watched `Day 11_Upload files on GitHub`
- [x] Configured git identity with a noreply email
- [x] Set up SSH key authentication after tokens failed
- [x] Cloned the repo to the EC2 instance
- [x] Staged, committed and pushed from the terminal
- [x] Created a branch, opened PR #1, merged, deleted the branch
- [x] Pulled the merge commit down and cleaned up stale refs
- [x] Created and resolved a merge conflict deliberately
- [x] Recovered from a rejected push and a detached HEAD
- [ ] Solve the 10 MCQs from the lecture description
- [ ] Post the Day 3 LinkedIn update, tagging Mise Academy and the instructor

---

*Day 3 of 20 · [devops-20-day-journey](https://github.com/fahadaliseemab/devops-20-day-journey)*
