# Setup lesson B — Git and your project history

## Goal

Allow 60–90 minutes. Install or verify Git, inspect changes, make commits, and
merge a branch. Practice in a separate repository before touching shared history.

## 1. Understand the workflow

Git tracks selected files over time. A commit records a snapshot with author,
message, identifier, and parent history. It does not automatically save every edit.

```text
Working files → git add → staging area → git commit → local history
                                                     ↓ optional git push
                                                remote repository
```

Git is the tool; GitHub is one optional hosting service. Local Git needs no
GitHub account. Git also does not back up PostgreSQL tables automatically.

## 2. Install Git on Mac

```sh
git --version
```

If a version prints, continue. If macOS offers developer tools, complete the
installation. You can request Apple's command-line tools with:

```sh
xcode-select --install
```

Then reopen the terminal and verify Git again. Do not reinstall tools that
already work. See [Git's official macOS installation options](https://git-scm.com/install/mac).

## 3. Create a separate practice repository

Use a new folder outside the data science project:

```sh
mkdir -p ~/projects/retail-git-practice
cd ~/projects/retail-git-practice
pwd
ls -a
```

If it already contains work or a `.git` directory, choose a different new folder.
Then run:

```sh
git init -b main
git config user.name "Your Name"
git config user.email "your-address@example.com"
git status
```

Replace the example identity with your chosen author details. These settings
apply to this repository only; they do not log you in anywhere. Author details
become visible if commits are shared, so choose your intended author email.
See [Git's first-time setup guide](https://git-scm.com/book/en/v2/Getting-Started-First-Time-Git-Setup)
for configuration scope. Expected: branch `main`, no commits yet.

## 4. Make the first commit

Open the practice folder in a separate VS Code window. Create `README.md`:

```markdown
# My retail project practice

I am learning to explain my data pipeline.
```

In the practice terminal:

```sh
git status
git add README.md
git diff --staged
git commit -m "Add project learning goal"
git log --oneline
```

Predict status before each command. `add` stages the file's current contents.
`diff --staged` previews the next commit. `commit` creates local history. Press
`q` if a diff/log opens in a pager.

## 5. Understand staging by experimenting

Append this line to the practice README and save:

```text
One raw row represents an invoice line.
```

```sh
git diff
git add README.md
git diff --staged
```

Now append another line **after staging**, and save:

```text
One model row represents a product on an observed selling day.
```

Run `git diff` and `git diff --staged` again. Explain why the second line is
unstaged. Include it with another `add`:

```sh
git add README.md
git diff --staged
git commit -m "Explain transaction and modeling grains"
```

After a first commit exists, `git restore --staged README.md` unstages that file
without deleting working edits. Practice this on your next edit, then stage again.
No destructive reset command is needed in this lesson.

## 6. Keep local data and credentials out of commits

Create a practice `.gitignore`:

```gitignore
.env
.venv/
__pycache__/
*.xlsx
```

Create a practice `.env` containing only this invented value:

```text
POSTGRES_PASSWORD=practice-only-not-a-real-secret
```

```sh
git status --short
git check-ignore .env
git add .gitignore
git diff --staged
git commit -m "Ignore local credentials and data files"
```

Expected: `.env` is ignored; `.gitignore` is tracked. `.env.example` can be tracked
when it contains placeholders. Ignore rules do not erase already tracked files
or earlier commits.

In the actual project, track code, SQL, documentation, requirements, and templates.
Keep the workbook, virtual environment, credentials, and database storage out.
Review generated artifacts individually: they can be large or contain customer
information. Stage explicit paths rather than blindly using `git add .`.

## 7. Create and merge a branch

Start from a clean status:

```sh
git status
git switch -c explain-baseline
```

Create `baseline.md`:

```markdown
A baseline predicts a product's earlier average units.
Regression must be compared with this simple reference.
```

```sh
git add baseline.md
git diff --staged
git commit -m "Document the baseline model"
git switch main
```

Inspect the folder: `baseline.md` is absent on main because its commit is on
the other branch. Bring the change into main:

```sh
git merge explain-baseline
git log --oneline --graph --all
git status
```

Expected: the file appears and status is clean. A branch points to commit history;
it is not a separate database or Python environment. Conflicting edits on branches
can require manual resolution; this example avoids conflicting changes.

## 8. Apply the habit to the learning project

Inspect before changing repository setup:

```sh
cd ~/projects/retail-pricing-learning
git rev-parse --show-toplevel
git status
```

If no repository exists, these commands report that. If they point to a parent
repository, discuss the intended layout before creating a nested repository.
Preserve any existing history and coordinate commits in shared files with your mentor.

For an intended standalone project with no existing repository, initialize it
once using `git init -b main` and configure local identity as above. Review the
existing `.gitignore`. Once lesson 1 creates `learning/db.py`, a focused commit is:

```sh
git add learning/db.py
git diff --staged
git commit -m "Add learning database connection helper"
```

Other useful commit boundaries: loader, SQL audit/views, exploration, modeling,
causal experiment, scenario checks, and dashboard. Commit meaningful checked
changes, with messages describing their purpose.

## 9. Understand optional remote hosting

```sh
git remote -v
```

This inspects remote locations; no output means none are configured. A commit
does not upload anything. A push sends commits to a configured remote and needs
authentication. Neither is required for the learning course.

Before later publishing, review staged files and existing history for credentials
and private data. Public/private visibility is a separate decision. Follow the
chosen host's official authentication instructions when you reach that stage.

## Troubleshooting

| Symptom | First check |
|---|---|
| Not a Git repository | Check `pwd` and use the initialized practice folder |
| Author identity unknown | Configure local name/email |
| Nothing to commit | Save edits, inspect status, stage the intended file |
| Latest edit absent from staged diff | Stage again after saving |
| Ignored file remains tracked | It was tracked before the rule; ignore rules are not history erasure |
| Unexpected editor | Use `-m` to supply a commit message in this lesson |
| Cannot switch due to local changes | Review and finish current work instead of discarding it |

## Checkpoint

Demonstrate two commits, an ignored practice `.env`, and a merged branch.
Explain working files, staging, commit, branch, merge, and remote without the
guide. Why can Git restore code but not automatically restore PostgreSQL rows?

Next: [lesson 1 — project foundations](01_foundations.md).
