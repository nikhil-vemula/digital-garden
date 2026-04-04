---
title: Claude Code API Call Analysis
draft: false
tags:
---

Analysis of HTTP calls made by Claude Code (v2.1.92) during a single session, captured via mitmproxy.

**Test performed:** Opened Claude Code CLI and asked it to edit the `<title>` of an HTML page from "test 2" to "test 3". This simple one-file edit was used to observe the full lifecycle of API calls — from startup through task completion to telemetry.

## TL;DR

- **"Tengu"** is the internal codename for Claude Code. All 207 feature flags are prefixed `tengu_`.
- **"Penguin Mode"** is Fast Mode. Same Opus 4.6 model, 2.5x faster, $30/$150 per MTok. Requires extra usage billing.
- **A simple "edit title" task takes 13 HTTP requests** — 6 startup, 4 tool-use loop turns, 1 title gen, 2 telemetry.
- **The entire system prompt is sent in plaintext** — all behavior rules, safety instructions, memory system, tool definitions. Cached globally for 1 hour.
- **Thinking blocks carry cryptographic signatures** — API-key-bound and model-bound, replayed in conversation history to prevent tampering.
- **Adaptive thinking skips thinking on mechanical tool calls** — only activated on the first turn (deciding what to do), saving tokens.
- **Prompt caching is aggressive** — only 1 uncached input token per continuation turn, ~16k served from cache.
- **Your exact session cost is tracked client-side** in `~/.claude.json` under `projects.<path>.lastCost`. Previous session: $0.0426. Title generation: $0.00041.
- **Telemetry goes to two sinks** — Anthropic's first-party event batch (base64-encoded metadata) AND Datadog (plaintext structured logs with cost, latency, process metrics).
- **Your email, device ID, account UUID, and plan tier (`max`) are sent on every telemetry event.**
- **SWE-bench evaluation fields** are baked into production Datadog logs (empty but present).
- **Food codenames** (`turmeric`, `bananagrams`, `sourdough`, `paprika`, `saffron`) are claude.ai web features. **Bird/mineral codenames** (`cobalt_heron`, `lapis_finch`) are Claude Code flags.
- **"Grove"** = privacy consent framework. `grove_enabled` means different things on different endpoints (consent completed vs. data collection preference).
- **"Dittos"** = mobile app codename. **"Cowork"** = enterprise team mode. **"Compass"** = advisor tool (consult stronger model).

## Minimizing Waste / Cost Optimization

**Batch tasks into a single session.** The 6 startup requests + quota probe + telemetry are fixed costs per session. One session with 10 tasks is far cheaper than 10 sessions with 1 task each.

**Stay within the 1-hour cache window.** The system prompt is cached globally for 1 hour (`scope: global`). Back-to-back tasks within that window pay near-zero for re-sending the ~4k word system prompt. Spreading sessions across hours wastes cache hits.

**Give precise, targeted prompts.** The title-edit took 4 tool-use turns (Glob → Read → Edit → Done), each a separate API call with ~16k tokens re-sent from cache. A prompt with the exact file path (e.g. `"Edit <title> in src/index.html from 'X' to 'Y'"`) skips Glob entirely — 3 turns instead of 4.

**Use `/compact` before context grows large.** Context compression is configured but disabled by default (`tengu_sm_compact: false`). Past ~150k tokens costs spike. Proactively compacting keeps cache-read tokens high and uncached tokens low.

**Use `claude -p` for scripted/automated tasks.** Non-interactive mode skips the 1-token Haiku quota probe and interactive startup checks.

**Avoid Fast Mode unless speed is critical.** "Penguin Mode" uses the same Opus 4.6 model but at **$30/$150 per MTok** (vs. standard rates), billed as extra usage outside your plan's included allocation from the first token.

**Every new session generates a title via Haiku ($0.00041).** Trivial individually, but multiplies across many short sessions.

| Action | Impact |
|--------|--------|
| Batch multiple tasks per session | High — amortizes fixed startup cost |
| Stay within 1h cache window | High — maximizes cache hits |
| Give precise file paths in prompts | Medium — reduces tool-use turns |
| Use `claude -p` for automation | Low-medium — skips quota probe |
| Avoid Fast Mode unless needed | Situational — 2-3x higher rate |

## High-Level Call Flow

**Startup (6 calls):**
1. `POST /api/eval/...` — Feature flags (207 flags)
2. `GET /api/oauth/account/settings` — Account preferences
3. `GET /api/claude_code_grove` — Privacy consent check
4. `GET /api/claude_cli/bootstrap` — Server-pushed config
5. `GET /api/claude_code_penguin_mode` — Fast mode eligibility
6. `POST /v1/messages` (Haiku, 1 token) — Quota/rate limit probe

**Conversation (5 calls):**
7. `POST /v1/messages` (Haiku) — Session title generation
8. `POST /v1/messages` (Opus) → tool_use: Glob — Find HTML files
9. `POST /v1/messages` (Opus) → tool_use: Read — Read index.html
10. `POST /v1/messages` (Opus) → tool_use: Edit — Edit title
11. `POST /v1/messages` (Opus) → end_turn: "Done." — Final response

