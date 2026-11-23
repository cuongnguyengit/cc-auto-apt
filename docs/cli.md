# CLI guide: Go 1.3

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
