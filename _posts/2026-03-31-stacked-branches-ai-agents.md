---
layout: post
title: "Stacked branches: why your AI agent code reviews will improve dramatically"
date: 2026-03-31 10:00:00 -0300
categories: [git, workflow, ai]
tags: [git, stacked-branches, code-review, ai-agents]
---

You've just finished explaining a feature to your AI coding assistant. After a few minutes of typing, it proudly announces: "Done! I've made changes to 12 files across 4 different subsystems."

You open the pull request and your heart sinks. 847 lines changed. A mix of bug fixes, refactoring, and the actual feature you asked for—all lumped together in a single massive PR.

This scenario is becoming increasingly common. AI agents are incredibly capable at generating code, but they don't naturally think in terms of "one logical change per review." They optimize for completing the task, not for making your code reviewer's life easier.

The good news? There's a workflow pattern that addresses exactly this problem: **stacked branches**.

## The Problem with Monolithic Pull Requests

When you ask an AI agent to implement something, it often needs to:

- Create supporting infrastructure before the feature itself
- Fix underlying issues it encounters along the way
- Refactor code it finds in the way
- Add tests for everything

The result is a PR that tries to do too much at once. Traditional code review becomes painful because reviewers must hold a massive mental model in their heads to understand if the changes are correct.

With AI agents, this problem intensifies. A human developer might naturally break their work into smaller commits. An AI agent, however, will happily refactor a utility function in file A to support a change in file B, all as part of "the feature you asked for."

The consequences are real:

- **Reviewers skip thorough checks** when faced with 800+ lines
- **Bugs slip through** because no one can track all the side effects
- **Reverts become nuclear** — one broken change takes down the entire PR
- **Feedback gets delayed** as reviewers procrastinate on intimidating PRs

## What Are Stacked Branches?

Stacked branches (also called "stacked PRs") is a workflow where you create a chain of dependent branches, each building on the previous one.

```
main
  └─> feature/auth
        └─> feature/two-factor
              └─> feature/password-reset
```

Each branch represents a single logical unit of work. You review and merge them in sequence — from the base of the stack to the tip.

Instead of one PR with 12 files changed, you might have:

1. `feature/auth` — 45 lines: Basic authentication setup
2. `feature/two-factor` — 180 lines: 2FA implementation
3. `feature/password-reset` — 120 lines: Password recovery flow
4. `feature/tests` — 220 lines: Test coverage for all above

Each PR is focused, reviewable in 10 minutes, and easy to approve or reject independently.

## The Git Mechanics

The core concept is simple: each branch is based on the previous one, not on main.

```bash
# Start with your base branch
git checkout -b feature/auth

# Do work, commit
git add .
git commit -m "Add authentication model"

# Create the next branch ON TOP of current one
git checkout -b feature/two-factor
# (now feature/two-factor already contains feature/auth changes)

# Work on two-factor...
git add .
git commit -m "Implement two-factor flow"
```

### The Secret Sauce: git rebase --update-refs

This is where most developers get stuck. You created a stack of 4 branches, everything looks great, and then you discover a bug in the base branch (`feature/auth`). You fix it, but now all the branches on top are out of date.

Before Git 2.38, you had to manually rebase each downstream branch one by one:

```bash
git checkout feature/auth
# fix the bug
git commit --amend

# manually rebase each branch
git rebase feature/auth feature/two-factor
git rebase feature/two-factor feature/password-reset
git rebase feature/password-reset feature/tests
```

Tedious. Error-prone. A dealbreaker for this workflow.

**Enter `git rebase --update-refs`**:

```bash
git checkout feature/auth
# fix the bug
git commit --amend

# this ONE command updates all downstream branches automatically
git rebase --update-refs
```

That's it. One command, and Git figures out the entire dependency chain and rebases everything in the right order.

This single command is what makes stacked branches practical in 2026. It's the key that unlocks the entire workflow.

## Why This Works Better with AI Agents

Here's where stacked branches really shine in the AI era:

### 1. Task Decomposition Becomes Natural

When working with an AI agent, you can structure your prompts differently:

> "First, let's add the user authentication model and login controller. Just those two files. Create a branch called `feature/auth` and make those changes."

Wait for it to be merged. Then:

> "Great, now let's add two-factor authentication. Create branch `feature/two-factor` on top of the auth changes we just merged."

This "assembly line" approach aligns perfectly with how AI agents work — one clear task at a time, with visible progress between each step.

### 2. Review Context Shrinks Dramatically

A reviewer looking at `feature/password-reset` doesn't need to understand how authentication works. They only need to see:

- What changed in the password reset flow
- How it interacts with the (already reviewed) auth layer

The cognitive load drops significantly.

### 3. Bisectability Improves

If something breaks after merging `feature/password-reset`, you can use `git bisect` to find exactly which commit caused it. With a monolithic PR, you don't have that granularity.

### 4. Partial Reverts Are Safe

Suppose the 2FA implementation has a bug, but the password reset is fine. With stacked branches, you can revert just `feature/two-factor` and the rest of your stack still works (after rebasing).

With a monolithic PR? You'd need to revert the entire thing and ask the agent to re-do everything.

## Tools That Make This Easier

While you can do stacked branches with vanilla Git, some tools make life smoother:

- **`git rebase --update-refs`** (Git 2.38+) — The game changer mentioned above
- **ghstack** — GitHub's tool for managing stacked PRs
- **git-town** — A Git extension for sophisticated workflows
- **git worktree** — For those times when you need to work on multiple branches simultaneously

If you use GitHub, the `ghstack` CLI is particularly nice. It automatically creates stacked PRs and keeps them in sync as you rebase.

## Conclusion

AI coding agents are powerful tools, but they need structure to be truly effective. Stacked branches provide that structure by enforcing small, reviewable units of change.

The workflow might feel unfamiliar at first — especially if you're used to one-big-PR-per-feature. But once you experience the relief of reviewing a 50-line PR instead of an 800-line monster, you won't want to go back.

The next time you work with an AI agent, try this: before asking it to write code, first ask it to create a branch. Then ask for a small, focused change. Review and merge. Repeat.

Your code reviewers (and your future self) will thank you.