---
title: "How We Used AI Workspaces to Deliver Faster Without Breaking the Platform"
description: "A simple story about solving a real business problem with Claude Code CLI, cmux, Git worktrees, symlinks, and Playwright automation."
pubDate: 2026-05-22
tags: ["AI", "Software Engineering", "Productivity", "Git Worktrees", "Claude Code", "Automation"]
readingTime: "8 min read"
draft: false
---

# How We Used AI Workspaces to Deliver Faster Without Breaking the Platform

Every technical improvement should start with a business problem.

For us, the problem was simple:

We had to deliver new features fast, while still maintaining a platform that was already running.

That sounds normal for most product teams, but in practice it creates a lot of pressure.

A customer needs a fix.  
A product manager needs a feature.  
The platform needs to stay stable.  
The team needs to move fast without creating new problems.

This is where engineering quality matters.

Not only in the code itself, but in the way the team works.

## The Real Problem Was Context Switching

Our platform was split across multiple repositories and technologies.

We had:

- AngularJS applications
- Angular 16 applications
- Node.js services with Express
- NestJS services

These repositories represented different parts of the same business platform.

A developer could be working on a new feature in one part of the system, then suddenly need to fix a bug in another part.

That meant switching branches, preparing environments, installing dependencies, logging in again, and rebuilding mental context.

The work itself was not always the slowest part.

The slow part was preparing to work.

And every time this happened, we lost focus.

For the business, this meant slower delivery.

For the team, it meant more friction.

## AI Was Not the Goal

When AI coding tools became more useful, we started exploring how they could help us.

But the goal was not to “use AI” just because it was popular.

The goal was to solve a real delivery problem.

We wanted a workflow where we could:

- work on bugs and features in parallel
- create clean development workspaces quickly
- avoid breaking our current work
- reduce setup time
- allow AI tools to work safely in isolated environments

That is when we started using Claude Code CLI.

Claude Code gave us a way to work with AI directly from the terminal.

But to make it useful in a real project, we needed a better workspace setup.

## Using cmux for Multiple AI Workspaces

We installed `cmux` so we could manage multiple workspaces from a left-side panel.

This helped us organize different development contexts.

For example, one workspace could be used for a bug fix.

Another workspace could be used for a new feature.

Another could be used for investigation or refactoring.

Each workspace had its own context.

This was important because AI works better when the task is clear and isolated.

Instead of one messy environment where everything was mixed together, we created separate rooms for separate tasks.

That made the workflow easier to manage.

## Discovering Git Worktrees

The next important piece was Git worktrees.

Normally, when working with Git, one repository has one active branch checked out.

So if you are working on a feature and need to fix a bug, you usually have to stop, stash your changes, switch branches, and later come back.

That creates friction.

Git worktrees allow you to have multiple working folders from the same repository.

Each folder can use a different branch.

This means you can keep your feature open in one folder and fix a bug in another folder.

For us, this was a perfect match for AI-assisted development.

Each AI session could work inside its own worktree.

That gave us isolation, safety, and speed.

## Building Our Own Worktree Wrapper

Git worktrees are powerful, but using them manually across many repositories can still be repetitive.

So we created a wrapper script.

The goal of the wrapper was simple:

Create a new development workspace with the same stack, quickly and consistently.

The wrapper did a few important things.

First, it created a new worktree from the source repository.

Then it asked us which branch we wanted to start from.

After that, it allowed us to give the workspace a custom name directly from the command.

This made it easy to create workspaces like:

```bash
create-workspace fix-login-bug
create-workspace new-checkout-flow
create-workspace investigate-payment-issue
```

The workspace name mattered because we often had multiple tasks running at the same time.

Clear names helped us understand what each workspace was for.

## The Slow Part: Dependencies

At first, creating new worktrees helped us separate work.

But there was still a big problem.

Each new workspace needed dependencies.

In JavaScript and Node.js projects, that usually means installing `node_modules`.

And that can be slow.

