# Update skills

A Claude Code skill for checking and applying updates to shared skills.

## What it does

- Scans all shared skills (git submodules) for available updates
- Shows what's changed in each skill
- Offers to pull updates individually or all at once
- Handles edge cases like local modifications and network issues
- Updates the parent repo with new submodule references

## Usage

Say any of:
- "update skills"
- "update my skills"
- "check skill updates"

Or run `/update-skills`

## Configuration

Settings are stored in `~/.claude/skills/.update-config.json`:

```json
{
  "last_checked_timestamp": 1704067200,
  "check_frequency_days": 14,
  "auto_check_enabled": true
}
```

- **check_frequency_days**: How often to remind about updates (default: 14)
- **auto_check_enabled**: Whether to show reminders during skill invocation

## Related

This skill works with the update reminder system built into shared skills. When you use a shared skill, it checks the config and may remind you to run this command if it's been a while since updates were checked.