**Telemetry (2 calls):**
12. `POST /api/event_logging/v2/batch` — First-party events (30+ events)
13. `POST datadoghq.com/api/v2/logs` — Datadog per-call structured logs

**Total: ~13 HTTP requests for a single "edit title" task.**

## REST API Calls

### `POST https://api.anthropic.com/api/eval/...`
- **Purpose:** Feature flags and experiment evaluation. On startup, Claude Code fetches the full set of feature flags for your account from Anthropic's feature flagging system (GrowthBook-based). Returns which features are enabled/disabled and which A/B experiments you're enrolled in.
- **Internal codename:** "Tengu" is the internal name for Claude Code. All flags are prefixed `tengu_`.
- **Flag naming:** Uses obscured codenames with bird/mineral themes (e.g. `cobalt_heron`, `lapis_finch`, `amber_wren`).
- View appendix for complete table.


### `GET https://api.anthropic.com/api/oauth/account/settings`
- **Purpose:** Fetches user account settings and preferences. Returns UI feature toggles, onboarding state, dismissed banners, and product-specific configurations for claude.ai, Claude Code, and Claude Code Remote (CCR).
- **Key insights:**
  - **`grove_enabled: false`** — The actual "Help improve Claude" data collection preference (`false` = opted out). Grove is Claude Code's Terms & Privacy consent framework with 24h cache. Gets its own dedicated API call (#3).
  - **`paprika_mode: "off"`** — Multi-state mode toggle (string, not boolean), suggesting multiple states. Claude.ai feature.
  - **`wiggle_egress`** — Controlled external network access from Claude with host allowlists and templates. Server-managed egress filtering.
  - **`orbit`** — Timezone-aware scheduling feature (stores `orbit_timezone`)
  - **`compass` / `enabled_compass`** — Maps to the **Advisor tool**. Lets Claude consult a stronger reviewer model before substantive work.
  - **`cowork_*`** — **Enterprise/team collaboration mode** ("Coworker"). Has its own plugin directory, CLI flag `--cowork`, SMS notifications, trial periods.
  - **`ccr_sharing_*` / `ccr_auto_*`** — Claude Code Remote features: session sharing with team, auto-archive on PR close, autofix on PR create, plugin mounting, persistent cross-session memory
  - **`dittos_mobile_onboarding`** — "Dittos" is the mobile companion app codename
  - **Food codenames** (`turmeric`, `bananagrams`, `sourdough`, `foccacia`, `yukon_gold`, `paprika`, `saffron`, `melange`) — claude.ai web features, not used in Claude Code. Anthropic uses food/spice themes for web product features.
  - **`onboarding_use_case: "personal"`** — User self-identified as personal use during onboarding
  - **Banner dismissals** reveal daily `mcp_directory_chin` banners (MCP marketplace push, one per day) and one-time NUX banners (`install-hub-nux`, `dispatch-sidebar-nux`)

### `GET https://api.anthropic.com/api/claude_code_grove`
- **Purpose:** Checks Grove (privacy/data collection consent) status. A startup gate that determines whether Claude Code can proceed or must show a consent dialog.
- **Key insights:**
  - **`grove_enabled` meaning differs by endpoint:**
    - **Account settings** (`/api/oauth/account/settings`): `grove_enabled: false` = user opted out of data collection
    - **Grove config** (`/api/claude_code_grove`): `grove_enabled: true` = user has completed the consent flow (not the choice itself)
    - If account settings has `grove_enabled: null` (never chosen) — **blocks usage**. Interactive mode shows consent dialog. Non-interactive (`claude -p`) prints warning during grace period or **force-exits after grace period ends**.
  - **`domain_excluded: false`** — Enterprise orgs can disable the toggle at the domain level (grays it out)
  - **`notice_is_grace_period: false`** — Grace period has ended; if `grove_enabled` were `null`, Claude Code would refuse to run in headless mode
  - **`notice_reminder_frequency: 0`** — No recurring reminders (during grace period, this controls how often to re-nag)
  - Cached for **24 hours** — after TTL expires, re-fetched from API
  - Users can toggle later via `/privacy-settings` command

### `GET https://api.anthropic.com/api/claude_cli/bootstrap`
- **Purpose:** Startup bootstrap data fetch. Returns server-pushed client configuration and additional model options available to the user.
- **Key insights:**
  - **`client_data: {}`** — Empty object; this is a generic key-value bag the server can use to push arbitrary config to the client without a code change
  - **`additional_model_options: null`** — No extra models beyond the defaults. When populated, each entry has `model`, `name`, `description` — these appear in the model selector (e.g. early access or org-specific models)
  - Cached to disk and only re-written if data changes (avoids unnecessary config writes on every startup)
  - Skipped entirely for third-party API providers or when nonessential traffic is disabled
  - Has a **5-second timeout** — designed to not block startup
  - Auth: prefers OAuth, falls back to API key for console users

