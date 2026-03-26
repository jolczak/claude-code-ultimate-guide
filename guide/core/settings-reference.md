# Claude Code Settings Reference

> Complete reference for `settings.json` configuration and environment variables. Covers all confirmed settings as of Claude Code v2.1.81.

**Sources:** [Official settings docs](https://code.claude.com/docs/en/settings) · [Official env-vars docs](https://code.claude.com/docs/en/env-vars) · [JSON Schema](https://json.schemastore.org/claude-code-settings.json)

**Legend:**
- No badge = confirmed in official documentation
- `📋 Schema only` = present in JSON schema but not on official settings page — still valid
- `⚠️ Unverified` = not confirmed in official sources

---

## Scope and Precedence

Claude Code uses four settings scopes, applied from highest to lowest priority:

| Priority | Scope | Location | Shared? | Purpose |
|----------|-------|----------|---------|---------|
| 1 | **Managed** | Server, MDM profile, registry, or system `managed-settings.json` | Yes (IT-deployed) | Org-wide policies, cannot be overridden |
| 2 | **Command line** | `--` flags at startup | No | Temporary session overrides |
| 3 | **Local** | `.claude/settings.local.json` | No (gitignored) | Personal project-specific |
| 4 | **Project** | `.claude/settings.json` | Yes (committed) | Team-shared settings |
| 5 | **User** | `~/.claude/settings.json` | No | Global personal defaults |

**Array merging:** Settings like `permissions.allow`, `sandbox.filesystem.allowWrite`, and `allowedHttpHookUrls` are concatenated and deduplicated across scopes — not replaced.

**Deny precedence:** `permissions.deny` rules always take effect regardless of allow/ask rules at any scope.

**Managed settings delivery methods:**
- Server-managed (Claude.ai admin console)
- macOS MDM: `com.anthropic.claudecode` plist
- Windows registry: `HKLM\SOFTWARE\Policies\ClaudeCode`
- File: `managed-settings.json` at `/Library/Application Support/ClaudeCode/` (macOS), `/etc/claude-code/` (Linux/WSL), `C:\Program Files\ClaudeCode\` (Windows)
- Drop-in directory: `managed-settings.d/*.json` alongside `managed-settings.json`, merged alphabetically

**Other config:** `~/.claude.json` stores OAuth session, MCP server configs, per-project trust state, and preferences like `editorMode`. Do not put `~/.claude.json` keys into `settings.json` — it will trigger schema validation errors.

---

## Settings Keys

### Core Configuration

#### `$schema`
**Type:** string
**Scope:** all
**Default:** none

JSON Schema URL for IDE validation and autocomplete. Add `"https://json.schemastore.org/claude-code-settings.json"` to enable inline validation in VS Code, Cursor, and other editors.

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json"
}
```

#### `model`
**Type:** string
**Scope:** all
**Default:** `"default"`

Override the default model for all sessions. Accepts aliases (`"sonnet"`, `"opus"`, `"haiku"`, `"opusplan"`) or full model IDs like `"claude-sonnet-4-6"`. The `ANTHROPIC_MODEL` environment variable takes precedence.

```json
{ "model": "opus" }
```

#### `agent`
**Type:** string
**Scope:** all
**Default:** none

Run the main thread as a named subagent. Applies that agent's system prompt, tool restrictions, and model. Value must match an agent defined in `.claude/agents/`. Also available via `--agent` CLI flag.

```json
{ "agent": "code-reviewer" }
```

#### `language`
**Type:** string
**Scope:** all
**Default:** `"english"`

Claude's preferred response language. Also sets the voice dictation language. Examples: `"japanese"`, `"spanish"`, `"french"`.

#### `cleanupPeriodDays`
**Type:** number
**Scope:** all
**Default:** `30`

Sessions inactive longer than this number of days are deleted at startup. Setting to `0` deletes all existing transcripts at startup and disables session persistence entirely — no `.jsonl` files are written, `/resume` shows no conversations, and hooks receive an empty `transcript_path`.

#### `autoUpdatesChannel`
**Type:** string
**Scope:** all
**Default:** `"latest"`
**Values:** `"latest"` | `"stable"`

Release channel to follow. `"stable"` is typically about one week behind `"latest"` and skips versions with major regressions.

#### `alwaysThinkingEnabled`
**Type:** boolean
**Scope:** all
**Default:** `false`

Enable extended thinking by default for all sessions. Usually configured via `/config` rather than editing directly.

#### `includeGitInstructions`
**Type:** boolean
**Scope:** all
**Default:** `true`

Include built-in commit and PR workflow instructions and a git status snapshot in the system prompt. Set to `false` when using a custom git workflow skill. The `CLAUDE_CODE_DISABLE_GIT_INSTRUCTIONS` env var takes precedence.

#### `voiceEnabled`
**Type:** boolean
**Scope:** user
**Default:** none

Enable push-to-talk voice dictation. Written automatically when you run `/voice`. Requires a Claude.ai account.

#### `companyAnnouncements`
**Type:** array of strings
**Scope:** all
**Default:** none

Announcements displayed to users at startup. Multiple announcements are cycled through at random.

```json
{
  "companyAnnouncements": [
    "Welcome to Acme Corp! Review code guidelines at docs.acme.com",
    "All PRs require code review before merge"
  ]
}
```

#### `availableModels`
**Type:** array of strings
**Scope:** all
**Default:** none

Restrict which models users can select via `/model`, `--model`, Config tool, or `ANTHROPIC_MODEL`. Does not affect the Default option.

```json
{ "availableModels": ["sonnet", "haiku"] }
```

#### `fastModePerSessionOptIn`
**Type:** boolean
**Scope:** all
**Default:** `false`

When `true`, fast mode does not persist across sessions. Each session starts with fast mode off, requiring users to enable it with `/fast`. The user's preference is still saved.

#### `teammateMode`
**Type:** string
**Scope:** all
**Default:** `"auto"`
**Values:** `"auto"` | `"in-process"` | `"tmux"`

How agent team teammates display. `"auto"` uses split panes in tmux or iTerm2, in-process otherwise.

#### `showClearContextOnPlanAccept`
**Type:** boolean
**Scope:** all
**Default:** `false`

Show the "clear context" option on the plan accept screen. Set to `true` to restore the option, which was hidden by default starting in v2.1.81.

#### `feedbackSurveyRate`
**Type:** number
**Scope:** all
**Default:** none

Probability (0–1) that the session quality survey appears when eligible. Set to `0` to suppress entirely. Useful when using Bedrock, Vertex, or Foundry.

#### `disableAutoMode`
**Type:** string
**Scope:** all
**Default:** none
**Values:** `"disable"`

Set to `"disable"` to prevent auto mode from being activated. Removes `auto` from the `Shift+Tab` cycle and rejects `--permission-mode auto` at startup.

#### `useAutoModeDuringPlan`
**Type:** boolean
**Scope:** user / local
**Default:** `true`

Whether plan mode uses auto mode semantics when auto mode is available. Not read from shared project settings.

#### `autoMode`
**Type:** object
**Scope:** user / local
**Default:** none

Customize the auto mode classifier. Contains `environment`, `allow`, and `soft_deny` arrays of prose rules. Not read from shared project settings.

#### `defaultShell`
**Type:** string
**Scope:** all
**Default:** `"bash"`
**Values:** `"bash"` | `"powershell"`

Default shell for input-box `!` commands. `"powershell"` requires `CLAUDE_CODE_USE_POWERSHELL_TOOL=1`.

#### `skipWebFetchPreflight` `📋 Schema only`
**Type:** boolean
**Scope:** all
**Default:** `false`

Skip the WebFetch blocklist check before fetching URLs.

#### `env`
**Type:** object
**Scope:** all
**Default:** none

Environment variables applied to every session. Use this instead of wrapper scripts to set variables. See the [Environment Variables](#environment-variables) section for all supported keys.

```json
{
  "env": {
    "NODE_ENV": "development",
    "CLAUDE_CODE_EFFORT_LEVEL": "medium"
  }
}
```

---

### Plans and Memory

#### `plansDirectory`
**Type:** string
**Scope:** all
**Default:** `"~/.claude/plans"`

Directory where `/plan` outputs are stored. Path is relative to the project root.

#### `autoMemoryEnabled`
**Type:** boolean
**Scope:** all
**Default:** `true`

Enable or disable the auto-memory feature that automatically saves context across sessions.

#### `autoMemoryDirectory`
**Type:** string
**Scope:** user / local / managed
**Default:** none

Custom directory for auto-memory storage. Accepts `~/`-expanded paths. Not accepted in project settings (`.claude/settings.json`) to prevent shared repos from redirecting memory writes to sensitive locations.

---

### Permissions

Control what tools and operations Claude can perform.

#### `permissions.allow`
**Type:** array of strings
**Scope:** all
**Default:** `[]`

Permission rules that allow tool use without prompting. Arrays are concatenated across scopes. See [Permission Rule Syntax](#permission-rule-syntax) below.

#### `permissions.ask`
**Type:** array of strings
**Scope:** all
**Default:** `[]`

Permission rules requiring user confirmation before tool use.

#### `permissions.deny`
**Type:** array of strings
**Scope:** all
**Default:** `[]`

Permission rules blocking tool use. Highest safety precedence — cannot be overridden by allow/ask rules at any scope.

#### `permissions.additionalDirectories`
**Type:** array of strings
**Scope:** all
**Default:** `[]`

Additional working directories that Claude has access to, beyond the current project root.

```json
{ "permissions": { "additionalDirectories": ["../shared-libs/"] } }
```

#### `permissions.defaultMode`
**Type:** string
**Scope:** all
**Default:** `"default"`
**Values:** `"default"` | `"acceptEdits"` | `"plan"` | `"bypassPermissions"`

Default permission mode when opening Claude Code. In Remote environments, only `"acceptEdits"` and `"plan"` are honored.

#### `permissions.disableBypassPermissionsMode`
**Type:** string
**Scope:** all
**Default:** none
**Values:** `"disable"`

Set to `"disable"` to prevent `bypassPermissions` mode from being activated. Disables the `--dangerously-skip-permissions` flag. Most useful in managed settings.

#### `allowManagedPermissionRulesOnly`
**Type:** boolean
**Scope:** managed only
**Default:** `false`

When `true`, user and project `allow`, `ask`, and `deny` rules are ignored. Only managed permission rules apply.

### Permission Rule Syntax

Rules follow the format `Tool` or `Tool(specifier)`. Evaluation order: deny first, then ask, then allow. The first matching rule wins.

| Tool | Pattern | Example |
|------|---------|---------|
| `Bash` | Command pattern with wildcards | `Bash(npm run *)`, `Bash(git *)` |
| `Read` | File path pattern | `Read(.env)`, `Read(./secrets/**)` |
| `Edit` | File path pattern | `Edit(src/**)`, `Edit(*.ts)` |
| `Write` | File path pattern | `Write(*.md)` |
| `WebFetch` | `domain:hostname` | `WebFetch(domain:example.com)` |
| `WebSearch` | No specifier | `WebSearch` |
| `Task` | Agent name | `Task(Explore)` |
| `Agent` | Agent name | `Agent(researcher)` |
| `MCP` | `mcp__server__tool` or `MCP(server:tool)` | `mcp__memory__*` |

**Path prefixes for Read/Edit rules:**

| Prefix | Meaning |
|--------|---------|
| `//` | Absolute path from filesystem root |
| `~/` | Relative to home directory |
| `/` | Relative to project root |
| `./` or none | Relative path (current directory) |

**Bash wildcard notes:** `*` matches at any position. `Bash(ls *)` (space before `*`) matches `ls -la` but NOT `lsof`. `Bash(*)` is equivalent to `Bash` (matches all commands). The legacy `:*` suffix (e.g., `Bash(npm:*)`) is deprecated.

```json
{
  "permissions": {
    "allow": ["Edit(*)", "Bash(npm run *)", "Bash(git *)"],
    "ask": ["Bash(git push *)"],
    "deny": ["Read(.env)", "Read(./secrets/**)"],
    "defaultMode": "acceptEdits"
  }
}
```

---

### Hooks

#### `hooks`
**Type:** object
**Scope:** all
**Default:** none

Configure custom commands to run at lifecycle events. See the [hooks documentation](https://code.claude.com/docs/en/hooks) for full format, all 19 events, exit codes, and environment variables.

#### `disableAllHooks`
**Type:** boolean
**Scope:** all
**Default:** `false`

Disable all hooks and any custom status line.

#### `allowManagedHooksOnly`
**Type:** boolean
**Scope:** managed only
**Default:** `false`

When `true`, only managed hooks and SDK hooks are loaded. User, project, and plugin hooks are blocked.

#### `allowedHttpHookUrls`
**Type:** array of strings
**Scope:** all
**Default:** none (no restriction)

Allowlist of URL patterns that HTTP hooks may target. Supports `*` as a wildcard. When defined, hooks with non-matching URLs are silently blocked. Empty array blocks all HTTP hooks. Arrays merge across settings sources.

```json
{ "allowedHttpHookUrls": ["https://hooks.example.com/*"] }
```

#### `httpHookAllowedEnvVars`
**Type:** array of strings
**Scope:** all
**Default:** none (no restriction)

Allowlist of environment variable names that HTTP hooks can interpolate into header values. Each hook's effective `allowedEnvVars` is the intersection with this list. Arrays merge across settings sources.

---

### MCP Servers

#### `enableAllProjectMcpServers`
**Type:** boolean
**Scope:** all
**Default:** `false`

Automatically approve all MCP servers defined in project `.mcp.json` files. Avoids per-server confirmation prompts.

#### `enabledMcpjsonServers`
**Type:** array of strings
**Scope:** all
**Default:** none

Allowlist of specific server names from `.mcp.json` files to approve.

#### `disabledMcpjsonServers`
**Type:** array of strings
**Scope:** all
**Default:** none

Blocklist of specific server names from `.mcp.json` files to reject.

#### `allowedMcpServers`
**Type:** array
**Scope:** managed only
**Default:** none (no restrictions)

Allowlist of MCP servers users can configure. Each entry matches by `serverName`, `serverCommand`, or `serverUrl`. Undefined = no restrictions, empty array = lockdown.

```json
{
  "allowedMcpServers": [
    { "serverName": "github" },
    { "serverCommand": "npx @modelcontextprotocol/*" },
    { "serverUrl": "https://mcp.company.com/*" }
  ]
}
```

#### `deniedMcpServers`
**Type:** array
**Scope:** managed only
**Default:** none

Blocklist of MCP servers that are explicitly blocked. Applies to all scopes including managed servers. Takes precedence over `allowedMcpServers`.

#### `allowManagedMcpServersOnly`
**Type:** boolean
**Scope:** managed only
**Default:** `false`

When `true`, only `allowedMcpServers` from managed settings are respected. Users can still add MCP servers, but only admin-defined servers are usable. `deniedMcpServers` still merges from all sources.

#### `channelsEnabled`
**Type:** boolean
**Scope:** managed only
**Default:** `false`

Allow channels for Team and Enterprise users. When unset or `false`, channel message delivery is blocked regardless of what users pass to `--channels`.

#### `allowedChannelPlugins`
**Type:** array
**Scope:** managed only
**Default:** none (uses default Anthropic allowlist)

Allowlist of channel plugins that may push messages. Replaces the default Anthropic allowlist when set. Requires `channelsEnabled: true`. Empty array blocks all channel plugins.

---

### Sandbox

Configure bash command sandboxing for security. Available on macOS, Linux, and WSL2.

#### `sandbox.enabled`
**Type:** boolean
**Scope:** all
**Default:** `false`

Enable bash sandboxing. Isolates bash commands from your filesystem and network.

#### `sandbox.failIfUnavailable`
**Type:** boolean
**Scope:** all
**Default:** `false`

Exit with an error at startup if `sandbox.enabled` is `true` but the sandbox cannot start (missing dependencies, unsupported platform). When `false`, a warning is shown and commands run unsandboxed. Useful in managed deployments that require sandboxing as a hard gate.

#### `sandbox.autoAllowBashIfSandboxed`
**Type:** boolean
**Scope:** all
**Default:** `true`

Auto-approve bash commands when sandboxed. When the sandbox is active, bash commands that would normally require confirmation are automatically approved.

#### `sandbox.excludedCommands`
**Type:** array of strings
**Scope:** all
**Default:** `[]`

Commands that bypass the sandbox and run directly in your environment.

#### `sandbox.allowUnsandboxedCommands`
**Type:** boolean
**Scope:** all
**Default:** `true`

Allow commands to opt out of the sandbox via the `dangerouslyDisableSandbox` parameter. Set to `false` for strict sandboxing where all commands must run inside the sandbox or be in `excludedCommands`.

#### `sandbox.enableWeakerNestedSandbox`
**Type:** boolean
**Scope:** all
**Default:** `false`

Enable a weaker sandbox for unprivileged Docker environments (Linux and WSL2 only). Reduces security.

#### `sandbox.enableWeakerNetworkIsolation`
**Type:** boolean
**Scope:** all
**Default:** `false`

(macOS only) Allow access to the system TLS trust service (`com.apple.trustd.agent`). Required for Go-based tools like `gh`, `gcloud`, and `terraform` when using `httpProxyPort` with a MITM proxy and custom CA. Reduces security.

#### `sandbox.network.allowUnixSockets`
**Type:** array of strings
**Scope:** all
**Default:** `[]`

Specific Unix socket paths accessible in the sandbox (for SSH agents, Docker, etc.).

#### `sandbox.network.allowAllUnixSockets`
**Type:** boolean
**Scope:** all
**Default:** `false`

Allow all Unix socket connections in the sandbox. Overrides `allowUnixSockets`.

#### `sandbox.network.allowLocalBinding`
**Type:** boolean
**Scope:** all
**Default:** `false`

Allow binding to localhost ports (macOS only).

#### `sandbox.network.allowedDomains`
**Type:** array of strings
**Scope:** all
**Default:** `[]`

Domains allowed for outbound network traffic. Supports wildcards like `*.example.com`.

#### `sandbox.network.allowManagedDomainsOnly`
**Type:** boolean
**Scope:** managed only
**Default:** `false`

When `true`, only `allowedDomains` and `WebFetch(domain:...)` allow rules from managed settings are respected. Non-allowed domains are blocked without prompting. Denied domains are still respected from all sources.

#### `sandbox.network.httpProxyPort`
**Type:** number
**Scope:** all
**Default:** none

HTTP proxy port for a custom proxy (1–65535). If not specified, Claude runs its own proxy.

#### `sandbox.network.socksProxyPort`
**Type:** number
**Scope:** all
**Default:** none

SOCKS5 proxy port for a custom proxy (1–65535).

#### `sandbox.network.deniedDomains` `⚠️ Unverified`
**Type:** array of strings
**Scope:** all
**Default:** `[]`

Network domain denylist for the sandbox. Not confirmed in official documentation.

#### `sandbox.filesystem.allowWrite`
**Type:** array of strings
**Scope:** all
**Default:** `[]`

Additional paths where sandboxed commands can write. Merged across all settings scopes. Also merged with paths from `Edit(...)` allow permission rules.

**Path prefix conventions:** `/` = absolute, `~/` = home-relative, `./` or no prefix = project-relative in project settings / `~/.claude`-relative in user settings.

#### `sandbox.filesystem.denyWrite`
**Type:** array of strings
**Scope:** all
**Default:** `[]`

Paths where sandboxed commands cannot write. Merged with `Edit(...)` deny rules.

#### `sandbox.filesystem.denyRead`
**Type:** array of strings
**Scope:** all
**Default:** `[]`

Paths where sandboxed commands cannot read. Merged with `Read(...)` deny rules.

#### `sandbox.filesystem.allowRead`
**Type:** array of strings
**Scope:** all
**Default:** `[]`

Paths to re-allow read access within `denyRead` regions. Takes precedence over `denyRead`. Arrays merge across all settings scopes.

#### `sandbox.filesystem.allowManagedReadPathsOnly`
**Type:** boolean
**Scope:** managed only
**Default:** `false`

When `true`, only `allowRead` paths from managed settings are respected. `allowRead` entries from user, project, and local settings are ignored.

**Sandbox example:**
```json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "excludedCommands": ["git", "docker"],
    "filesystem": {
      "allowWrite": ["/tmp/build", "~/.kube"],
      "denyRead": ["~/.aws/credentials"]
    },
    "network": {
      "allowedDomains": ["github.com", "*.npmjs.org"],
      "allowUnixSockets": ["/var/run/docker.sock"],
      "allowLocalBinding": true
    }
  }
}
```

---

### Plugins and Marketplaces

#### `enabledPlugins`
**Type:** object
**Scope:** all
**Default:** none

Enable or disable specific plugins by key (format: `plugin-name@marketplace-name`).

```json
{
  "enabledPlugins": {
    "formatter@acme-tools": true,
    "experimental@acme-tools": false
  }
}
```

#### `extraKnownMarketplaces`
**Type:** object
**Scope:** project
**Default:** none

Add custom plugin marketplaces. Use `source: "settings"` to declare plugins inline without hosting a repository.

#### `strictKnownMarketplaces`
**Type:** array
**Scope:** managed only
**Default:** none (no restrictions)

Allowlist of permitted plugin marketplaces. When set, users can only add plugins from listed marketplaces. Empty array blocks all additions.

#### `blockedMarketplaces`
**Type:** array
**Scope:** managed only
**Default:** none

Block specific plugin marketplace sources. Blocked sources are checked before downloading, so they never touch the filesystem.

#### `pluginTrustMessage`
**Type:** string
**Scope:** managed only
**Default:** none

Custom message appended to the plugin trust warning shown before installation. Use for org-specific context like confirming plugins from an internal marketplace are vetted.

#### `skippedMarketplaces` `📋 Schema only`
**Type:** array
**Scope:** all
**Default:** none

Marketplaces the user declined to install (stored automatically).

#### `skippedPlugins` `📋 Schema only`
**Type:** array
**Scope:** all
**Default:** none

Plugins the user declined to install (stored automatically).

#### `pluginConfigs` `📋 Schema only`
**Type:** object
**Scope:** all
**Default:** none

Per-plugin MCP server configurations, keyed by `plugin@marketplace`.

---

### Model Configuration

#### `effortLevel`
**Type:** string
**Scope:** all
**Default:** `"medium"`
**Values:** `"low"` | `"medium"` | `"high"`

Persist the effort level across sessions. Controls reasoning depth. Written automatically when you run `/effort low|medium|high`. Supported on Opus 4.6 and Sonnet 4.6. The `CLAUDE_CODE_EFFORT_LEVEL` env var takes precedence.

#### `modelOverrides`
**Type:** object
**Scope:** all
**Default:** none

Map Anthropic model IDs to provider-specific model IDs (e.g., Bedrock inference profile ARNs). Each key is a model picker entry name; each value is the provider model ID.

```json
{
  "modelOverrides": {
    "claude-opus-4-6": "arn:aws:bedrock:us-east-1:123456789:inference-profile/anthropic.claude-opus-4-6-v1:0"
  }
}
```

**Model aliases reference:**

| Alias | Description |
|-------|-------------|
| `"default"` | Recommended model for your account type |
| `"sonnet"` | Latest Sonnet (Claude Sonnet 4.6) |
| `"opus"` | Latest Opus (Claude Opus 4.6) |
| `"haiku"` | Fast Haiku model |
| `"sonnet[1m]"` | Sonnet with 1M token context |
| `"opusplan"` | Opus for planning, Sonnet for execution |

---

### Display and UX

#### `statusLine`
**Type:** object
**Scope:** all
**Default:** none

Configure a custom status line. The command receives a JSON object on stdin with fields like `context_window.used_percentage`, `rate_limits.five_hour.used_percentage`, etc.

```json
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/statusline.sh",
    "padding": 0
  }
}
```

#### `fileSuggestion`
**Type:** object
**Scope:** all
**Default:** none

Configure a custom script for `@` file path autocomplete. The command receives JSON on stdin with a `query` field and outputs newline-separated file paths (max 15).

```json
{
  "fileSuggestion": {
    "type": "command",
    "command": "~/.claude/file-suggestion.sh"
  }
}
```

#### `outputStyle`
**Type:** string
**Scope:** all
**Default:** `"default"`

Configure an output style to adjust the system prompt. Example: `"Explanatory"`.

#### `spinnerTipsEnabled`
**Type:** boolean
**Scope:** all
**Default:** `true`

Show tips in the spinner while Claude is working.

#### `spinnerVerbs`
**Type:** object
**Scope:** all
**Default:** none

Customize the action verbs shown in the spinner and turn duration messages. Set `mode` to `"replace"` to use only your verbs, or `"append"` to add to defaults.

```json
{
  "spinnerVerbs": {
    "mode": "replace",
    "verbs": ["Cooking", "Brewing", "Crafting", "Conjuring"]
  }
}
```

#### `spinnerTipsOverride`
**Type:** object
**Scope:** all
**Default:** none

Override spinner tips with custom strings. `tips`: array of strings. `excludeDefault`: when `true`, only show custom tips.

```json
{
  "spinnerTipsOverride": {
    "tips": ["Use /compact at 50% context", "Plan mode helps for complex tasks"],
    "excludeDefault": true
  }
}
```

#### `respectGitignore`
**Type:** boolean
**Scope:** all
**Default:** `true`

Control whether the `@` file picker respects `.gitignore` patterns.

#### `prefersReducedMotion`
**Type:** boolean
**Scope:** all
**Default:** `false`

Reduce or disable UI animations (spinners, shimmer, flash effects) for accessibility.

---

### Authentication

#### `apiKeyHelper`
**Type:** string
**Scope:** all
**Default:** none

Shell script path (executed in `/bin/sh`) that outputs an auth token sent as `X-Api-Key` and `Authorization: Bearer` headers for model requests. Useful for short-lived credentials.

```json
{ "apiKeyHelper": "/bin/generate_temp_api_key.sh" }
```

#### `forceLoginMethod`
**Type:** string
**Scope:** all
**Default:** none
**Values:** `"claudeai"` | `"console"`

Restrict login to Claude.ai accounts (`"claudeai"`) or Claude Console API-billing accounts (`"console"`).

#### `forceLoginOrgUUID`
**Type:** string
**Scope:** all
**Default:** none

UUID of an organization to automatically select during login, bypassing the org selection step. Requires `forceLoginMethod` to be set.

---

### Attribution

#### `attribution.commit`
**Type:** string
**Scope:** all
**Default:** Git trailer with co-authored-by line

Attribution text added to git commits. Supports git trailers. Set to empty string to disable commit attribution entirely.

#### `attribution.pr`
**Type:** string
**Scope:** all
**Default:** Generated message with Claude Code link

Attribution text added to pull request descriptions. Set to empty string to disable.

#### `includeCoAuthoredBy`
**Type:** boolean
**Scope:** all
**Default:** `true`
**Status:** DEPRECATED — use `attribution` instead

Whether to include the `Co-Authored-By` byline. Superseded by the `attribution` object.

```json
{
  "attribution": {
    "commit": "Generated with AI\n\nCo-Authored-By: AI <ai@example.com>",
    "pr": ""
  }
}
```

---

### Worktrees

#### `worktree.symlinkDirectories`
**Type:** array of strings
**Scope:** all
**Default:** `[]`

Directories to symlink from the main repository into each worktree, avoiding large duplicated directories on disk (e.g., `node_modules`).

#### `worktree.sparsePaths`
**Type:** array of strings
**Scope:** all
**Default:** `[]`

Directories to check out in each worktree via git sparse-checkout (cone mode). Only listed paths are written to disk — useful for large monorepos.

```json
{
  "worktree": {
    "symlinkDirectories": ["node_modules", ".cache"],
    "sparsePaths": ["packages/my-app", "shared/utils"]
  }
}
```

---

### AWS and Cloud

#### `awsAuthRefresh`
**Type:** string
**Scope:** all
**Default:** none

Custom script that modifies the `.aws` directory. Runs to refresh AWS credentials before API calls.

```json
{ "awsAuthRefresh": "aws sso login --profile myprofile" }
```

#### `awsCredentialExport`
**Type:** string
**Scope:** all
**Default:** none

Custom script that outputs JSON with AWS credentials. Used for non-standard credential sources.

#### `otelHeadersHelper`
**Type:** string
**Scope:** all
**Default:** none

Script to generate dynamic OpenTelemetry headers. Runs at startup and periodically. See the monitoring docs for the expected output format.

---

### Global Config (`~/.claude.json`)

These settings are stored in `~/.claude.json`, not `settings.json`. Adding them to `settings.json` triggers a schema validation error.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `autoConnectIde` | boolean | `false` | Auto-connect to a running IDE when Claude Code starts from an external terminal |
| `autoInstallIdeExtension` | boolean | `true` | Auto-install the Claude Code IDE extension when running from a VS Code terminal |
| `editorMode` | string | `"normal"` | Key binding mode: `"normal"` or `"vim"`. Written automatically by `/vim` |
| `showTurnDuration` | boolean | `true` | Show turn duration messages after responses (e.g., "Cooked for 1m 6s") |
| `terminalProgressBarEnabled` | boolean | `true` | Show terminal progress bar in ConEmu, Ghostty 1.2.0+, and iTerm2 3.6.6+ |

---

### Additional Schema-Only Keys

Keys confirmed in the JSON schema not covered in the sections above:

| Key | Type | Description |
|-----|------|-------------|
| `claudeMdExcludes` `📋 Schema only` | array | Glob patterns for CLAUDE.md files to exclude from loading |
| `allowManagedMcpServersOnly` | boolean | (Managed) Only managed MCP servers are usable |
| `allowManagedHooksOnly` | boolean | (Managed) Only managed and SDK hooks are loaded |
| `autoMemoryEnabled` | boolean | Enable/disable auto-memory feature |
| `feedbackSurveyRate` | number | Survey appearance probability (0–1) |

---

## Environment Variables

Set in your shell before launching `claude`, or configure under the `env` key in `settings.json` to apply to every session.

### Authentication

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | API key for direct Anthropic API access |
| `ANTHROPIC_AUTH_TOKEN` | OAuth token (alternative to API key) |
| `ANTHROPIC_BASE_URL` | Custom API endpoint (for proxies or private deployments) |
| `ANTHROPIC_CUSTOM_HEADERS` | Custom headers for API requests. Format: `Name: Value`, newline-separated for multiple |
| `CLAUDE_CODE_USER_EMAIL` | Provide user email synchronously for authentication |
| `CLAUDE_CODE_ORGANIZATION_UUID` | Provide organization UUID synchronously for authentication |
| `CLAUDE_CODE_ACCOUNT_UUID` | Override account UUID for authentication |
| `CLAUDE_CONFIG_DIR` | Custom config directory path (overrides default `~/.claude`) |
| `CLAUDE_ENV_FILE` | Custom environment file path |

### Model Selection

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_MODEL` | Model to use. Accepts aliases (`sonnet`, `opus`, `haiku`) or full model IDs. Overrides the `model` setting |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Override the Haiku model alias with a custom model ID |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL_NAME` | Display name for the Haiku model override |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL_DESCRIPTION` | Description for the Haiku model override |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL_SUPPORTED_CAPABILITIES` | Capabilities for the Haiku model override |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Override the Sonnet model alias |
| `ANTHROPIC_DEFAULT_SONNET_MODEL_NAME` | Display name for the Sonnet model override |
| `ANTHROPIC_DEFAULT_SONNET_MODEL_DESCRIPTION` | Description for the Sonnet model override |
| `ANTHROPIC_DEFAULT_SONNET_MODEL_SUPPORTED_CAPABILITIES` | Capabilities for the Sonnet model override |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Override the Opus model alias (e.g., `claude-opus-4-6[1m]`) |
| `ANTHROPIC_DEFAULT_OPUS_MODEL_NAME` | Display name for the Opus model override |
| `ANTHROPIC_DEFAULT_OPUS_MODEL_DESCRIPTION` | Description for the Opus model override |
| `ANTHROPIC_DEFAULT_OPUS_MODEL_SUPPORTED_CAPABILITIES` | Capabilities for the Opus model override |
| `ANTHROPIC_CUSTOM_MODEL_OPTION` | Model ID to add as a custom entry in the `/model` picker |
| `ANTHROPIC_CUSTOM_MODEL_OPTION_NAME` | Display name for the custom model entry |
| `ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION` | Display description for the custom model entry |
| `ANTHROPIC_SMALL_FAST_MODEL` | **DEPRECATED** — use `ANTHROPIC_DEFAULT_HAIKU_MODEL` instead |
| `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION` | AWS region for the deprecated Haiku-class model override |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Override model for subagents (e.g., `haiku`) |
| `CLAUDE_CODE_EFFORT_LEVEL` | Set effort level: `low`, `medium`, `high`, `max` (Opus 4.6 only), or `auto`. Takes precedence over `/effort` and the `effortLevel` setting |

### Cloud Providers (Bedrock, Vertex, Foundry)

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_USE_BEDROCK` | Use AWS Bedrock (`1` to enable) |
| `CLAUDE_CODE_USE_VERTEX` | Use Google Vertex AI (`1` to enable) |
| `CLAUDE_CODE_USE_FOUNDRY` | Use Microsoft Foundry (`1` to enable) |
| `AWS_BEARER_TOKEN_BEDROCK` | Bedrock API key for authentication |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | Skip AWS auth for Bedrock (`1` to skip) |
| `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` | Skip Azure auth for Foundry (`1` to skip) |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH` | Skip Google auth for Vertex (`1` to skip) |
| `ANTHROPIC_FOUNDRY_API_KEY` | API key for Microsoft Foundry authentication |
| `ANTHROPIC_FOUNDRY_BASE_URL` | Base URL for the Foundry resource |
| `ANTHROPIC_FOUNDRY_RESOURCE` | Foundry resource name |
| `VERTEX_REGION_CLAUDE_3_5_HAIKU` | Vertex AI region override for Claude 3.5 Haiku |
| `VERTEX_REGION_CLAUDE_3_7_SONNET` | Vertex AI region override for Claude 3.7 Sonnet |
| `VERTEX_REGION_CLAUDE_4_0_OPUS` | Vertex AI region override for Claude 4.0 Opus |
| `VERTEX_REGION_CLAUDE_4_0_SONNET` | Vertex AI region override for Claude 4.0 Sonnet |
| `VERTEX_REGION_CLAUDE_4_1_OPUS` | Vertex AI region override for Claude 4.1 Opus |

### Timeouts and Limits

| Variable | Description |
|----------|-------------|
| `BASH_DEFAULT_TIMEOUT_MS` | Default bash command timeout in milliseconds |
| `BASH_MAX_TIMEOUT_MS` | Maximum bash command timeout in milliseconds |
| `BASH_MAX_OUTPUT_LENGTH` | Maximum bash output length |
| `MAX_THINKING_TOKENS` | Maximum extended thinking tokens per response |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Max output tokens per response. Default: 32,000 (64,000 for Opus 4.6 as of v2.1.77). Upper bound: 128,000 for Opus 4.6 and Sonnet 4.6 |
| `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` | Override default file read token limit |
| `CLAUDE_CODE_EXIT_AFTER_STOP_DELAY` | Auto-exit SDK mode after this idle duration (ms) |
| `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` | SessionEnd hook timeout in ms (replaces hard 1.5s limit) |
| `CLAUDE_CODE_API_KEY_HELPER_TTL_MS` | Credential refresh interval in ms for `apiKeyHelper` |
| `CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS` | Plugin marketplace git clone timeout in ms (default: 120000) |
| `CLAUDE_CODE_OTEL_HEADERS_HELPER_DEBOUNCE_MS` | Debounce interval in ms for OTel headers helper script |
| `MCP_TIMEOUT` | MCP server startup timeout in ms |
| `MCP_TOOL_TIMEOUT` | MCP tool execution timeout in ms |

### Behavior Control

| Variable | Description |
|----------|-------------|
| `CLAUDECODE` | Set to `1` in shell environments Claude Code spawns (Bash tool, tmux). Not set in hooks or status line commands. Use to detect when a script runs inside Claude Code |
| `CLAUDE_CODE_SHELL` | Override automatic shell detection |
| `CLAUDE_CODE_SHELL_PREFIX` | Command prefix prepended to all bash commands |
| `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR` | Keep working directory between bash calls (`1` to enable) |
| `CLAUDE_CODE_NEW_INIT` | Set to `true` to make `/init` run an interactive setup flow asking which files to generate |
| `CLAUDE_CODE_PLAN_MODE_REQUIRED` | Require plan mode for sessions |
| `CLAUDE_CODE_SKIP_FAST_MODE_NETWORK_ERRORS` | Set to `1` to allow fast mode when org status check fails due to network error (useful with corporate proxies) |
| `CLAUDE_CODE_SIMPLE` | Enable simple/minimal UI mode |
| `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` | Environment variables to scrub from subprocess environments |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` | Additional directories to load CLAUDE.md files from |
| `CLAUDE_CODE_TMPDIR` | Custom temporary directory for Claude Code operations |
| `USE_BUILTIN_RIPGREP` | Use the built-in ripgrep binary instead of system ripgrep |

### Context Window and Compaction

| Variable | Description |
|----------|-------------|
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | Auto-compact threshold percentage (1–100). Default ~95%. Lower values (e.g., `50`) trigger compaction earlier. Values above 95% have no effect |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | Context capacity in tokens used for compaction calculations. Defaults to model's context window (200K standard, 1M for extended). A lower value (e.g., `500000`) on a 1M model treats it as 500K for compaction purposes |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | Disable 1M token context window (`1` to disable) |

### Telemetry and Observability

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_ENABLE_TELEMETRY` | Enable telemetry (`1` to enable) |
| `DISABLE_TELEMETRY` | Disable telemetry (`1` to disable) |
| `DISABLE_ERROR_REPORTING` | Disable error reporting (`1` to disable) |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Equivalent of setting `DISABLE_AUTOUPDATER`, `DISABLE_FEEDBACK_COMMAND`, `DISABLE_ERROR_REPORTING`, and `DISABLE_TELEMETRY` together |

### Feature Toggles

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` | Disable adaptive thinking (`1` to disable) |
| `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` | Disable experimental beta features (`1` to disable) |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | Disable background tasks (`1` to disable) |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | Disable auto-memory (`1` to disable) |
| `CLAUDE_CODE_DISABLE_FAST_MODE` | Disable fast mode entirely (`1` to disable) |
| `CLAUDE_CODE_DISABLE_CRON` | Disable scheduled/cron tasks (`1` to disable) |
| `CLAUDE_CODE_DISABLE_GIT_INSTRUCTIONS` | Disable git-related system prompt instructions. Takes precedence over `includeGitInstructions` setting |
| `CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK` | Disable the non-streaming fallback for failed streaming requests |
| `CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY` | Disable feedback survey prompts (`1` to disable) |
| `CLAUDE_CODE_DISABLE_TERMINAL_TITLE` | Disable terminal title updates (`1` to disable) |
| `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION` | Enable prompt suggestions |
| `CLAUDE_CODE_ENABLE_TASKS` | Set to `true` to enable task tracking in non-interactive mode (`-p` flag). Tasks are on by default in interactive mode |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | Enable experimental agent teams feature |
| `ENABLE_CLAUDEAI_MCP_SERVERS` | Enable Claude.ai MCP servers |
| `CLAUDE_CODE_USE_POWERSHELL_TOOL` | Enable the PowerShell tool on Windows (requires `defaultShell: "powershell"` in settings) |
| `FORCE_AUTOUPDATE_PLUGINS` | Force plugin auto-updates (`1` to enable) |
| `IS_DEMO` | Enable demo mode |

### Prompt Caching

| Variable | Description |
|----------|-------------|
| `DISABLE_PROMPT_CACHING` | Disable all prompt caching (`1` to disable) |
| `DISABLE_PROMPT_CACHING_HAIKU` | Disable prompt caching for Haiku model requests |
| `DISABLE_PROMPT_CACHING_SONNET` | Disable prompt caching for Sonnet model requests |
| `DISABLE_PROMPT_CACHING_OPUS` | Disable prompt caching for Opus model requests |

### MCP Configuration

| Variable | Description |
|----------|-------------|
| `MAX_MCP_OUTPUT_TOKENS` | Max MCP output tokens per call (default: 25000). A warning is shown when output exceeds 10,000 tokens |
| `ENABLE_TOOL_SEARCH` | MCP tool search threshold (e.g., `auto:5` — activates tool search when there are more than 5 MCP tools) |
| `MCP_CLIENT_SECRET` | MCP OAuth client secret |
| `MCP_OAUTH_CALLBACK_PORT` | MCP OAuth callback port |

### Proxy and Network

| Variable | Description |
|----------|-------------|
| `HTTP_PROXY` | HTTP proxy URL for network requests |
| `HTTPS_PROXY` | HTTPS proxy URL for network requests |
| `NO_PROXY` | Comma-separated list of hosts that bypass the proxy |
| `CLAUDE_CODE_PROXY_RESOLVES_HOSTS` | Allow the proxy to perform DNS resolution |
| `CLAUDE_CODE_CLIENT_CERT` | Client certificate path for mTLS |
| `CLAUDE_CODE_CLIENT_KEY` | Client private key path for mTLS |
| `CLAUDE_CODE_CLIENT_KEY_PASSPHRASE` | Passphrase for an encrypted mTLS key |

### UI and Display

| Variable | Description |
|----------|-------------|
| `DISABLE_COST_WARNINGS` | Disable cost warning messages |
| `DISABLE_INSTALLATION_CHECKS` | Disable installation warnings |
| `DISABLE_AUTOUPDATER` | Disable the auto-updater |
| `DISABLE_FEEDBACK_COMMAND` | Disable the `/feedback` command. The older name `DISABLE_BUG_COMMAND` is also accepted |
| `CLAUDE_CODE_IDE_SKIP_AUTO_INSTALL` | Skip automatic IDE extension installation (`1` to skip). Also configurable via `autoInstallIdeExtension` in `~/.claude.json` |
| `SLASH_COMMAND_TOOL_CHAR_BUDGET` | Character budget for slash command tool output |

### Agent Teams and Tasks

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_TEAM_NAME` | Team name for agent teams |
| `CLAUDE_CODE_TASK_LIST_ID` | Task list ID for task integration |

### Unverified Variables

These appear in community sources or older documentation but are not confirmed in current official docs.

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_MAX_TURNS` `⚠️ Unverified` | Maximum agentic turns before stopping |
| `CLAUDE_CODE_SKIP_SETTINGS_SETUP` `⚠️ Unverified` | Skip first-run settings setup flow |
| `CLAUDE_CODE_DISABLE_TOOLS` `⚠️ Unverified` | Comma-separated list of tools to disable |
| `CLAUDE_CODE_DISABLE_MCP` `⚠️ Unverified` | Disable all MCP servers (`1` to disable) |
| `CLAUDE_CODE_HIDE_ACCOUNT_INFO` `⚠️ Unverified` | Hide email/org info from UI |
| `CLAUDE_CODE_PROMPT_CACHING_ENABLED` `⚠️ Unverified` | Override prompt caching behavior |
| `DISABLE_NON_ESSENTIAL_MODEL_CALLS` `⚠️ Unverified` | Disable flavor text and non-essential model calls |

---

## Complete Example

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "model": "sonnet",
  "language": "english",
  "cleanupPeriodDays": 30,
  "autoUpdatesChannel": "stable",
  "alwaysThinkingEnabled": false,
  "includeGitInstructions": true,
  "effortLevel": "medium",
  "plansDirectory": "./plans",

  "worktree": {
    "symlinkDirectories": ["node_modules"],
    "sparsePaths": ["packages/my-app", "shared/utils"]
  },

  "permissions": {
    "allow": [
      "Edit(*)",
      "Write(*)",
      "Bash(npm run *)",
      "Bash(git *)",
      "WebFetch(domain:*)",
      "mcp__*"
    ],
    "ask": ["Bash(git push *)"],
    "deny": [
      "Read(.env)",
      "Read(./secrets/**)"
    ],
    "additionalDirectories": ["../shared/"],
    "defaultMode": "acceptEdits"
  },

  "enableAllProjectMcpServers": true,

  "sandbox": {
    "enabled": true,
    "excludedCommands": ["git", "docker"],
    "filesystem": {
      "allowWrite": ["/tmp/build"],
      "denyRead": ["~/.aws/credentials"]
    },
    "network": {
      "allowedDomains": ["github.com", "*.npmjs.org"],
      "allowUnixSockets": ["/var/run/docker.sock"]
    }
  },

  "attribution": {
    "commit": "Generated with Claude Code",
    "pr": ""
  },

  "statusLine": {
    "type": "command",
    "command": "git branch --show-current"
  },

  "spinnerTipsEnabled": true,
  "prefersReducedMotion": false,

  "env": {
    "NODE_ENV": "development",
    "CLAUDE_CODE_EFFORT_LEVEL": "medium"
  }
}
```

---

## Quick Reference

| Task | Setting / Variable |
|------|--------------------|
| Set default model | `model` in settings or `ANTHROPIC_MODEL` env var |
| Lock model choices | `availableModels` array |
| Silence telemetry | `DISABLE_TELEMETRY=1` or `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` |
| Auto-approve bash | `permissions.defaultMode: "acceptEdits"` |
| Block sensitive files | `permissions.deny: ["Read(.env)"]` |
| Reduce context compaction | `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70` |
| Set response language | `language: "japanese"` |
| Custom spinner text | `spinnerVerbs` + `spinnerTipsOverride` |
| Remove attribution | `attribution.commit: ""`, `attribution.pr: ""` |
| Pin to stable releases | `autoUpdatesChannel: "stable"` |
| Dynamic auth token | `apiKeyHelper: "/path/to/script.sh"` |
| Large monorepo | `worktree.symlinkDirectories` + `worktree.sparsePaths` |
| Enable sandboxing | `sandbox.enabled: true` |
| Trust all project MCP servers | `enableAllProjectMcpServers: true` |