When you have multiple repositories and large applications, dependency installation can take minutes.

This was a problem because we wanted workspaces to be fast.

If creating a workspace takes too long, people stop using the workflow.

The process must feel instant.

Otherwise, it becomes another task.

## The Big Improvement: Quick Symlinks for node_modules

The biggest improvement came when we added quick symlinks for `node_modules`.

Instead of reinstalling dependencies every time we created a new workspace, we reused the existing `node_modules` folder from the source workspace.

The wrapper created a symbolic link from the new worktree to the existing dependencies.

In simple words, the new workspace did not need to rebuild the toolbox.

It reused the same toolbox.

Before this improvement, creating a workspace could take minutes because dependencies had to be installed again.

After this improvement, creating a workspace became almost instant.

That changed the feeling of the workflow completely.

It became normal to create a clean workspace for a new task.

And because it was fast, we used it more often.

## Automating Login with Playwright

The next problem was application login.

Our frontend applications were integrated with each other.

One app could redirect to another.  
Authentication had to work.  
The platform had to be opened in the correct state.

Doing this manually for every new workspace was annoying and slow.

So we created scripts using Playwright.

Playwright allowed us to automate the browser.

The script used saved development credentials from my machine and logged into the platform automatically.

This meant that after creating a workspace, we could also prepare the browser session quickly.

The workspace was not only created.

It was ready to use.

## What the Final Workflow Looked Like

After these improvements, the workflow became much simpler.

We could create a new workspace with one command.

That command would:

1. create a Git worktree
2. ask which branch to use
3. give the workspace a clear name
4. connect the needed dependencies using quick symlinks
5. prepare the environment
6. use Playwright scripts to login into the platform

Then Claude Code could work inside that isolated workspace.

This allowed us to run multiple development flows in parallel.

One workspace could be fixing a production-like bug.

Another workspace could be building a feature.

Another could be used to investigate an issue without touching the others.

## Why This Mattered for the Business

The business problem was not “developers need nicer tools.”

The business problem was speed and stability.

The company needed new features.

But the existing platform still needed care.

If developers lose too much time switching context, delivery slows down.

If developers rush because switching is painful, quality goes down.

Our workflow helped with both sides.

It allowed us to move faster without mixing unrelated work.

It reduced waiting time.

It made bug fixing easier.

It made feature development easier.

And it gave AI tools a safer and cleaner environment to work in.

## The Productivity Impact

After creating this workflow, we were able to fix bugs and actively continue feature development at the same time.

The productivity improvement was significant.

In many daily development tasks, it felt close to a 10x improvement.

Not because AI magically wrote everything.

But because the full workflow removed a lot of waste:

- less waiting
- less branch switching
- less dependency installation
- less manual login
- less environment setup
- less context loss

AI helped, but the real value came from combining AI with a workflow that matched the business need.

## Good Code Represents the Business

One important lesson from this experience is that good code is not only about clean syntax.

Good code should represent the business correctly.

And good engineering workflows should support the way the business needs to move.

In our case, the business needed fast delivery and stable maintenance.

So we created a workflow that supported both.

The architecture of the workflow matched the reality of the company:

- multiple apps
- multiple services
- multiple tasks
- multiple priorities

Instead of forcing developers into one workspace and one context, we allowed the workflow to represent the real world.

That made the system easier to work with.

## Final Thoughts

This was not only an AI experiment.

It was an engineering improvement built around a real business problem.

We started with a simple question:

How can we deliver faster while still maintaining the platform properly?

The answer was not one tool.

It was a combination of tools:

- Claude Code CLI for AI-assisted development
- cmux for managing multiple workspaces
- Git worktrees for isolated branches
- a custom wrapper for creating workspaces quickly
- quick symlinks for reusing `node_modules`
- Playwright scripts for automatic login

Together, these tools created a development workflow that helped us move faster and stay focused.

That is where AI becomes useful.

Not when it replaces understanding.

But when it supports a team that already understands the business problem.