### `GET https://api.anthropic.com/api/claude_code_penguin_mode`
- **Purpose:** Checks if **Fast Mode** (codename "Penguin Mode") is allowed for the user's org/account. Fast Mode uses Opus 4.6 with faster output.
- **Key insights:**
  - **`enabled: false`** — Fast mode is not available for this account
  - **`disabled_reason: "extra_usage_disabled"`** — Fast mode requires extra usage billing (overage). Message shown: *"Fast mode requires extra usage billing · /extra-usage to enable"*
  - **5 possible disabled reasons:** `free` (no paid plan), `preference` (org admin disabled it), `extra_usage_disabled` (no overage billing), `network_error`, `unknown`
  - **Penguin = Fast Mode** — the entire fast mode system is internally called "penguin mode" (`penguinModeOrgEnabled` in config cache)
  - Fast mode is **not a different model** — same Opus 4.6, different API config prioritizing speed over cost. **2.5x faster** at **$30/$150 per MTok** (input/output)
  - Billed directly to extra usage from the first token — does not count against plan's included usage
  - **Cooldown system:** On rate limit (429) or overload, fast mode enters cooldown with a `resetAt` timestamp, then auto-re-enables
  - **Overage rejection:** If 429 indicates extra usage isn't available, fast mode is permanently disabled for the session (unless reason is `out_of_credits`)
  - **Ant employees** default to enabled on network failure — internal users aren't blocked
  - Result is cached to disk (`penguinModeOrgEnabled`) and re-prefetched with a **30-second minimum interval**

### `POST https://api.anthropic.com/v1/messages?beta=true` (Haiku — quota check)
- **Purpose:** A **throwaway 1-token request** to Haiku to check the user's rate limit / quota status. Not a real conversation — it's a probe.
- **Key insights:**
  - Model: `claude-haiku-4-5-20251001` (cheapest/fastest model, via `getSmallFastModel()`)
  - Input: 8 tokens, output: 1 token (`"#"`), `stop_reason: "max_tokens"` — `max_tokens` is set to 1 intentionally
  - The actual response content is **discarded** — Claude Code only cares about the HTTP response **headers** which contain rate limit info (remaining tokens, reset time)
  - Skipped in non-interactive mode (`claude -p`) since the real query that follows will provide the same header info
  - Used to populate the rate limit status bar (tokens remaining, time until reset) before the user starts chatting
  - `service_tier: "standard"` and `inference_geo: "not_available"` — metadata about how the request was served

### `POST https://api.anthropic.com/v1/messages?beta=true` (Haiku — session title)
- **Purpose:** Auto-generates a short session title from the user's first message using Haiku. This is the title shown in session history.
- **Key insights:**
  - Uses `json_schema` structured output to force a `{"title": "..."}` response
  - Prompt: *"Generate a concise, sentence-case title (3-7 words)"* with good/bad examples
  - Input is just the user's first message: `"edit title of html page to test 3"`
  - `temperature: 1` — allows creative variation in titles
  - This is a "side query" — runs in background, doesn't block the main conversation

### `POST https://api.anthropic.com/v1/messages?beta=true` (Opus — main conversation)
- **Purpose:** The actual conversation turn. This is Claude Code's main loop — sends the user's message to Opus 4.6 and streams back the response.
- **Response insights:**
  - **Streaming SSE format:** `message_start` → `content_block_start/delta/stop` → `message_delta` → `message_stop`
  - **Thinking block with signature:** First content block is `type: "thinking"` with a cryptographic `signature` (base64, ~200 bytes). Signatures are API-key-bound and model-bound — prevent tampering, must be replayed in subsequent turns.
  - **Prompt caching in action:** `cache_creation_input_tokens: 4493` (1h cache), `cache_read_input_tokens: 11173`. Two cache tiers: `ephemeral_5m` and `ephemeral_1h`.
  - **Tool use loop:** Claude calls `Glob {"pattern": "**/*.html"}`, stops (`stop_reason: "tool_use"`), waits for result, then continues in next API call.
  - **`caller: {"type": "direct"}`** — tool called by the model directly (vs. subagent)
- **Request structure (full payload captured):**
  - **System prompt:** 4 text blocks — billing header, identity, full behavior instructions (~4k words), session-specific guidance + memory system + environment info
  - **Billing header baked into system prompt:** `cc_version=2.1.92.7a0; cc_entrypoint=cli; cch=db3db;` — version, entrypoint, and hash sent as first system block
  - **System prompt has 1h global cache:** `cache_control: {"type": "ephemeral", "ttl": "1h", "scope": "global"}` — the massive instruction prompt is cached globally across sessions
  - **Tool results also cached:** `cache_control: {"type": "ephemeral", "ttl": "1h"}` on tool_result content
  - **8 tools defined:** Agent, Bash, Edit, Glob, Grep, Read, Skill, ToolSearch, Write — each with full JSON Schema. Deferred tools (TaskCreate, WebFetch, etc.) loaded on-demand via ToolSearch.
  - **`thinking: {"type": "adaptive"}`** — thinking is enabled but adaptive (model decides when to think)
  - **`output_config: {"effort": "medium"}`** — effort level setting passed directly to the API
  - **`max_tokens: 8000`** — output capped at 8k tokens per turn
  - **`metadata.user_id`** — JSON string containing `device_id` (SHA-256), `account_uuid`, and `session_id`
  - **Conversation structure:** User message has 4 content blocks — system reminders (deferred tools, skills, date) injected before the actual user text
  - **Tool result in history:** Previous Glob result (`"index.html"`) sent back with the thinking signature block, completing the tool use loop
