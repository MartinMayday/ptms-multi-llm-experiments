# Multi-LLM Experimental Workflow
## Project Task Management System - Parallel Variant Development

**Repository:** https://github.com/MartinMayday/ptms-multi-llm-experiments  
**Approach:** Git Worktrees with isolated LLM variants  
**Goal:** Compare Claude, GPT, and Gemini implementations; merge best components

---

## Architecture Overview

```
                    GitHub Repository
         ptms-multi-llm-experiments
                    │
        ┌───────────┼───────────┐
        │           │           │
    ┌───┴───┐   ┌───┴───┐   ┌───┴───┐
    │ main  │   │variant│   │variant│
    │(stable│   │-claude│   │-gpt   │
    │ref)   │   │       │   │       │
    └───┬───┘   └───┬───┘   └───┬───┘
        │           │           │
   Worktrees (Independent Working Directories)
        │           │           │
┌───────┴──────┬────┴────┬──────┴──────┐
│              │         │             │
│ptms-       │ptms-   │ptms-       │ptms-
│experiments │experim │experiments │experiments
│(main)      │-claude │-gpt        │-gemini
│            │        │            │
│ Planning   │ Claude │ GPT        │ Gemini
│ Files      │ Code   │ Code       │ Code
│            │        │            │
└────────────┴────────┴────────────┴─────┘
        │           │           │
        └───────────┼───────────┘
                    │
              Validation & Merge
                    │
              → main branch
```

---

## Worktree Structure

| Directory | Branch | LLM Provider | Status | Purpose |
|-----------|--------|--------------|--------|---------|
| `ptms-experiments/` | `main` | Reference | ✅ Stable | Planning files, specs, integration |
| `ptms-experiments-claude/` | `variant-claude` | **Claude** (Anthropic) | 🔄 Ready | Claude-optimized implementation |
| `ptms-experiments-gpt/` | `variant-gpt` | **GPT** (OpenAI) | 🔄 Ready | GPT-optimized implementation |
| `ptms-experiments-gemini/` | `variant-gemini` | **Gemini** (Google) | 🔄 Ready | Gemini-optimized implementation |

**Location:** `/Users/nosrcadmin/aos/`

---

## Quick Commands

### View All Worktrees
```bash
cd /Users/nosrcadmin/aos/ptms-experiments
git worktree list
```

### Switch Between Variants
```bash
# Work in Claude variant
cd /Users/nosrcadmin/aos/ptms-experiments-claude

# Work in GPT variant
cd /Users/nosrcadmin/aos/ptms-experiments-gpt

# Work in Gemini variant
cd /Users/nosrcadmin/aos/ptms-experiments-gemini

# Back to main (planning/specs)
cd /Users/nosrcadmin/aos/ptms-experiments
```

### Check Status Across All
```bash
# View status of all worktrees
cd /Users/nosrcadmin/aos/ptms-experiments
git worktree list

# Check each variant's git status
for dir in ptms-experiments ptms-experiments-claude ptms-experiments-gpt ptms-experiments-gemini; do
  echo "=== $dir ==="
  cd /Users/nosrcadmin/aos/$dir && git status --short
done
```

---

## Workflow: LLM Variant Development

### Phase 1: Setup (DONE ✅)

```bash
# 1. Clone main repo
git clone https://github.com/MartinMayday/ptms-multi-llm-experiments.git

# 2. Create variant branches
git branch variant-claude
git branch variant-gpt
git branch variant-gemini

# 3. Add worktrees (each branch gets its own directory)
git worktree add ../ptms-experiments-claude variant-claude
git worktree add ../ptms-experiments-gpt variant-gpt
git worktree add ../ptms-experiments-gemini variant-gemini

# 4. Verify
git worktree list
```

### Phase 2: Parallel Development

Each LLM works independently in their own directory:

```
Claude Session (opencode/Claude Desktop)
  └─→ cd /Users/nosrcadmin/aos/ptms-experiments-claude
  └─→ Implements Plan 01 with Claude patterns
  └─→ Commits to variant-claude branch

GPT Session (ChatGPT/VS Code)
  └─→ cd /Users/nosrcadmin/aos/ptms-experiments-gpt
  └─→ Implements Plan 01 with GPT patterns
  └─→ Commits to variant-gpt branch

Gemini Session (Gemini API/Interface)
  └─→ cd /Users/nosrcadmin/aos/ptms-experiments-gemini
  └─→ Implements Plan 01 with Gemini patterns
  └─→ Commits to variant-gemini branch
```

