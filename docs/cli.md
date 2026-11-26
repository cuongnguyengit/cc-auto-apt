# CLI guide: Go 1.3

## Scheduled soft activation / warmup (1.5.0+)

```sh
cc-auto warmup dry-run
cc-auto warmup enable --all --times 05:00,10:00,15:00 --timezone Asia/Ho_Chi_Minh
cc-auto warmup status
cc-auto warmup disable
# Or select particular profiles:
cc-auto warmup enable --profiles work,personal --times 05:00,10:00,15:00 --timezone Asia/Ho_Chi_Minh
```

Warmup is **disabled by default**. Enabling explicitly registers a per-user
Windows Task Scheduler task, Linux systemd user timer, or macOS LaunchAgent.
The schedule checks once a minute. The machine must be awake and user scheduling
must be available (normally logged in; Linux needs a working systemd user bus).
After sleep/offline periods, slots older than the default 15-minute grace period
are not replayed. Set `--grace-minutes 1..30` if needed. Disable warmup before
uninstalling; `remove-integrations` also disables it. `status` includes the next
slots in the chosen IANA timezone, outcomes and measured tokens. Dry-run previews
the current or next slot without quota API calls or inference (Keychain access
may still require permission).

`--all` includes every locally signed-in profile, including accounts outside
router priority. Each eligible account gets at most one application-level CLI
inference attempt per slot: a short Haiku prompt in a private config directory,
no project history/settings, tools, user hooks or MCP. The CLI's managed policy
still applies. It uses `--safe-mode`, not `--bare`, because bare mode does not
use subscription OAuth. Output is capped at 64 tokens, one turn, a 90-second
process timeout and a $0.02 client-estimated budget; this is not a provider-side
billing guarantee. Real token use depends on CLI/model version and is recorded
including prompt cache tokens. A measured local trial used 385 input tokens
(including cache) and 4 output tokens; this is an observation, not a fixed cost.

Warmup respects weekly/routing limits and never submits when quota cannot be
verified. An active five-hour window is skipped; if it ends within the slot's
grace period, the account waits until just after reset. Failed or uncertain CLI
attempts are not retried automatically in that slot. `sent` means the CLI
succeeded but a new reset is not yet verified; `warmed` requires a future reset
reported by the quota endpoint. It cannot force an existing window to restart,
increase weekly allowance, or guarantee three new windows on every day.
Native retries, managed settings and provider behavior are outside the local
scheduler's control. macOS uses the existing Keychain adapter; permission and
live-account behavior still need validation on a user's Mac.

Tokens, native model responses and raw provider errors are never stored in the
warmup log. OAuth refreshes are copied back only if the source login has not
changed; MCP auth and user conversation state stay separate. Configuration is
in `warmup.json`; bounded per-slot history is in `warmup-state.json`.