- **Tool use loop observed (4 API calls for 1 task):**
  1. Opus → `Glob {"pattern": "**/*.html"}` (with thinking, 80 output tokens)
  2. Opus → `Read {"file_path": ".../index.html"}` (no thinking — adaptive skipped it, 68 tokens)
  3. Opus → `Edit {"old_string": "<title>test 2</title>", "new_string": "<title>test 3</title>"}` (no thinking, 118 tokens)
  4. Opus → `"Done. Title changed from \"test 2\" to \"test 3\"."` (end_turn, 19 tokens)
  - Total: 285 output tokens, ~1 uncached input token per turn, 15-16k tokens served from cache each turn
  - **Adaptive thinking** only activated on turn 1 (deciding what to do), skipped for mechanical tool calls

### `POST https://api.anthropic.com/api/event_logging/v2/batch`
- **Purpose:** First-party telemetry batch. Sends a batch of 30+ `ClaudeCodeInternalEvent` events covering the entire startup lifecycle and previous session summary.
- **Key insights:**
  - **All `additional_metadata` and `process` fields are base64-encoded JSON** — not encrypted, just encoded
  - **`process` field** on every event: `{uptime, rss, heapTotal, heapUsed, external, arrayBuffers, constrainedMemory, cpuUsage, cpuPercent}` — full Node.js process stats snapshot
  - **Events include email in plaintext** on every single event
  - **`betas` string reveals all active API betas:** `claude-code-20250219, oauth-2025-04-20, context-1m-2025-08-07, interleaved-thinking-2025-05-14, redact-thinking-2026-02-12, context-management-2025-06-27, prompt-caching-scope-2026-01-05`
  - **`tengu_exit` (previous session summary):**
    - `last_session_cost: $0.0426` — exact dollar cost of previous session
    - `last_session_duration: 942295ms` (~15.7 minutes)
    - `last_session_total_input_tokens: 343`, `output_tokens: 37`, `cache_read: 26831`
    - **UI frame performance:** p50=0.57ms, p95=1.58ms, p99=6.09ms (246 frames measured)
    - Hook duration stats (p50=278ms)
  - **`tengu_init` reveals full config:**
    - `permissionMode: "default"`, `thinkingType: "adaptive"`, `numAllowedTools: 24`
    - `mcpClientCount: 1`, `autoUpdatesChannel: "latest"`
    - `apiKeySource: "none"` (using OAuth, not API key)
  - **`tengu_startup_telemetry`:** `sandbox_enabled: false`, `gh_auth_status: "not_installed"`, `is_git: false`
  - **`tengu_startup_manual_model_config`:** `subscriptionType: "max"` — reveals your plan tier
  - **`tengu_concurrent_sessions`:** `num_sessions: 2` — tracking parallel sessions
  - **Startup event sequence:** `shell_set_cwd` → `started` → `dir_search` (commands, agents, output-styles, skills, workflows) → `version_lock_failed` → `exit` (prev session) → `timer` (422ms startup) → `claudemd_initial_load` → `prompt_suggestion_init` → `mcp_tools_commands_loaded` → `init` → `skill_loaded` (×6) → `startup_telemetry` → `ripgrep_availability` → `file_suggestions_ripgrep` → `context_size` → `claudeai_mcp_eligibility` → `mcp_servers` → `claudeai_limits_status_changed` → `version_check_success` → `native_update_complete` → `native_auto_updater_up_to_date` → `native_version_cleanup`

### `POST https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- **Purpose:** Sends structured logs to Datadog (US5 region) for each API call. This is the second telemetry sink alongside Anthropic's first-party event logging.
- **Key insights:**
  - **Every API call gets a Datadog log** — this one is for the session title Haiku call (`query_source: "generate_session_title"`)
  - **Exact cost per call:** `cost_u_s_d: 0.00041` — less than a tenth of a cent for the title generation
  - **Detailed latency:** `duration_ms: 667`, `ttft_ms: 665` (time to first token), `attempt: 1`
  - **`user_bucket: 13`** — users are bucketed (likely for A/B testing or load analysis)
  - **`subscription_type: "max"`** — plan tier sent to Datadog
  - **SWE-bench fields present but empty** — `swe_bench_run_id`, `swe_bench_instance_id`, `swe_bench_task_id` — infrastructure for benchmark evaluation is baked into production telemetry
  - **`is_claubbit: false`** — "Claubbit" is another internal mode/tool codename
  - **`is_conductor: false`** — "Conductor" is yet another mode
  - **`global_cache_strategy: "system_prompt"`** — confirms the caching approach
  - **Process metrics inline** (not base64 like first-party events): RSS=345MB, heap=91MB, CPU=17%
  - **`ddtags` string** for Datadog filtering: event name, arch, client type, model, platform, subscription, user bucket, user type, version
  - **`build_age_mins: 1232`** (~20.5 hours since this build was created)

## Appendix: Complete Feature Flags Table

Source column: `default` = default value, `force` = server-forced override, `experiment` = A/B test assignment.

