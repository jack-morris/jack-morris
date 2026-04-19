# Getting Good at Git with VS Code and WSL

## Overview
This guide helps you master Git using Visual Studio Code (VS Code) and Windows Subsystem for Linux (WSL). We'll cover basics, VS Code integration, WSL tips, resources, and hands-on exercises.

## Git Basics
Git is a distributed version control system. Key concepts:
- **Repository**: A folder tracked by Git.
- **Commit**: A snapshot of changes.
- **Branch**: A parallel version of the code.
- **Merge**: Combining branches.
- **Remote**: Online repository (e.g., GitHub).

Essential commands:
- `git init`: Initialize a repo.
- `git add .`: Stage all changes.
- `git commit -m "message"`: Commit staged changes.
- `git status`: Check repo status.
- `git log`: View commit history.
- `git branch`: List branches.
- `git checkout <branch>`: Switch branches.
- `git merge <branch>`: Merge a branch.

## Using Git in VS Code
VS Code has built-in Git support via the Source Control panel (Ctrl+Shift+G).

### Key Features:
- **View Changes**: See modified files in the panel.
- **Stage Files**: Click + to stage.
- **Commit**: Enter message and click checkmark.
- **Branches**: Manage branches in the bottom-left status bar.
- **Sync**: Push/pull from remotes.
- **Diff View**: Click a file to see changes.
- **Terminal Integration**: Run Git commands in the integrated terminal.

### Tips:
- Use the Git Graph extension for visual history.
- Enable GitLens for advanced features.
- Configure user settings: `git config --global user.name "Your Name"` and `git config --global user.email "your.email@example.com"`.

## WSL Integration
WSL allows running Linux commands on Windows. Git works seamlessly in WSL terminals.

### Setup:
- Install Git in WSL: `sudo apt update && sudo apt install git`.
- Use VS Code's WSL extension for remote development.
- Open folders in WSL: `code .` from WSL terminal.

### Advantages:
- Native Linux performance.
- Consistent environment.
- Access to Linux tools.

## Resources
- **Official Git Documentation**: https://git-scm.com/doc
- **VS Code Git Guide**: https://code.visualstudio.com/docs/sourcecontrol/overview
- **Interactive Tutorials**:
  - Learn Git Branching: https://learngitbranching.js.org/
  - GitHub Learning Lab: https://lab.github.com/
  - Codecademy Git Course: https://www.codecademy.com/learn/learn-git
- **Books**: "Pro Git" by Scott Chacon (free online).
- **Videos**: FreeCodeCamp Git tutorial on YouTube.

## Exercises
Practice in your `/home/jmorris/jack-morris` repo. Assume you have a `Python/file1.py` file.

### Exercise 1: Basic Workflow
1. Modify `Python/file1.py` (add a comment).
2. Stage and commit: `git add . && git commit -m "Add comment"`.
3. View log: `git log --oneline`.

### Exercise 2: Branching
1. Create branch: `git checkout -b feature-branch`.
2. Modify file (add a function).
3. Commit changes.
4. Switch back: `git checkout main`.
5. Merge: `git merge feature-branch`.
6. Delete branch: `git branch -d feature-branch`.

### Exercise 3: Remote Operations
1. Create a GitHub repo.
2. Add remote: `git remote add origin <url>`.
3. Push: `git push -u origin main`.
4. Make changes, commit, push again.
5. Simulate collaboration: Clone to another folder, make changes, push/pull.

### Exercise 4: Resolving Conflicts
1. Create two branches from main.
2. Modify the same line in `file1.py` in both branches.
3. Merge one into the other: See conflict, resolve in VS Code's diff view, commit.

### Exercise 5: Rebasing
1. Create a branch, make commits.
2. On main, make a commit.
3. Rebase branch onto main: `git rebase main`.
4. Practice interactive rebase: `git rebase -i HEAD~3`.

### Exercise 6: Advanced Commands
- Stashing: `git stash`, `git stash pop`.
- Resetting: `git reset --soft HEAD~1` (undo commit, keep changes).
- Cherry-picking: Pick a commit from another branch.
- Bisecting: Find buggy commit with `git bisect`.

### Tips for Exercises:
- Use VS Code's Source Control panel for staging/committing.
- Run commands in WSL terminal for practice.
- Experiment safely: Use `git reflog` to recover.
- Practice daily: Make small commits.

## Next Steps
- Contribute to open-source projects on GitHub.
- Use Git in team workflows (pull requests, code reviews).
- Learn Git hooks for automation.
- Explore GitHub Actions for CI/CD.

Start with basics, then progress to complex scenarios. Consistency is key!