### Phase 3: Validation & Testing

Each variant must pass quality gates:

```bash
# Run in EACH worktree directory
cd /Users/nosrcadmin/aos/ptms-experiments-<variant>

# 1. Type checking
bun run typecheck  # or: tsc --noEmit

# 2. Run tests
bun test

# 3. Check coverage
bun test --coverage

# 4. Lint check
bun run lint

# 5. Performance benchmarks
bun run benchmark
```

**Quality Gates:**
- ✅ All tests pass
- ✅ Coverage > 90%
- ✅ TypeScript compiles
- ✅ Linting passes
- ✅ Performance targets met

### Phase 4: Comparison & Selection

Compare implementations across variants:

```bash
# Compare file differences
diff -r ptms-experiments-claude/src ptms-experiments-gpt/src

# Compare specific components
diff ptms-experiments-claude/src/core/FileSystem.ts \
    ptms-experiments-gpt/src/core/FileSystem.ts

# View commit history for each
cd ptms-experiments-claude && git log --oneline
cd ptms-experiments-gpt && git log --oneline
cd ptms-experiments-gemini && git log --oneline
```

**Selection Criteria:**
1. **Test Coverage** - Highest coverage wins
2. **Performance** - Fastest benchmark times
3. **Code Quality** - Cleanest, most maintainable
4. **Completeness** - Implements all requirements
5. **Innovation** - Novel approaches worth keeping

### Phase 5: Merge to Main

#### Option A: Full Variant Merge (Pick One Winner)

```bash
# If Claude variant is the winner
cd /Users/nosrcadmin/aos/ptms-experiments
git checkout main
git merge variant-claude --no-ff -m "Merge: Claude variant wins for Plan 01"
git push origin main
```

#### Option B: Cherry-Pick Best Components (Hybrid Approach)

```bash
cd /Users/nosrcadmin/aos/ptms-experiments

# 1. Create integration branch
git checkout -b integration/plan-01-combined

# 2. Cherry-pick best components from each variant
# Claude's FileSystem implementation
git cherry-pick <claude-commit-hash>

# GPT's StateMachine implementation  
git cherry-pick <gpt-commit-hash>

# Gemini's test suite
git cherry-pick <gemini-commit-hash>

# 3. Resolve any conflicts
# 4. Test combined solution
# 5. Merge to main
git checkout main
git merge integration/plan-01-combined
```

#### Option C: Incremental Component Merge

```bash
# Merge component-by-component with review
# Example: Merge just the FileSystem from Claude

cd /Users/nosrcadmin/aos/ptms-experiments

# Create feature branch for this component
git checkout -b feature/filesystem-claude

# Copy specific files from Claude variant
cp ../ptms-experiments-claude/src/core/FileSystem.ts src/core/
cp ../ptms-experiments-claude/src/core/FileSystem.test.ts tests/core/

# Commit and test
git add .
git commit -m "Add: Claude's FileSystem implementation"
bun test src/core/FileSystem.test.ts

# If tests pass, merge to main
git checkout main
git merge feature/filesystem-claude
```

---

## Git Commands Reference

### Worktree Management

```bash
# List all worktrees
git worktree list

# Add new worktree
git worktree add <path> <branch>

# Remove worktree (keeps branch)
git worktree remove <path>

# Prune stale worktrees
git worktree prune

# Lock worktree (prevent auto-removal)
git worktree lock <path>

# Unlock worktree
git worktree unlock <path>
```

### Branch Management

```bash
# View all branches
git branch -a

# Create and checkout new branch
git checkout -b <new-branch>

# Delete branch (safe - only if merged)
git branch -d <branch>

# Force delete branch (unmerged commits lost!)
git branch -D <branch>

# Push branch to remote
git push -u origin <branch>
```

### Comparing Variants

```bash
# Compare two branches
git diff variant-claude variant-gpt

# Compare specific files across branches
git diff variant-claude variant-gpt -- src/core/FileSystem.ts

# View commits unique to each branch
git log variant-claude ^variant-gpt  # Claude-only commits
git log variant-gpt ^variant-claude  # GPT-only commits

# View common commits
git log variant-claude variant-gpt --not main
```

