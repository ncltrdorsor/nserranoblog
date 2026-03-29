---
layout: post
title: "Sharing Copilot Agent Skills Across Repositories with Git Submodules"
date: 2026-03-29
categories: copilot ai devops
excerpt: "How to maintain a single source of truth for your team's AI coding instructions using git submodules and auto-update hooks."
---

If your team works across multiple repositories, you've probably solved the "shared code" problem already — NuGet packages, npm modules, git submodules, or just copying files around. But the rise of AI coding agents and the tooling around them has introduced a new challenge: **shared AI knowledge**.

GitHub Copilot now lets you define [**agent skills**](https://code.visualstudio.com/docs/copilot/customization/agent-skills) — bundled instructions, procedures, and templates that teach the AI how your team works. Think of them as codified expertise: your coding standards, review checklists, and testing patterns, packaged so the agent follows them consistently. The problem is, these skills live inside each repository. If you have 10 projects, you're maintaining 10 copies of the same skills. Someone updates the SQL review standards in one repo, and the other nine are immediately stale.

This article shows how to solve that with a single source of truth for agent skills, distributed automatically to every project repo using [git submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules).

---

## What Are Agent Skills?

[Agent skills](https://code.visualstudio.com/docs/copilot/customization/agent-skills) are folders containing a `SKILL.md` file that Copilot discovers and loads on-demand. They live in `.github/skills/<skill-name>/` inside your repository.

A skill looks like this:

```yaml
---
name: dotnet-api
description: 'Build .NET Web API endpoints. Use when creating REST controllers, API routes, DTOs, or service layers in ASP.NET Core projects.'
---

# .NET API Development

## When to Use
- Creating new API controllers or endpoints
- Designing DTOs and request/response models
- Wiring up dependency injection for services

## Procedure
1. Create the DTO models in the Models/ folder
2. Create or update the service interface and implementation
3. Register the service in Program.cs
4. Create the controller with appropriate route attributes
5. Add input validation using data annotations or FluentValidation

## Standards
- Use [ApiController] attribute on all controllers
- Return ActionResult<T> from endpoints
- Use async/await for all I/O-bound operations
- Follow REST naming conventions: plural nouns for resources
```

The `description` field is critical — it's how Copilot decides whether to load the skill. When you ask Copilot to "create a new API endpoint," it matches against those keywords and loads the full procedure.

Skills can also include scripts, templates, and reference docs in subfolders. Copilot [loads them progressively](https://code.visualstudio.com/docs/copilot/customization/agent-skills#_how-skills-are-loaded) — only the name and description at first (~100 tokens), then the full body when relevant, then referenced files only when needed.

---

## The Problem: Skills Don't Travel

Skills are great for a single repo. But teams don't work in a single repo. You might have:

- A backend API project
- A frontend SPA
- A shared library
- A data pipeline
- Infrastructure-as-code

Each one benefits from the same coding standards, review procedures, and testing patterns. Without a sharing mechanism, you either:

1. **Copy skills manually** — they drift immediately
2. **Don't share them** — each repo has incomplete or inconsistent guidance
3. **Put everything in one monorepo** — not always practical

---

## The Solution: One Core Repo, Git Submodules Everywhere

The approach is simple:

1. **Create a core repository** that holds all shared skills at the root level
2. **Add it as a git submodule** in every project repo, mapped to `.github/skills/`
3. **Use git hooks** to auto-update the submodule so developers always have the latest version

Here's what the core repo looks like:

```
core-repo/
├── dotnet-api/SKILL.md
├── dotnet-testing/SKILL.md
├── sql-review/SKILL.md
├── code-review/SKILL.md
└── README.md
```

Each folder at the root is a self-contained skill. No `.github/skills/` nesting inside core-repo itself — this is important because when it's mounted as a submodule at `.github/skills/`, the folders end up exactly where Copilot expects them.

And here's what a consuming project looks like:

```
my-project/
├── .github/
│   └── skills/              ← git submodule → core-repo
│       ├── dotnet-api/
│       ├── dotnet-testing/
│       ├── sql-review/
│       └── code-review/
├── .githooks/
│   ├── post-checkout        ← auto-updates skills on branch switch
│   └── post-merge           ← auto-updates skills on pull
├── .gitmodules
└── src/
    └── ...
```

Copilot scans `.github/skills/` and discovers every skill from core-repo automatically. No configuration needed.

---

## Step-by-Step Setup

### 1. Create the Core Repository

Create a new repo (e.g., `core-repo`) and add your skills as root-level folders:

```bash
mkdir core-repo && cd core-repo
git init

# Create your first skill
mkdir dotnet-api
```

Add a `SKILL.md` to each skill folder with the required frontmatter:

```yaml
---
name: dotnet-api       # Must match the folder name
description: 'Build .NET Web API endpoints. Use when creating REST controllers, API routes, DTOs, or service layers.'
---

# .NET API Development

## Procedure
1. Create the DTO models
2. Create the service interface and implementation
3. Register in Program.cs
4. Create the controller
5. Add input validation
```

Commit and push:

```bash
git add -A
git commit -m "Add initial shared skills"
git remote add origin https://github.com/your-org/core-repo.git
git push -u origin main
```

**Key design decision:** Skills live at the repo root, not inside `.github/skills/`. This is because the submodule mount point in consuming repos will be `.github/skills/` — so the folder structure maps correctly.

### 2. Add the Submodule to a Project Repo

In any project repository:

```bash
git submodule add --branch main -- https://github.com/your-org/core-repo.git .github/skills
```

This does three things:
- Clones core-repo into `.github/skills/`
- Creates a `.gitmodules` file tracking the submodule URL and branch
- Records the pinned commit

Your `.gitmodules` will look like:

```ini
[submodule ".github/skills"]
    path = .github/skills
    url = https://github.com/your-org/core-repo.git
    branch = main
```

The `branch = main` part means the submodule tracks core-repo's main branch, so `git submodule update --remote` always pulls the latest.

### 3. Add Auto-Update Hooks

Submodules are [pinned by design](https://git-scm.com/book/en/v2/Git-Tools-Submodules#_pulling_in_upstream_changes_from_the_submodule_remote) — they don't update automatically. Without hooks, developers would need to manually run `git submodule update --remote` to get new skills. Let's automate that.

Create a `.githooks/` folder in your project repo with two [git hooks](https://git-scm.com/docs/githooks):

**`.githooks/post-merge`** — runs after every `git pull`:

```bash
#!/bin/sh
echo "Updating shared agent skills from core-repo..."
git submodule update --remote --merge .github/skills 2>/dev/null
if [ $? -eq 0 ]; then
    echo "Agent skills updated successfully."
else
    echo "Warning: Could not update agent skills submodule. Run manually:"
    echo "  git submodule update --remote --merge .github/skills"
fi
```

**`.githooks/post-checkout`** — runs after branch switches:

```bash
#!/bin/sh
# $3 = 1 if a branch checkout, 0 if a file checkout
if [ "$3" = "1" ]; then
    echo "Updating shared agent skills from core-repo..."
    git submodule update --remote --merge .github/skills 2>/dev/null
    if [ $? -eq 0 ]; then
        echo "Agent skills updated successfully."
    else
        echo "Warning: Could not update agent skills submodule. Run manually:"
        echo "  git submodule update --remote --merge .github/skills"
    fi
fi
```

The `post-checkout` hook checks `$3` to only run on branch checkouts, not file checkouts. This avoids unnecessary fetches when you're just restoring a single file.

Then configure git to use the versioned hooks directory:

```bash
git config core.hooksPath .githooks
```

Commit everything:

```bash
git add .github/skills .githooks .gitmodules
git commit -m "Add shared agent skills submodule and auto-update hooks"
git push
```

### 4. Onboard Your Team

For developers cloning the project for the first time:

```bash
git clone --recurse-submodules https://github.com/your-org/my-project.git
cd my-project
git config core.hooksPath .githooks
```

If they already cloned without `--recurse-submodules`:

```bash
git submodule update --init
git config core.hooksPath .githooks
```

That's it. From this point on, skills update automatically whenever they pull or switch branches.

---

## The Workflow in Practice

Here's what this looks like day-to-day:

**Adding a new skill** (done once, in core-repo):

```bash
cd core-repo
mkdir dotnet-testing
# Create SKILL.md with frontmatter and procedures
git add -A && git commit -m "Add dotnet-testing skill" && git push
```

**Every other developer** — the next time they `git pull` or switch branches in any project repo, the `post-merge` or `post-checkout` hook fires and pulls the new skill automatically. They see:

```
Updating shared agent skills from core-repo...
Agent skills updated successfully.
```

And the new skill is immediately available in Copilot chat.

**Project-specific skills** that shouldn't be shared can go in [`.agents/skills/`](https://code.visualstudio.com/docs/copilot/customization/agent-skills#_skill-file-locations) — an alternative discovery path that won't conflict with the submodule at `.github/skills/`.

---

## Things to Watch Out For

### Submodule Pointer Drift

When the hook updates the submodule, it changes the pinned commit in the working tree. This shows up as a modified `.github/skills` in `git status`. You can either:

- **Commit the updated pointer** — tracks which version of skills each project was using at any given commit
- **Ignore it** — add `.github/skills` to a personal `.git/info/exclude` if you don't want to see the noise

### Hooks Don't Fire on No-Op Pulls

If you run `git pull` and there are no new commits in the project repo, the `post-merge` hook doesn't fire (there's no merge). The skills only update when there's actual new content to merge. For a manual refresh:

```bash
git submodule update --remote --merge .github/skills
```

### There's No `post-fetch` Hook

Git doesn't have a hook for `git fetch`. Since `git pull` = fetch + merge, the `post-merge` hook covers pulls. But a standalone `git fetch` won't trigger a skills update. In practice this rarely matters — `fetch` doesn't change your working tree anyway.

### The `core.hooksPath` Setting Is Local

[`core.hooksPath`](https://git-scm.com/docs/git-config#Documentation/git-config.txt-corehooksPath) is a local config setting — it doesn't travel with the repo. Every developer needs to run it once after cloning. Document it in your README or onboarding guide. If your team uses a setup script, add it there.

### Skill Discovery Depends on the `description` Field

The [`description`](https://code.visualstudio.com/docs/copilot/customization/agent-skills#_skillmd-format) in the SKILL.md frontmatter is how Copilot finds skills. If the keywords aren't in the description, the agent won't load it. Use the "Use when..." pattern with specific trigger words:

```yaml
# Bad — too vague
description: 'A helpful testing skill'

# Good — keyword-rich with trigger phrases
description: 'Write and review unit tests in .NET. Use when creating xUnit/NUnit tests, mocking dependencies, or improving test coverage.'
```

---

## Adding This to More Repos

For every new project, it's two commands:

```bash
git submodule add --branch main -- https://github.com/your-org/core-repo.git .github/skills
git config core.hooksPath .githooks
```

Then copy the `.githooks/` folder from any existing project (or keep it in core-repo's README as a template).

If you find yourself doing this often, consider making a [repo template](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-template-repository) on GitHub with the submodule and hooks pre-configured.

---

## Wrapping Up

This approach gives you:

- **Single source of truth** — skills are defined once in core-repo
- **Automatic distribution** — git submodules + hooks keep every project current
- **Zero manual sync** — developers get updates on pull and branch switch
- **Version control** — full git history of skill changes
- **Works everywhere** — CI, headless environments, any git client

It mirrors the same pattern teams already use for shared libraries and NuGet packages, but applied to AI coding instructions. The skills go where Copilot looks for them, and git handles the rest.
