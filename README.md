# claude-rotate

Switches Claude Code to another of your Claude subscriptions before the current one hits its limit.

```
./claude-rotate add-sub            # runs the Claude login flow, stores the account in ./subs/
./claude-rotate add-sub work       # same, with an explicit name
./claude-rotate add-sub --from-current   # store the account Claude Code is logged in with now
./claude-rotate status             # 5h / 7d usage of every sub (* = active)
./claude-rotate use <sub>          # switch now
./claude-rotate check              # one check (cron-friendly)
./claude-rotate daemon             # check every 3 min, switch at 90%
```

How it works
- `add-sub` runs `claude auth login` with a throwaway `CLAUDE_CONFIG_DIR`, so your current login is untouched.
- The daemon reads each sub's usage from `/api/oauth/usage`, the same endpoint `/usage` uses.
  A sub's usage is its highest window, 5h or weekly (7d, 7d-opus/sonnet), so either one can trigger a switch.
  1. Active below `--threshold` (90%): stay.
  2. Active at 90% or more: switch to the least-used sub under threshold minus `--margin` (80%).
  3. No sub under 80%: keep using the active sub up to `--hard` (98%), then drain the leftovers by
     switching to whichever sub has the most left.
  4. Every sub at 98% or more: wait. When a sub's limit resets (5h resets come quickly), it becomes a target again.
  When candidates are close (within 5 points, or 2 while draining), it picks the one whose limit resets
  first, because that headroom expires soonest.
- To switch, it atomically replaces `~/.claude/.credentials.json`. Running Claude Code sessions
  notice the change and pick up the new account without a restart.
- Claude Code refreshes the live token on its own and the refresh token changes each time, so the
  script copies the live file back into the active sub before every switch. It refreshes idle subs'
  tokens itself so they stay valid.
- The usage endpoint rate-limits, so idle subs are checked at most every 10 minutes and the daemon backs off on HTTP 429.

Run as a service: `cp claude-rotate.service ~/.config/systemd/user/ && systemctl --user enable --now claude-rotate`
(or as root: copy it to /etc/systemd/system/ and change `default.target` to `multi-user.target`).

`./subs/` holds live OAuth tokens: keep it private and never commit it.

## Notes

- Linux only: on macOS, Claude Code keeps credentials in the Keychain, not in `~/.claude/.credentials.json`.
- It uses Claude Code's internal OAuth and usage endpoints, which are undocumented and may change.
- Check that your use complies with Anthropic's terms for the subscriptions involved.
- MIT licensed. Not affiliated with Anthropic.
