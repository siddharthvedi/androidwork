# Agent Guidelines

This repository is a Next.js project using pnpm and Node.js 20. Follow these practices when making changes:

## Tooling & Commands
- Use **pnpm** for installing dependencies and running scripts (CI uses `pnpm install` followed by `pnpm run build`).
- Target **Node.js 20** locally to match CI runners.
- Run `pnpm build` before opening a PR to mirror the CI build step and catch compile-time issues early.

## Localization
- Do **not** modify files under `public/locales/**` directly. Localization strings are synchronized from the GameTrainer-i18n repository; CI blocks direct edits.

## Next.js Best Practices
- Prefer TypeScript-first changes and keep components fully typed.
- Use functional React components with hooks; avoid adding new class components.
- Keep server/client boundaries explicit: avoid importing client-only modules into server code and vice versa.
- Ensure new environment variables are documented and consumed via `process.env` with appropriate runtime checks.

## Operational Notes
- When updating Node.js versions, remember that production uses pm2; ensure pm2 configuration is refreshed after node upgrades (see README).
- If you add build or runtime dependencies, verify they are compatible with pnpm's lockfile and the CI/CD pipelines defined under `.github/workflows/`.

## Code Formatting & Linting

This project uses **Prettier** for code formatting and **ESLint** for code quality. Follow these guidelines when making changes:

### Prettier Configuration

The project uses Prettier with the following settings (defined in `.prettierrc`):
- **Trailing commas**: ES5 style
- **Tab width**: 2 spaces
- **Semicolons**: Required
- **Quotes**: Single quotes

### When to Format Code

- **Always format files you modify** before completing your task
- Use `pnpm prettier` to format specific files or the entire project
- Format only files you've directly edited — do NOT format unrelated files

### Formatting Commands

```bash
# Format a specific file
pnpm prettier --write path/to/file.tsx

# Format multiple specific files
pnpm prettier --write path/to/file1.tsx path/to/file2.ts

# Format all files (use with caution - only if explicitly asked)
pnpm prettier
```

**Important**: 
- Do NOT run `pnpm prettier` on the entire project unless explicitly requested
- Format only files you've modified in your current task
- Prettier respects `.prettierignore` (excludes `.next`, `node_modules`, `dist`, `.yarn`)

### ESLint Configuration

The project uses ESLint with:
- **Next.js** recommended rules (`next`, `next/core-web-vitals`)
- **ESLint recommended** rules
- **Custom rules**: Unused variables are warnings (not errors), with `_` prefix ignored

### When to Run Linting

- **Before completing your task**: Check linting on files you've modified
- Use `read_lints` tool on your edited files only
- Do NOT run project-wide linting (`pnpm lint`) unless explicitly asked
- Fix linting errors in files you've modified

### Linting Commands

```bash
# Lint specific files (recommended)
npx eslint path/to/file.tsx path/to/file2.ts

# Lint entire project (only if explicitly asked)
pnpm lint

# Auto-fix linting issues (use with caution)
npx eslint --fix path/to/file.tsx
```

**Important**:
- Do NOT run `pnpm lint` on the entire project unless explicitly requested
- Only fix linting errors in files you've directly modified
- Report linting errors in unrelated files but do NOT fix them

### Scope Your Formatting & Linting

When checking or fixing code quality:

- ✅ **DO**: Format/lint files you've directly modified
- ✅ **DO**: Use `read_lints` on your edited files only
- ✅ **DO**: Fix formatting/linting issues in your own changes
- ❌ **DON'T**: Run formatting/linting on the entire project
- ❌ **DON'T**: Fix formatting/linting in files you didn't modify
- ❌ **DON'T**: Auto-format unrelated files "while you're at it"

### Integration with Other Guidelines

- **Parallel Agent Coordination**: Only format/lint files related to your task
- **Build & Server Commands**: Formatting/linting doesn't require a build
- **Pre-Push Checklist**: Ensure your modified files are formatted and pass linting before pushing

### Common Scenarios