Provider semantics: [five-hour usage windows](https://support.claude.com/en/articles/11647753-how-do-usage-and-length-limits-work),
[CLI flags](https://code.claude.com/docs/en/cli-reference) and
[OAuth versus bare mode](https://code.claude.com/docs/en/headless#start-faster-with-bare-mode).

## Account identity and subscription details (1.4.0+)

```sh
cc-account list
cc-account list --details
cc-account detail work
cc-account info work --json
cc-account detail --all --json
cc-account list --json
```

Plain list includes cached email/name when available. The details table maps
profile names to email, plan, display name, organization and credential state.
`detail` (aliases `info`, `details`) additionally shows full name, account and
organization IDs, organization type/role, workspace role, billing type, seat and
rate-limit tiers, extra-usage setting, known creation/fetch/trial dates and OAuth
expiry. `--all` covers default and named profiles; `auto` is a routing alias,
not another account. JSON for one profile is an object; all/list is an array.

This is local cached data, not a live billing or subscription lookup. Missing
fields are unknown (omitted in JSON); an expired or missing credential does not
erase cached identity. Plan is the stored subscription type, not inferred from
quota. Keychain may request access on macOS when viewing full details. Plain
list only reads identity metadata and never queries Keychain. Viewing details
does not switch profiles, update files, make network requests or refresh tokens.
Explicit allowlisted fields exclude access/refresh tokens, API keys, MCP auth,
project data and all unknown nested metadata. Email/name are personal data:
review detail output before sharing it in public support reports.

## Session ownership

From 1.3.2, a resumed conversation can reclaim its saved recovery record after
the old managed launcher exits, including after a crash or a missing SessionEnd.
Live owners remain protected. Recovery attempts and pending wakes reset on
handoff; the conversation itself is preserved. From 1.3.3, Linux also inspects
same-user process environments to reclaim legacy folders without owner leases.
A live Claude child remains protected even after its launcher exits. Unreadable
process data remains unknown, rather than being reported as a verified live
owner. Other platforms conservatively protect legacy folders without leases.
`cc-auto recover status --json` includes `owner.state` and matching `owner.pids`;
doctor warns about stale or unknown ownership.

`cc-account -h` / `--help` and `cc-auto -h` / `--help` print complete command
guides and examples. Setup installs shell functions; reopen the shell and open
new VS Code conversations after upgrading integrations.

## Watch

```sh
cc-auto check --watch -n 0.1
cc-auto check --watch --interval 2 --json
```

The minimum interval is 0.1 seconds. This refreshes local state and countdowns;
quota requests continue to honor polling schedules and failure backoff. Ctrl+C
restores the terminal. JSON watch emits JSON Lines; redirected text has no ANSI
screen controls. Ordinary `check --json` remains one pretty-printed object.

## Quota thresholds

```sh
cc-account limit show work
cc-account limit set global --5h 95 --week 85
cc-account limit set work --5h 90 --week 80
cc-account limit set global --switch 95 --return 90
cc-account limit pace work --week-min 10 --week-max 85 --cycle-days 7
cc-account limit reset work
```

These are account selection rules, not changes to Anthropic's actual limits.
Fixed meter caps override generic switch/return thresholds for that meter and
apply to both active accounts and candidates. Profile meter caps override global
meter caps. A paced profile rule takes precedence over a fixed/global cap for its
meter, rising linearly from minimum to maximum through the cycle inferred from
the quota reset timestamp. Setting a fixed cap removes the corresponding paced
rule. Generic return must be positive and no greater than switch, and all fixed
percentages must be greater than zero and at most 100.

`limit show` includes global and profile rules so inheritance can be inspected;
`check` shows the effective live limits. Resetting a profile removes its overrides;
resetting global restores generic 95/90 and removes global meter caps. Existing
unrelated router options are preserved, and invalid changes are not written.
`cc-auto limit ...` is an alias for the same commands.

## Choosing an algorithm

```sh
cc-account swap strategy sticky
cc-account swap forecast adaptive
cc-account swap cooldown 180
cc-account swap show
```

| Strategy | Selection behavior |
| --- | --- |
| `priority` (default) | First eligible account in priority order, including returning to a higher-priority account |
| `sticky` | Keep the current eligible account; otherwise choose by priority |
| `round-robin` | Keep the current eligible account; otherwise select the next eligible account cyclically in priority order |
| `least-used` | Lowest worst-meter utilization among eligible accounts; retain current if within 2 percentage points of the best |

Every strategy rejects exhausted, locked or unverified accounts. The existing
API-429 cache fallback can use age-discounted headroom when no fresh candidate
exists; stale data is never interpreted as zero usage. The router retains a
signed-in profile when observations are unavailable rather than signing it out.

Forecast is a separate axis: `adaptive` uses robust median utilization slopes,
cycle pressure and confidence from at least three samples to lower effective
caps proactively. `static` disables predictive reductions while still enforcing
fixed/linearly paced caps. Cooldown (0..1800 seconds, default 180) delays
nonurgent switches in both the shared router and managed processes; reaching an
effective limit or handling a failed request bypasses it. Recovery excludes
already failed accounts and has a bounded attempt count regardless of strategy.

Use `cc-account use work` or `run work ...` to pin an account manually. Use
`cc-account use auto` on Windows/Linux, or unset `CLAUDE_CONFIG_DIR` and launch
`cc-account terminal`, for automatic selection. `cc-auto swap ...` shares the
same policy commands. macOS managed processes use these rules with Keychain;
its shared quota daemon is not available.

## Doctor

```sh
cc-account doctor
cc-auto doctor --json
cc-auto doctor --online
```

Doctor checks manager metadata, shared data, policy, native executable startup,
credential readability/expiry, shared links, environment overrides, shell and
VS Code integration, user router service, cache age and recovery records. It
reports `OK`, `WARN` and `FAIL`; required failures exit 1, warnings alone exit 0.
Credential values, environment values and prompt content are never printed.

Offline doctor does not refresh tokens or repair files. `--online` explicitly
runs a quota check and may refresh OAuth credentials and route the shared alias.
Custom shell/settings paths may produce warnings that need manual inspection.
On macOS, credential reads and quota monitoring use Keychain. `check`, `check
--watch`, and `doctor --online` poll accounts with caching/backoff. The displayed
Active account is the configured VS Code profile; managed terminal/VS Code
processes select accounts independently. Monitoring never retargets an auth
alias or changes Keychain namespaces. Approve macOS Keychain access when asked.
Use the suggested setup/login/priority/recovery commands to repair findings.