### Stashing & Temporary Changes

```bash
# Stash changes in current worktree
git stash push -m "WIP: <description>"

# List stashes
git stash list

# Apply stash to current worktree
git stash apply stash@{0}

# Pop stash (apply and remove)
git stash pop

# Drop stash
git stash drop stash@{0}
```

---

## Testing & Validation Workflow

### Automated Test Suite

Create a validation script that runs on ALL variants:

```bash
#!/bin/bash
# validate-all-variants.sh

VARIANTS=("claude" "gpt" "gemini")
RESULTS=()

for variant in "${VARIANTS[@]}"; do
  echo "========================================"
  echo "Testing variant: $variant"
  echo "========================================"
  
  cd "/Users/nosrcadmin/aos/ptms-experiments-$variant"
  
  # Run tests
  if bun test; then
    TEST_RESULT="PASS"
  else
    TEST_RESULT="FAIL"
  fi
  
  # Check coverage
  COVERAGE=$(bun test --coverage 2>&1 | grep -o '[0-9]\+\.[0-9]\+%' | head -1)
  
  # Type check
  if bun run typecheck; then
    TYPE_RESULT="PASS"
  else
    TYPE_RESULT="FAIL"
  fi
  
  RESULTS+=("$variant: Tests=$TEST_RESULT, Coverage=$COVERAGE, Types=$TYPE_RESULT")
done

# Print summary
echo ""
echo "VALIDATION SUMMARY"
echo "========================================"
for result in "${RESULTS[@]}"; do
  echo "$result"
done
```

### Performance Benchmark Comparison

```bash
#!/bin/bash
# benchmark-all-variants.sh

echo "Performance Benchmark Comparison"
echo "================================"

for variant in claude gpt gemini; do
  echo ""
  echo "Variant: $variant"
  echo "--------------------------------"
  
  cd "/Users/nosrcadmin/aos/ptms-experiments-$variant"
  
  # Run benchmarks
  bun run benchmark 2>&1 | tee "benchmarks/$variant-results.json"
done

# Generate comparison report
echo ""
echo "BENCHMARK COMPARISON"
echo "================================"
# Parse and compare results from each variant
```

---

## Best Practices

### 1. Isolation Rules

✅ **DO:**
- Each LLM works ONLY in their assigned worktree
- Commit frequently with descriptive messages
- Test before committing
- Keep commits atomic (one logical change per commit)

❌ **DON'T:**
- Work directly in `main` worktree (planning/specs only)
- Mix changes between worktrees
- Force push to shared branches
- Delete worktrees with uncommitted work

### 2. Commit Message Convention

```
<plan>-<phase>: <description>

[Claude] Implement FileSystem with atomic writes
[GPT] Refactor StateMachine transitions  
[Gemini] Add comprehensive test coverage

- Bullet point details
- Another detail
```

Examples:
- `01-02: Implement FileSystem with atomic writes [Claude]`
- `01-04: Refactor StateMachine transitions [GPT]`
- `01-07: Add 94% test coverage [Gemini]`

### 3. Sync Strategy

Keep variants synchronized with main planning updates:

```bash
# When planning files update in main
cd /Users/nosrcadmin/aos/ptms-experiments
# ... update planning files ...
git add . && git commit -m "Update: Planning files for Plan 02"
git push origin main

# Sync to all variants
cd /Users/nosrcadmin/aos/ptms-experiments-claude
git checkout variant-claude
git pull origin main  # Merge planning updates

# (Repeat for gpt and gemini variants)
```

### 4. Conflict Resolution

When merging, conflicts may occur. Resolution priority:

1. **Keep tests** - Never discard working tests
2. **Keep types** - Type safety is paramount
3. **Choose performance** - Faster implementation wins
4. **Prefer clarity** - More readable/maintainable code
5. **Document decision** - Comment why this choice was made

---

## LLM-Specific Guidelines

### Claude (ptms-experiments-claude)

**Strengths:**
- TypeScript type safety
- Comprehensive error handling
- Clean architecture patterns

**Focus Areas:**
- Strict typing (no `any`)
- Result<T,E> error types
- Atomic operations

**Entry Command:**
```bash
cd /Users/nosrcadmin/aos/ptms-experiments-claude
start-orchestrate-execution .planning/KICKOFF-PTMS.yaml
```