**Scenario 1: You modify a file**
1. Make your code changes
2. Run `pnpm prettier --write path/to/your/file.tsx`
3. Run `read_lints` on that file
4. Fix any linting errors
5. Complete your task

**Scenario 2: You see formatting issues in a file you didn't modify**
- Report it but do NOT fix it
- Another agent might be working on that file
- Only fix formatting in files you've directly edited

**Scenario 3: User asks to format the entire project**
- This is an explicit request — proceed
- Run `pnpm prettier`
- Review changes before committing

## GitHub Workflow & Deployment

> **Scope:** These instructions apply ONLY to `GameTrainer/GameTrainer-Frontend` and `GameTrainer/GameTrainer-Backend` repositories.

### ⚠️ Explicit Push Approval Required

**By default, all work stays LOCAL.** Do NOT push to `feature/prd-design` or create PRs unless the user explicitly requests it.

| Action | Requires Explicit Request? |
|--------|---------------------------|
| Making code changes locally | ❌ No — proceed freely |
| Running dev server locally | ❌ No — proceed freely |
| Testing locally | ❌ No — proceed freely |
| Git commits (local only) | ❌ No — proceed freely |
| **Pushing to `feature/prd-design`** | ✅ **YES — ask first** |
| **Creating PRs** | ✅ **YES — ask first** |
| **Merging PRs** | ✅ **YES — ask first** |

**Examples of explicit requests:**
- "Push this to prd-design"
- "Deploy this"
- "Create a PR"
- "Ship it"
- "Let's push"

**If the user hasn't explicitly requested a push/deploy, do NOT do it.** Just confirm the local changes are working and wait for further instructions.

### PRD Design Environment (Development Testing)

This environment is for testing backend + frontend changes together **before** deploying to staging. It sits between local development and staging/main.

| Component | URL / Branch |
|-----------|--------------|
| Frontend App | `https://prd-designapp.gametrainer.dev` |
| Frontend Branch | `feature/prd-design` |
| Backend API | `https://prd-designapi.gametrainer.dev` |
| Backend Branch | `feature/prd-design` |
| Database | Staging MongoDB (shared with staging) |

#### How to Deploy Frontend Changes

1. Ensure you're on the `feature/prd-design` branch
2. **Always run `pnpm build` before pushing** to catch TypeScript errors locally
3. Push commits to `feature/prd-design`
4. GitHub Actions will automatically deploy (see `NodeCD-FEAT.yml`)
5. **Monitor the GitHub Actions workflow** to ensure deployment succeeds
6. Test your changes at `https://prd-designapp.gametrainer.dev`

#### How to Deploy Backend Changes