#### Infrastructure & Bridge

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu-off-switch` | `{activated: false}` | default | Global kill switch for Claude Code |
| `tengu_version_config` | `{minVersion: "1.0.24"}` | default | Minimum CLI version enforcement |
| `tengu_max_version_config` | `{}` | default | Maximum CLI version enforcement |
| `tengu_bridge_min_version` | `{minVersion: "2.1.70"}` | default | Minimum bridge version enforcement |
| `tengu_bridge_repl_v2` | `true` | force | REPL bridge v2 enablement |
| `tengu_bridge_repl_v2_config` | retry/timeout/heartbeat config | default | Bridge v2 configuration (retry attempts, timeouts, UUID dedup, heartbeat intervals) |
| `tengu_bridge_poll_interval_config` | poll interval config | default | Bridge polling intervals (seek work, at-capacity heartbeat) |
| `tengu_bridge_poll_interval_ms` | `0` | default | Legacy bridge polling interval |
| `tengu_pid_based_version_locking` | `true` | default | PID-based version locking to prevent conflicts |
| `auto_migrate_to_native` | `false` | default | Auto-migration to native installation |
| `tengu_native_installation` | `false` | default | Native app installation flow |

#### Authentication & Sessions

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_vscode_cc_auth` | `true` | default | VSCode Claude Code authentication |
| `tengu_otk_slot_v1` | `true` | force | One-time key slot authentication |
| `tengu_session_memory` | `false` | default | Async session memory extraction |
| `tengu_ccr_bridge` | `true` | force | Claude Code Remote (CCR) bridge enablement |
| `tengu_ccr_bridge_multi_session` | `true` | force | CCR multi-session support |
| `tengu_ccr_bundle_seed_enabled` | `true` | default | CCR bundle seeding for git telemetry |
| `tengu_ccr_bundle_max_bytes` | `104857600` (100MB) | default | CCR bundle size limit |
| `ccr_auto_permission_mode` | `false` | default | CCR auto permission mode |

#### Token Limits & Capacity

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_pewter_kestrel` | `{global:50k, Bash:30k, Grep:20k, Snip:1k, ...}` | default | Per-tool output token limits |
| `tengu_hawthorn_window` | `200000` | default | Context window size limit (200k tokens) |
| `tengu_c4w_usage_limit_notifications_enabled` | `true` | force | Rate limit / usage notification UI |

#### Context Management & Compression

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_sm_config` | `{minTokens:150k, gapBetween:40k, toolCalls:10}` | default | Context compression trigger thresholds |
| `tengu_sm_compact` | `false` | default | Context compression enablement |
| `tengu_sm_compact_config` | `{minTokens:2k, maxTokens:20k, ...}` | default | Compaction output size configuration |
| `tengu_cold_compact` | `false` | default | Cold/idle compaction strategy |
| `tengu_compact_cache_prefix` | `true` | force | Cache key prefix for compacted context |
| `tengu_compact_line_prefix_killswitch` | `false` | default | Kill line-prefix format in compaction |
| `tengu_cache_plum_violet` | `true` | force | Context caching strategy |
| `tengu_prompt_cache_1h_config` | `{allowlist: ["repl_main_thread*", "sdk", "auto_mode"]}` | default | 1-hour prompt caching allowlist |
| `tengu_system_prompt_global_cache` | `true` | force | Global system prompt caching |
| `tengu_cobalt_raccoon` | `false` | default | Context compaction optimization |

#### Thinking & Extended Analysis

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_crystal_beam` | `{budgetTokens: 0}` | default | Extended thinking budget (0 = default/auto) |
| `tengu_turtle_carbon` | `true` | default | Extended thinking enablement |
| `tengu_thinkback` | `false` | default | Thinking output handling/display |
| `tengu_defer_caveat_m9k` | `false` | default | Thinking output deferral |
| `tengu_defer_all_bn4` | `false` | default | Defer all responses mode |

#### Response Style & Formatting

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_swann_brevity` | `"focused"` | force | Response verbosity style (focused = concise) |
| `tengu_streaming_text` | `true` | force | Enable text streaming output |
| `tengu_streaming_tool_execution2` | `true` | force | Enable streaming during tool execution |
| `tengu_summarize_tool_results` | `true` | force | Summarize long tool results |
| `tengu_marble_sandcastle` | `false` | default | Fast mode formatting behavior |
| `tengu_sotto_voce` | `true` | default | Quiet/subdued output mode |
| `tengu_code_diff_cli` | `true` | force | CLI-based diff formatting |

