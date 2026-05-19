# Git Cheat Sheet

---

## 📦 Staging

| Command | Description |
|---|---|
| `git add <file>` | Stage a specific file |
| `git add .` | Stage all changed files |
| `git restore --staged <file>` | Unstage a specific file (keeps changes) |
| `git restore --staged .` | Unstage everything |

---

## ✅ Committing

| Command | Description |
|---|---|
| `git commit -m "message"` | Commit with a message |
| `git commit --amend -m "new msg"` | Edit the last commit message |
| `git commit --amend --no-edit` | Add staged changes to last commit (keep message) |

---

## 🚀 Push / Pull

| Command | Description |
|---|---|
| `git push origin <branch>` | Push to remote branch |
| `git push -u origin <branch>` | Push and set upstream |
| `git pull` | Pull and merge from remote |
| `git fetch origin` | Fetch without merging |

---

## 🌿 Branches

| Command | Description |
|---|---|
| `git branch` | List all local branches |
| `git branch <name>` | Create a new branch |
| `git checkout <branch>` | Switch to a branch |
| `git checkout -b <branch>` | Create and switch to new branch |
| `git branch -d <branch>` | Delete a local branch |
| `git branch -D <branch>` | Force delete a local branch |
| `git push origin --delete <branch>` | Delete a remote branch |
| `git merge <branch>` | Merge branch into current |

---

## 🔍 Status & Log

| Command | Description |
|---|---|
| `git status` | Show working tree status |
| `git log --oneline` | Compact commit history |
| `git log --oneline --graph --all` | Visual branch graph |
| `git diff` | Show unstaged changes |
| `git diff --staged` | Show staged changes |

---

## 🗂️ Pull / Get Specific File or Folder

| Command | Description |
|---|---|
| `git fetch origin` | Fetch remote changes |
| `git checkout origin/<branch> -- <file>` | Get a specific file from remote branch |
| `git checkout origin/<branch> -- <folder>/` | Get a specific folder from remote branch |
| `git restore <file>` | Discard local changes (revert to last commit) |

---

## ↩️ Undo / Reset

| Command | Description |
|---|---|
| `git revert <commit>` | Create a new commit that undoes a commit |
| `git reset --soft HEAD~1` | Undo last commit, keep changes staged |
| `git reset --mixed HEAD~1` | Undo last commit, keep changes unstaged |
| `git reset --hard HEAD~1` | Undo last commit and discard changes |
| `git clean -fd` | Remove untracked files and directories |

---

## 🧰 Stash

| Command | Description |
|---|---|
| `git stash` | Stash current changes |
| `git stash pop` | Apply and remove the latest stash |
| `git stash list` | List all stashes |
| `git stash drop` | Remove latest stash without applying |

---

## 🏷️ Tags

| Command | Description |
|---|---|
| `git tag` | List all tags |
| `git tag <name>` | Create a lightweight tag |
| `git tag -a <name> -m "msg"` | Create an annotated tag |
| `git push origin <tag>` | Push a tag to remote |
| `git push origin --tags` | Push all tags to remote |

---

## ⚙️ Config

| Command | Description |
|---|---|
| `git config --global user.name "Name"` | Set global username |
| `git config --global user.email "email"` | Set global email |
| `git config --list` | Show all config settings |

---

## 🔗 Remote

| Command | Description |
|---|---|
| `git remote -v` | List remotes |
| `git remote add origin <url>` | Add a remote |
| `git remote set-url origin <url>` | Change remote URL |