1. Push commits to the `feature/prd-design` branch in [GameTrainer-Backend](https://github.com/GameTrainer/GameTrainer-Backend/tree/feature/prd-design)
2. The frontend at `prd-designapp.gametrainer.dev` automatically points to this backend
3. Test your changes at `https://prd-designapp.gametrainer.dev`

#### Access Notes
- Frontend repo: Write access available
- Backend repo: Write access available for `feature/prd-design` and feature branches — can push directly or via PR

#### When to Use This Environment
- Testing backend API changes before merging to staging
- Validating frontend + backend integration
- Debugging issues that require backend modifications

### Syncing with Staging

When changes are pushed to `staging`, the `staging-feature-sync.yml` workflow automatically creates PRs to sync feature branches.

#### Handling Sync PRs

1. **Review the PR** created by the automation (labeled `automated,staging-feature-sync,frontend`)
2. **Check for conflicts** — if conflicts exist, resolve them manually
3. **ALWAYS let the user approve conflicts** before merging — never auto-resolve
4. After approval, merge the PR to keep `feature/prd-design` up-to-date with staging

### Pre-Push Checklist

Before pushing ANY changes to `feature/prd-design` or other branches:

1. **Format your changes:** Ensure all modified files are formatted with Prettier
   ```bash
   pnpm prettier --write path/to/your/files
   ```
2. **Check linting:** Verify your modified files pass ESLint
   ```bash
   read_lints path/to/your/files
   # or use: npx eslint path/to/your/files
   ```
3. **Build first:** Run `pnpm build` to catch TypeScript errors
   ```bash
   pnpm build
   ```
4. **Fix any errors** before pushing — do NOT push code that fails to build or has linting errors
5. **Push your changes**
6. **Monitor GitHub Actions:** Go to the Actions tab and verify the workflow completes successfully
7. **If the workflow fails:** Read the logs, fix the issue, and push again

### Always Share URLs

When performing GitHub operations, **always share the relevant URL** with the user:

| Action | URL to Share |
|--------|--------------|
| Creating a PR | PR URL (e.g., `https://github.com/.../pull/123`) |
| Merging a PR | PR URL that was merged |
| Pushing to branch | Link to the commit or branch |
| Checking Actions | Workflow run URL (e.g., `https://github.com/.../actions/runs/123`) |
| CI failure | Direct link to failed job logs |

This ensures the user can quickly access and verify the operation.

### GitHub Actions Monitoring

After pushing, **recursively monitor** the deployment until it completes:

1. Go to [GitHub Actions](https://github.com/GameTrainer/GameTrainer-Frontend/actions)
2. Find the workflow run for your push
3. **Keep checking the workflow status** until all jobs complete
4. If jobs are still running, wait and check again — do NOT assume success
5. Only confirm deployment is complete when all jobs show ✅
6. If any job fails:
   - Read the full error logs
   - Fix the issue locally
   - Run `pnpm build` to verify the fix
   - Push again and repeat monitoring from step 1

**Do NOT move on to other tasks until deployment is confirmed successful.**

## Intellectual Honesty

Prioritize truth over engagement. Follow these principles:

### Avoid Sycophancy
- Do NOT default to "You're absolutely right!" or similar affirmations.
- If you agree, explain **why** briefly. If you disagree, say so respectfully.
- The user values honest feedback over validation.

### Share Your Perspective
- If you see a better approach, propose it — even if it differs from what was asked.
- Flag potential issues, tradeoffs, or risks you notice.
- Don't hide concerns to avoid friction.

### Balanced Analysis
- When asked for opinions, give a balanced view with pros and cons.
- Acknowledge uncertainty when it exists — don't fake confidence.
- Distinguish between facts, best practices, and personal recommendations.

### Follow Commands, But Flag Concerns
- If the user insists on an approach after hearing your concerns, execute it.
- But always voice your concern **once** before proceeding.
- "I'll do this as requested, but note that [concern]" is a valid response.

## Problem-Solving Approach

Before writing any code, follow this process:

### Understand First, Implement Last
- Spend more time **analyzing the problem** than implementing the solution.
- Read and understand all relevant existing code before proposing changes.
- Ask clarifying questions if requirements are ambiguous — don't assume.

### Think from First Principles
- Don't copy patterns blindly. Understand **why** something is done a certain way.
- Consider: Is this the right abstraction? Is there a simpler approach?
- Challenge assumptions — existing code isn't always correct or optimal.

### Build for the Long Term
- Prefer maintainable, readable solutions over clever or quick fixes.
- Consider how this change affects the broader system architecture.
- Avoid introducing technical debt — if a shortcut is necessary, flag it explicitly.

### Step-by-Step Reasoning
- Break complex problems into smaller, manageable pieces.
- Validate your understanding at each step before moving on.
- When debugging, form hypotheses and test them systematically.

## Parallel Agent Coordination

This codebase may have multiple AI agents working simultaneously. Follow these rules to avoid conflicts:

### File Ownership
- Only edit files directly related to your assigned task.
- Do NOT "fix" TypeScript errors, linting issues, or warnings in files you didn't create or modify in this session.
- If you encounter errors in unrelated files, report them but do not fix them.

### Build & Server Commands
- Do NOT run `pnpm build`, `pnpm dev`, or restart the dev server unless explicitly asked.
- Do NOT run project-wide linting or type-checking commands (`pnpm lint`, `tsc`).
- If you need to verify your changes compile, use `npx tsc --noEmit <your-specific-files>` instead of a full build.
- **Formatting/Linting**: See [Code Formatting & Linting](#code-formatting--linting) section for guidelines on when to format/lint files.

### Scope Your Checks
- When checking for errors, only check files you've directly modified:
  - ✅ `read_lints` on your edited files only
  - ❌ `read_lints` on the entire workspace
- Assume other agents are handling their own files.

### When in Doubt
- Ask the user before modifying files outside your task scope.
- If a file you need to edit has recent uncommitted changes you didn't make, stop and ask.

## Git Safety & Destructive Operations

Git commands that can cause data loss require extra caution. **Always ask for explicit confirmation before running destructive git operations.**

### Destructive Commands Requiring Validation

These commands can permanently lose uncommitted work or rewrite history. **NEVER run these without user confirmation:**

| Command | Risk |
|---------|------|
| `git reset --hard` | Discards all uncommitted changes permanently |
| `git checkout -- <file>` | Discards uncommitted changes to specific files |
| `git restore --staged --worktree` | Discards both staged and unstaged changes |
| `git clean -fd` | Deletes untracked files and directories |
| `git stash drop` / `git stash clear` | Permanently deletes stashed changes |
| `git rebase` (interactive or not) | Rewrites commit history |
| `git push --force` / `git push -f` | Overwrites remote history |
| `git branch -D` | Force-deletes a branch (even if unmerged) |
| `git reflog expire` / `git gc --prune=now` | Removes ability to recover lost commits |

### First-Time Warnings

- If this is the **first destructive git command in the conversation**, provide an explicit warning and list what will be lost.
- Even if the user asked for it, confirm: *"This will permanently discard X. Proceed?"*
- If you've already done a similar operation in this conversation AND the user approved it, you may proceed with less friction — but still mention what's happening.

### Pre-Flight Checks (MANDATORY)

Before ANY destructive git operation, run these checks and **report the results to the user**:

```bash
# Check for uncommitted changes
git status --short

# Check for untracked files
git ls-files --others --exclude-standard
```

- If there are **uncommitted changes**: Stop and warn. List the affected files.
- If there are **untracked files** that would be deleted (e.g., `git clean`): Stop and list them.
- If the working directory is clean, you may proceed (but still confirm for first-time operations).

### Safe Alternatives to Suggest

When the user wants to undo or reset something, prefer safer alternatives:

| Instead of... | Suggest... |
|---------------|------------|
| `git reset --hard` | `git stash` (preserves changes, recoverable) |
| `git checkout -- .` | `git stash` or commit first |
| `git clean -fd` | `git clean -fdn` (dry-run first to preview) |
| `git push --force` | `git push --force-with-lease` (safer, checks for upstream changes) |
| `git branch -D` | `git branch -d` (refuses if unmerged) |

### Recovery Reminders

If a destructive operation is approved and executed, remind the user:
- `git reflog` can recover commits for ~30 days (if not pruned).
- Stashed changes are recoverable until explicitly dropped.
- Force-pushed remote changes may be recoverable via reflog on the server (time-limited).

### Never Assume, Always Confirm

- Do NOT assume the user wants to discard their work.
- Do NOT chain destructive commands silently.
- If a task requires discarding changes, break it into steps and confirm each destructive step.

## Testing Approach

When the user asks to test or verify functionality, follow this order:

### Terminal-First Testing
- **Always** start with terminal-based verification before browser-based testing.
- Check console logs, terminal output, and command-line feedback first.
- Use `console.log` debugging and terminal output to validate logic before spinning up browser tools.
- Run relevant scripts or commands in the terminal to catch issues early.

### Browser Testing Second
- Only move to browser-based testing after terminal verification passes.
- Browser testing is for visual/UI validation and end-to-end user flows.
- Use browser tools (`browser_snapshot`, `browser_click`, etc.) for interaction testing after logic is confirmed working.

### Rationale
Terminal-based testing is faster, more deterministic, and catches most logical errors. Browser testing adds overhead and should be reserved for UI-specific validation.