#### Effort Level & UI

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_grey_step` | `true` | default | Effort level UI enablement |
| `tengu_grey_step2` | dialog title/description config | default | Effort level recommendation dialog ("We recommend medium effort for Opus") |
| `tengu_grey_wool` | `true` | default | Effort level styling/presentation |
| `tengu_jade_anvil_4` | `false` | default | Rate limit purchase UI flow |

#### File Operations & Tools

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_file_write_optimization` | `true` | force | Optimized file writing strategy |
| `tengu_quartz_lantern` | `true` | default | File read/write optimization |
| `tengu_tool_result_persistence` | `true` | force | Persist tool results across context |
| `tengu_satin_quoll` | `{}` | default | Tool result storage persistence threshold override |
| `tengu_editafterwrite_qpl` | `false` | default | Edit-after-write file handling behavior |
| `tengu_edit_anchorwaste_kp3` | `false` | default | Anchor waste cleanup in file edits |
| `tengu_read_dedup_killswitch` | `false` | default | Killswitch for file read deduplication |
| `tengu_noreread_q7m_velvet` | `false` | default | Prevent re-reading unchanged files |
| `tengu_relpath_gh7k` | `false` | default | Relative path handling in git operations |
| `tengu_collage_kaleidoscope` | `true` | default | Image paste/attachment enablement |
| `tengu_marble_fox` | `false` | default | Image paste/attachment handling |
| `tengu_moth_copse` | `false` | default | Attachment handling |
| `tengu_malort_pedway` | `{enabled:true, pixelValidation:false, clipboardPaste...}` | force | Clipboard paste configuration |

#### Tool Search & Deferred Tools

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_glacier_2xr` | `true` | default | Tool search enablement |
| `tengu_tool_search_unsupported_models` | `["haiku"]` | default | Models where tool search isn't supported |
| `tengu_mcp_tool_search` | `true` | force | MCP tool search in prompts |
| `tengu_tool_pear` | `false` | default | Strict tool use schema support |
| `tengu_brief_tool_enabled` | `false` | default | Brief tool mode |

#### Agents & Worktrees

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_worktree_mode` | `true` | force | Git worktree isolation for subagents |
| `tengu_amber_stoat` | `true` | default | Built-in agents enablement |
| `tengu_amber_flint` | `true` | default | Agent teams/swarms feature |
| `tengu_sub_nomdrep_q7k` | `false` | default | Subagent experiment |
| `tengu_lean_sub_pf7q` | `false` | default | Lean subagent mode |
| `tengu_mcp_subagent_prompt` | `false` | default | Subagent prompt customization |
| `tengu_workout2` | `true` | default | Worktree v2 implementation |

#### MCP & Plugins

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_harbor` | `true` | force | MCP channel allowlist/authorization |
| `tengu_harbor_permissions` | `true` | default | MCP channel permissions enforcement |
| `tengu_harbor_ledger` | list of plugins (discord, telegram, etc.) | default | MCP audit ledger — tracks specific plugin allowlist |
| `tengu_lapis_finch` | `true` | force | Plugin hint recommendation system |
| `tengu_claudeai_mcp_connectors` | `true` | force | Claude.ai MCP connectors integration |
| `tengu_copper_bridge` | `true` | default | Chrome MCP server bridge |
| `tengu_mcp_elicitation` | `true` | force | MCP prompt elicitation |
| `tengu_basalt_3kr` | `true` | default | MCP instruction deltas |
| `tengu_plugin_official_mkt_git_fallback` | `true` | default | Plugin marketplace git fallback |
| `tengu_amber_lattice` | `{plugins: ["security-guidance", "code-review", "commit-..."]}` | force | Official plugin list configuration |

#### Permissions & Safety

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_permission_explainer` | `true` | default | Permission request explanation UI |
| `tengu_disable_bypass_permissions_mode` | `false` | default | Killswitch for bypass permissions mode |
| `tengu_destructive_command_warning` | `false` | default | Warning before destructive commands (rm, kill, etc.) |
| `tengu_react_vulnerability_warning` | `false` | default | React vulnerability check/warning |
| `tengu_accept_with_feedback` | `true` | force | Accept tool calls with inline feedback |

#### UltraPlan

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_ultraplan_config` | `{enabled: true}` | default | UltraPlan feature enablement |
| `tengu_ultraplan_prompt_identifier` | `"visual_plan"` | force | UltraPlan prompt variant selection |
| `tengu_ultraplan_timeout_seconds` | `1800` (30 min) | default | UltraPlan timeout limit |

#### Code Review

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_review_bughunter_config` | `{fleet_size:5, max_duration_minutes:10, ...}` | default | Bug hunter review agent config (fleet size, timeout) |
| `tengu_pr_status_cli` | `true` | force | PR status in CLI |
| `tengu_gha_plugin_code_review` | `false` | default | GitHub Actions plugin code review |

#### Idle / Return Detection (Willow)

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_willow_mode` | `"hint_v2"` | force | Idle return behavior (dialog, hint, hint_v2, off) |
| `tengu_willow_sentinel_ttl_hours` | `1` | default | Idle sentinel TTL (hours) |
| `tengu_willow_census_ttl_hours` | `24` | default | Idle session census TTL (hours) |
| `tengu_willow_refresh_ttl_hours` | `0` | default | Idle refresh TTL |
| `tengu_willow_prism` | `true` | force | Idle return prism behavior |

#### Memory & Extraction

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_bramble_lintel` | `7` | default | Memory extraction batch multiplier |
| `tengu_cedar_halo` | `false` | default | Memory feature flag |
| `tengu_herring_clock` | `false` | default | Team memory sync |
| `tengu_passport_quail` | `false` | default | Team memory feature (KAIROS-only) |
| `tengu_slate_thimble` | `false` | default | Team memory extraction in non-interactive mode |
| `tengu_coral_fern` | `false` | default | Team memory feature |

