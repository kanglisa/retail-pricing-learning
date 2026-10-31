# Publish this learning lab to your GitHub account

These are instructions for you to run. No repository has been created remotely
and nothing has been pushed by preparing this folder.

## 1. Copy the subproject to a standalone folder

From the original data science project root, choose a destination that does not
already exist. For example:

```sh
mkdir -p ~/projects
cp -R retail-pricing-learning ~/projects/retail-pricing-learning
cd ~/projects/retail-pricing-learning
pwd
ls -a
```

If that destination already exists, use another new folder name rather than
merging copies. Copy the whole subproject, including its dotfiles—not the parent
project's `.git`, workbook, `.env`, reports, or artifacts.

## 2. Initialize local history

```sh
git init -b main
git config user.name "Your Name"
git config user.email "your-chosen-author-email@example.com"
git add .gitignore .env.example README.md PUBLISHING.md compose.yaml requirements.txt lessons learning
git status --short
git diff --staged --stat
git diff --staged
```

Replace the example name/email. Verify staged paths contain only the learning
course and its starter files. Local `.env`, workbooks, generated learner reports,
and model artifacts should not appear. Press `q` to exit the diff pager.

```sh
git commit -m "Add hands-on retail data science learning course"
```

This creates a local commit. It does not upload anything.

## 3. Create an empty repository on GitHub

Create a repository under your own account, for example `retail-pricing-learning`.
Choose your intended visibility. Leave it empty: do not initialize it with a
README, license, or ignore file because this local repository already has content.
Choose any sharing license separately when you are ready; none is selected here.

Authenticate Git on the command line using your chosen GitHub-supported method.
The following example uses an HTTPS remote. Replace `YOUR_USERNAME` and the
repository name if necessary:

```sh
git remote add origin https://github.com/YOUR_USERNAME/retail-pricing-learning.git
git remote -v
git push -u origin main
```

GitHub's [official guide for adding local code](https://docs.github.com/en/migrations/importing-source-code/using-the-command-line-to-import-source-code/adding-locally-hosted-code-to-github)
covers authentication and remote setup. If an `origin` already exists, inspect
it before changing anything. If the remote already contains commits, do not
force-push; reconcile the histories or start with a new empty repository.

## 4. Check the published repository

Confirm the README renders, lesson links work, and no workbook or `.env` is
visible. A new learner should be able to clone the repository, follow README
setup, provide their own workbook, and complete the lessons without the parent
project. Future commits can add your reviewed Python/SQL implementations.
