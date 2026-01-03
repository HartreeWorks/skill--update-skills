---
name: check-skill-updates
description: Check for available updates to shared Claude Code skills. Use when the user says "check skill updates", "update skills", "are there skill updates", "update my skills", "/check-skill-updates", or asks about updating their Claude Code skills. Also use for configuring update preferences (frequency, enable/disable).
---

# Check skill updates

Check for and apply updates to shared skills (git submodules).

## Prerequisites

- Git installed
- Network access (for fetching from remotes)

## Configuration

Config file: `~/.claude/skills/.update-config.json`

```json
{
  "last_checked_timestamp": 0,
  "check_frequency_days": 14,
  "auto_check_enabled": true
}
```

## Workflow

### Step 1: Read current configuration

```bash
cat ~/.claude/skills/.update-config.json 2>/dev/null || echo '{"last_checked_timestamp": 0, "check_frequency_days": 14, "auto_check_enabled": true}'
```

### Step 2: Identify shared skills

Get the list of shared skills from `.gitmodules`:

```bash
cd ~/.claude/skills && grep "path = " .gitmodules | awk '{print $3}'
```

### Step 3: Check each skill for updates

For each shared skill, run these commands:

```bash
SKILL_NAME="skill-name"
cd ~/.claude/skills/$SKILL_NAME

# Fetch latest from remote
git fetch origin 2>&1

# Count commits behind
BEHIND=$(git rev-list HEAD..origin/main --count 2>/dev/null || git rev-list HEAD..origin/master --count 2>/dev/null || echo "0")

# If behind, show what's new
if [ "$BEHIND" -gt 0 ]; then
  echo "=== $SKILL_NAME: $BEHIND update(s) available ==="
  git log HEAD..origin/main --oneline --format="  - %s (%ar)" 2>/dev/null || git log HEAD..origin/master --oneline --format="  - %s (%ar)"
else
  echo "=== $SKILL_NAME: up to date ==="
fi

# Check for local modifications
LOCAL_CHANGES=$(git status --porcelain)
if [ -n "$LOCAL_CHANGES" ]; then
  echo "  ⚠️ Local modifications detected"
fi
```

### Step 4: Present results

Format the results as a summary:

```markdown
## Skill update check

**Checked:** X shared skills
**Updates available:** Y

### Skills with updates

| Skill | Updates | Latest change |
|-------|---------|---------------|
| skill-name | N commits | Commit message (time ago) |

### Skills up to date
- skill1
- skill2
```

### Step 5: Handle user preferences

If updates are available, use `AskUserQuestion` tool with options:
- "Update all" - Pull updates for all skills with updates
- "Choose which to update" - Let user select individual skills
- "Skip for now" - Don't update, but mark as checked

If user wants to change settings, offer:
- Change check frequency (current: X days)
- Enable/disable automatic reminders

### Step 6: Apply updates (if requested)

For each skill the user wants to update:

```bash
SKILL_NAME="skill-name"
cd ~/.claude/skills/$SKILL_NAME

# Check for local modifications
if [ -n "$(git status --porcelain)" ]; then
  echo "Stashing local changes in $SKILL_NAME..."
  git stash push -m "Auto-stash before skill update $(date +%Y-%m-%d)"
  STASHED=true
fi

# Pull updates
git pull origin main 2>/dev/null || git pull origin master

# Report stashed changes
if [ "$STASHED" = true ]; then
  echo "Note: Local changes were stashed. Run 'git stash pop' in $SKILL_NAME to restore."
fi
```

After updating skills, update the parent repo reference:

```bash
cd ~/.claude/skills
git add $SKILL_NAME
```

If any skills were updated, commit the parent repo:

```bash
cd ~/.claude/skills
git commit -m "Update shared skills

Updated: skill1, skill2

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

### Step 7: Update configuration

After checking (whether or not updates were applied):

```bash
# Get current timestamp
TIMESTAMP=$(date +%s)

# Update config file
cat > ~/.claude/skills/.update-config.json << EOF
{
  "last_checked_timestamp": $TIMESTAMP,
  "check_frequency_days": 14,
  "auto_check_enabled": true
}
EOF
```

Preserve any user customisations to `check_frequency_days` and `auto_check_enabled`.

## Changing settings

If user asks to change update settings:

### Disable automatic reminders
```bash
# Read current config, set auto_check_enabled to false
```

### Change check frequency
```bash
# Read current config, update check_frequency_days to new value
```

## Edge cases

### Network unavailable
If `git fetch` fails for a skill:
- Note which skills couldn't be checked
- Continue checking others
- Report at the end: "Could not check: skill1, skill2 (network unavailable)"

### Local modifications
If a skill has uncommitted changes:
- Warn the user before updating
- Offer to stash changes
- After update, remind user they can restore with `git stash pop`

### Merge conflicts
If `git pull` fails:
- Run `git merge --abort` to cancel
- Report the conflict to user
- Suggest manual resolution: "cd ~/.claude/skills/SKILL_NAME && git status"

### Diverged history
If the skill has local commits not on remote:
```bash
LOCAL_ONLY=$(git rev-list origin/main..HEAD --count 2>/dev/null || echo "0")
if [ "$LOCAL_ONLY" -gt 0 ]; then
  echo "⚠️ $SKILL_NAME has $LOCAL_ONLY local commit(s) not on remote"
fi
```
Warn user and ask if they still want to pull (may cause merge).

## Output

After completing the check:
1. Show summary of what was checked
2. Show what was updated (if anything)
3. Confirm config timestamp was updated
4. Remind about any stashed changes