#### Telemetry & Analytics

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_1p_event_batch_config` | batch scheduling/size config | force | First-party event batch configuration |
| `tengu_event_sampling_config` | `{}` | default | Per-event sampling rate configuration |
| `tengu_frond_boric` | `{}` | default | Analytics sink killswitch (Datadog/firstParty) |
| `tengu_log_datadog_events` | `true` | force | Datadog event logging gate |
| `tengu_trace_lantern` | `false` | default | Beta session tracing |
| `tengu_attribution_header` | `true` | force | Attribution header in API requests |
| `tengu_ant_attribution_header_new` | `true` | force | New Anthropic attribution header format |

#### Feedback & Surveys

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_feedback_survey_config` | min time / frequency config | default | Feedback survey timing configuration |
| `tengu_good_survey_transcript_ask_config` | `{probability: 0}` | default | Probability of asking for good transcript feedback |
| `tengu_bad_survey_transcript_ask_config` | `{probability: 0}` | default | Probability of asking for bad transcript feedback |
| `tengu_negative_interaction_transcript_ask_config` | `{probability: 0}` | default | Negative interaction survey config |
| `tengu_post_compact_survey` | `false` | default | Survey after context compaction |
| `tengu_dunwich_bell` | `false` | default | Memory survey gate |
| `tengu_accept_with_feedback` | `true` | force | Accept action with inline feedback option |

#### Onboarding & Promotions

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_flint_harbor` | `false` | experiment | Onboarding guide A/B experiment |
| `tengu_flint_harbor_prompt` | full prompt template | default | Team onboarding guide generation prompt |
| `tengu_vscode_onboarding` | `false` | default | VSCode extension onboarding flow |
| `tengu_vscode_review_upsell` | `false` | default | VSCode code review upsell |
| `tengu_desktop_upsell` | `{enable_shortcut_tip:true, enable_startup_dialog:false}` | force | Desktop app upsell configuration |
| `tengu_penguin_mode_promo` | `{discountPercent:0, endDate:"Feb 16"}` | default | Penguin mode promotional offer |
| `tengu_year_end_2025_campaign_promo` | `false` | default | Year-end 2025 campaign promo |
| `tengu_birthday_hat` | `false` | default | Birthday celebration UI |
| `tengu-top-of-feed-tip` | `{tip:"", color:""}` | default | Top-of-feed tip banner |
| `tengu_prompt_suggestion` | `true` | force | Prompt suggestion feature |
| `tengu_chomp_inflection` | `false` | default | Prompt suggestion chomping behavior |

#### A/B Experiments

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_plank_river_frost` | `"user_intent"` | experiment | How Claude interprets ambiguous requests: "user_intent" vs "stated_intent" |
| `tengu_amber_prism` | `true` | experiment | Message filtering experiment |
| `tengu_kv7_prompt_sort` | `false` | experiment | Prompt sorting strategy experiment |

