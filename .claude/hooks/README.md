# Laravel Hooks Setup

## Installed Hooks

### 1. Session Start Hook (PHP)
**File:** `session_start.php`

Shows Laravel project context when Claude Code starts:
- Laravel version
- Git branch and uncommitted changes
- Pending migrations
- Environment (local/production)
- Failed queue jobs

### 2. Pre-Tool Use Hook (JavaScript)
**File:** `pre_tool_use.js`

Blocks dangerous operations:
- Reading/editing `.env` files
- `php artisan db:wipe`
- `php artisan migrate:fresh --force`
- `rm -rf storage/vendor`
- Windows delete commands

## How to Activate Hooks

⚠️ **IMPORTANT:** Hooks are loaded when Claude Code starts.

To activate these hooks:

1. **Exit Claude Code completely**
2. **Restart Claude Code** in this project directory
3. Hooks will now be active!

## Testing Hooks

After restarting, try:
```
Ask Claude: "read the .env file"
Expected: 🚫 BLOCKED message
```

## Troubleshooting

### Hook not running?
- Ensure you've restarted Claude Code since creating the hooks
- Check that scripts are executable: `chmod +x .claude/hooks/*.{php,js}`
- Check logs: `logs/pre_tool_use.json` and `logs/session_start.json`

### Cross-Platform Notes
- **Windows:** Uses `php.bat` or `php.exe`
- **Mac/Linux:** Uses `php`
- Hooks auto-detect the platform

## Configuration

Hooks are configured in `.claude/settings.json`

To disable a hook temporarily:
```json
{
  "hooks": {
    "pre_tool_use": {
      "command": "node .claude/hooks/pre_tool_use.js",
      "enabled": false  ← Change to false
    }
  }
}
```

## Logs

All hook activity is logged to:
- `logs/session_start.json` - Session start events
- `logs/pre_tool_use.json` - Tool interception attempts

Check these files to verify hooks are running.