### GPT (ptms-experiments-gpt)

**Strengths:**
- Creative problem solving
- Comprehensive documentation
- Innovative approaches

**Focus Areas:**
- Novel implementation patterns
- Performance optimizations
- Edge case handling

**Entry Command:**
```bash
cd /Users/nosrcadmin/aos/ptms-experiments-gpt
# Use GPT-4 or GPT-4-turbo with full system prompt
```

### Gemini (ptms-experiments-gemini)

**Strengths:**
- Fast iteration
- Large context handling
- Code completion

**Focus Areas:**
- Test coverage
- Documentation completeness
- Integration scenarios

**Entry Command:**
```bash
cd /Users/nosrcadmin/aos/ptms-experiments-gemini
# Use Gemini Pro/Ultra with context
```

---

## Progress Tracking

### Current Status

| Plan | Phase | Claude | GPT | Gemini | Winner | Merged |
|------|-------|--------|-----|--------|--------|--------|
| 01 | 01 | 🔄 | 🔄 | 🔄 | - | No |
| 01 | 02 | 🔄 | 🔄 | 🔄 | - | No |
| 01 | 03 | 🔄 | 🔄 | 🔄 | - | No |
| 01 | 04 | ⏳ | ⏳ | ⏳ | - | No |
| 01 | 05 | ⏳ | ⏳ | ⏳ | - | No |
| 01 | 06 | ⏳ | ⏳ | ⏳ | - | No |
| 01 | 07 | ⏳ | ⏳ | ⏳ | - | No |

Legend:
- ✅ Complete
- 🔄 In Progress
- ⏳ Pending
- ❌ Failed

### Update Progress

Edit this file to track progress:

```bash
cd /Users/nosrcadmin/aos/ptms-experiments
vim MULTI-LLM-WORKFLOW.md  # Update status table

git add MULTI-LLM-WORKFLOW.md
git commit -m "Update: Plan 01 Phase 02 - Claude complete"
git push origin main
```

---

## Emergency Procedures

### Worktree Corruption

```bash
# If worktree becomes corrupted:
cd /Users/nosrcadmin/aos/ptms-experiments

# 1. Remove corrupted worktree
git worktree remove ../ptms-experiments-claude --force

# 2. Recreate from branch
git worktree add ../ptms-experiments-claude variant-claude

# 3. Verify
git worktree list
```

### Lost Commits

```bash
# Find lost commits
git reflog

# Recover specific commit
git checkout -b recovery-branch <commit-hash>

# Cherry-pick to main branch
git checkout main
git cherry-pick <commit-hash>
```

### Merge Conflicts

```bash
# Abort merge and start over
git merge --abort

# Or resolve manually:
# 1. Edit conflicted files
# 2. git add <resolved-files>
# 3. git commit
```

---

## Summary

### What You Have Now

✅ **GitHub Repository:** https://github.com/MartinMayday/ptms-multi-llm-experiments  
✅ **Main Branch:** Planning files and specifications  
✅ **3 Variant Branches:** Claude, GPT, Gemini  
✅ **4 Worktrees:** Isolated working directories  
✅ **This Documentation:** Complete workflow guide

### Next Steps

1. **Start LLM Sessions:**
   - Open Claude Desktop → cd ptms-experiments-claude
   - Open ChatGPT → cd ptms-experiments-gpt
   - Open Gemini → cd ptms-experiments-gemini

2. **Begin Plan 01:**
   - Each LLM implements Phase 01-01 independently
   - Commit to respective branch
   - No interference between variants

3. **Validate & Compare:**
   - Run test suites on all variants
   - Compare performance benchmarks
   - Document differences

4. **Merge Best Components:**
   - Cherry-pick winning implementations
   - Create hybrid solution
   - Maintain test coverage

### Support

- **Git Issues:** Check `.git/config` and remote settings
- **Worktree Issues:** Run `git worktree prune` to clean up
- **Merge Conflicts:** See Emergency Procedures above
- **Questions:** Refer to this document or main SUMMARY.md

---

**Ready to orchestrate parallel LLM development!**

```bash
# Verify setup
cd /Users/nosrcadmin/aos/ptms-experiments
git worktree list
git branch -a
```

Each LLM can now work independently without conflicts. Let the experiments begin! 🚀