#### Penguin Mode & Special Modes

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_penguins_enabled` | `true` | default | Penguin mode enablement |
| `tengu_lodestone_enabled` | `true` | default | Deep link handling (lodestone protocol) |
| `tengu_surreal_dali` | `true` | force | Remote triggers and agent scheduling |
| `tengu_kairos_cron` | `true` | force | KAIROS cron job scheduling |
| `tengu_kairos_cron_durable` | `false` | default | Durable KAIROS cron scheduling |
| `tengu_sumi` | `true` | force | Sumi feature (ink-based rendering) |
| `tengu_oboe` | `true` | default | Oboe feature |
| `tengu_auto_mode_config` | `{enabled: "disabled"}` | force | Auto mode configuration (currently disabled) |

#### Remaining / Obscured Purpose

| Flag | Value | Source | Purpose |
|------|-------|--------|---------|
| `tengu_cobalt_heron` | `false` | default | Legacy/deprecated feature |
| `tengu_basalt_kite` | `false` | default | Unknown — disabled |
| `tengu_slate_finch` | `false` | default | Unknown — disabled |
| `tengu_copper_lantern` | `false` | default | Unknown — disabled |
| `tengu_brass_pebble` | `false` | default | Unknown — disabled |
| `tengu_miraculo_the_bard` | `false` | default | Unknown — disabled |
| `tengu_ember_latch` | `false` | default | Unknown — disabled |
| `tengu_walrus_canteen` | `false` | default | Unknown — disabled |
| `tengu_onyx_wren` | `false` | default | Unknown — disabled |
| `tengu_quill_anvil` | `false` | default | Tool anvil implementation v1 |
| `tengu_quill_anvil_v2` | `false` | default | Tool anvil implementation v2 |
| `tengu_hawthorn_steeple` | `false` | default | Unknown — disabled |
| `tengu_amber_wren` | `{targetedRangeNudge:true, maxTokens:10000}` | default | Targeted range nudge with 10k token max |
| `tengu_ember_quill` | `false` | default | Unknown — disabled |
| `tengu_amber_swift` | `false` | default | Unknown — disabled |
| `tengu_basalt_wren` | `false` | default | Unknown — disabled |
| `tengu_vinteuil_phrase` | `true` | force | Unknown — enabled via force |
| `tengu_cobalt_compass` | `true` | default | Compass feature |
| `tengu_slate_reef` | `false` | default | Unknown — disabled |
| `tengu_pewter_ledger` | `"OFF"` | force | Ledger/logging feature (explicitly OFF) |
| `tengu_sage_compass` | `{}` | default | Unknown — disabled |
| `tengu_garnet_loom` | `false` | default | Unknown — disabled |
| `tengu_olive_hinge` | `""` | default | Unknown — empty/disabled |
| `tengu_maple_forge_w8k` | `false` | default | Unknown — disabled |
| `tengu_onyx_plover` | `{enabled:false, minHours:24, minSessions:3}` | default | Session-gated feature (requires 24h and 3 sessions) |
| `tengu_slate_ridge` | `true` | default | Unknown — enabled |
| `tengu_tst_hint_m7r` | `false` | default | Test hint feature |
| `tengu_coral_whistle` | `false` | default | Unknown — disabled |
| `tengu_blue_coaster` | `false` | default | Unknown — disabled |
| `tengu_velvet_anchor` | `false` | default | Unknown — disabled |
| `tengu_gleaming_fair` | `true` | force | Unknown — enabled via force |
| `tengu_cork_lantern` | `false` | default | Unknown — disabled |
| `tengu_jasper_finch` | `false` | default | Unknown — disabled |
| `tengu_garnet_plover` | `false` | default | Unknown — disabled |
| `tengu_amber_lark` | `false` | default | Model capability experiment |
| `tengu_turnip_cathedral` | `false` | default | Unknown — disabled |
| `tengu_marble_whisper2` | `true` | default | Unknown — enabled |
| `tengu_marble_whisper` | `true` | default | Unknown — enabled |
| `tengu_slate_heron` | `{enabled:false, gapThresholdMinutes:60, ...}` | default | Gap-based session splitting config |
| `tengu_sepia_heron` | `false` | force | Unknown — force disabled |
| `tengu_orchid_trellis` | `false` | default | Unknown — disabled |
| `tengu_quiet_hollow` | `true` | default | Quiet/silent mode for background ops |
| `tengu_quiet_fern` | `true` | default | Quiet mode feature |
| `tengu_tern_alloy` | `"off"` | default | Unknown — off |
| `tengu_pyrite_wren` | `false` | default | Unknown — disabled |
| `tengu_marble_anvil` | `true` | force | Unknown — enabled via force |
| `tengu_cobalt_lantern` | `true` | force | Background task preconditions |
| `tengu_amber_redwood` | `""` | default | Model capability feature — empty |
| `tengu_amber_quartz` | `true` | force | A/B experiment feature |
| `tengu_tangerine_ladder_boost` | `true` | default | Unknown — enabled |
| `tengu_snippet_save` | `false` | default | Snippet saving feature |
| `tengu_copper_wren` | `false` | default | Unknown — disabled |
| `tengu_amber_coil_r4n` | `false` | default | Experiment — disabled |
| `tengu_red_coaster` | `false` | default | Unknown — disabled |
| `tengu_flint_heron` | `false` | default | Unknown — disabled |
| `tengu_swinburne_dune` | `true` | force | Unknown — enabled via force |
| `tengu_tide_elm` | `"off"` | default | Unknown — off |
| `tengu_cobalt_frost` | `true` | force | Unknown — enabled via force |
| `tengu_chair_sermon` | `false` | default | Unknown — disabled |
| `tengu_birch_mist` | `true` | force | Unknown — enabled via force |
| `tengu_slate_moth` | `false` | default | Transcript generation |
| `tengu_onyx_basin_m1k` | `false` | default | Unknown — disabled |
| `tengu_scratch` | `false` | default | Scratchpad feature |
| `tengu_frozen_nest` | `false` | default | Unknown — disabled |
| `tengu_mulberry_fog` | `false` | default | Unknown — disabled |
| `tengu_bergotte_lantern` | `false` | default | Unknown — disabled |
| `tengu_silver_lantern` | `false` | default | Unknown — disabled |
| `tengu_cork_m4q` | `true` | force | Unknown — enabled via force |
| `tengu_slate_nexus` | `true` | force | Transcript storage/management |
| `tengu_billiard_aviary` | `false` | default | Unknown — disabled |
| `tengu_borax_j4w` | `false` | default | Experiment — disabled |
| `tengu_immediate_model_command` | `false` | default | Immediate model command execution |
| `tengu_lichen_compass` | `false` | default | Unknown — disabled |
| `tengu_gypsum_kite` | `false` | default | Unknown — disabled |
| `tengu_shale_finch` | `false` | default | Unknown — disabled |
| `tengu_plum_vx3` | `true` | force | Unknown — enabled via force |
| `tengu_pebble_leaf_prune` | `false` | default | Unknown — disabled |
| `tengu_keybinding_customization_release` | `true` | force | Custom keybinding feature release |
| `tengu_laurel_crown` | `false` | default | Unknown — disabled |
| `tengu_hayate` | `false` | default | Unknown — disabled |
| `tengu_opus_default_pro_plan` | `false` | default | Default Opus on Pro plan |
| `tengu_scarf_coffee` | `false` | default | Unknown — disabled |

**Total flags: 207** | **Enabled: ~90** | **Disabled: ~117** | **A/B Experiments: 4**
