# t3code-no-telemetry

[T3 Code](https://github.com/pingdotgg/t3code) sends product analytics by default, and its Settings page has no switch to turn them off. This repo turns them off with T3 Code's own configuration: a managed policy file, environment variables and a hosts file block. You keep the official builds and updates. Nothing is forked or patched.

The findings below were checked against the source at tag [`v0.1.12`](https://github.com/pingdotgg/t3code/tree/v0.1.12), released 2026-11-25. See [Changes in v0.1.12](#changes-in-v0112) for what upstream changed after the November 2026 criticism.

## Quick start (Linux)

```sh
git clone https://github.com/Project516/t3code-no-telemetry
cd t3code-no-telemetry
./t3-no-telemetry
```

The script asks for `sudo` to write to `/etc`. Restart T3 Code afterwards, because the server reads its configuration only at startup:

- Background service: `systemctl --user restart t3code`
- Desktop app: quit it and open it again.

Then check that the server loaded the policy:

```sh
t3 policy show
```

The last line should say that the running server loaded the policy.

The script is safe to run more than once. To undo everything it added, run `./t3-no-telemetry --uninstall` and restart T3 Code.

## What the script changes

| Change | Where | What it stops |
| --- | --- | --- |
| Managed policy with `"limits": { "telemetry": false }` | `/etc/t3code/policy.json` | Product analytics and relay traces from the server and the desktop app. Environment variables cannot turn them back on. |
| `T3CODE_TELEMETRY_ENABLED=false` and `T3CODE_OTEL_SDK_DISABLED=true` | systemd drop-in for `t3code.service`, if the service exists | The same, as a second layer. |
| The same two variables | `~/.bashrc` and `~/.zshrc`, if they exist | The desktop app reads them from your login shell. |
| `0.0.0.0` and `::` entries for `ingest.t3.codes` and `us-east-1.aws.edge.axiom.co` | `/etc/hosts` | Relay traces from a browser on this machine when it uses the hosted web app, or any T3 Code older than v0.1.12. No setting reaches those. |

If `/etc/t3code/policy.json` already exists and does not turn telemetry off, the script stops and asks you to add `"telemetry": false` under `"limits"` yourself. It does not overwrite a policy it did not write.

The policy sets the organization name to `t3code-no-telemetry`. Settings shows "t3code-no-telemetry turned off anonymous usage data". The policy locks no other settings.

## What T3 Code sends

### Product analytics

Analytics are on by default, and Settings still has no switch for them. The server buffers events and sends a batch every 10 seconds to `https://ingest.t3.codes/batch/` ([AnalyticsService.ts#L46](https://github.com/pingdotgg/t3code/blob/v0.1.12/apps/server/src/telemetry/AnalyticsService.ts#L46), [#L70-L75](https://github.com/pingdotgg/t3code/blob/v0.1.12/apps/server/src/telemetry/AnalyticsService.ts#L70-L75)). The code comment at the top of that file says the ingest server keeps each event and forwards it to PostHog.

Analytics stay off when any of these is true ([#L130-L144](https://github.com/pingdotgg/t3code/blob/v0.1.12/apps/server/src/telemetry/AnalyticsService.ts#L130-L144)):

- A managed policy sets `telemetry: false`.
- `T3CODE_TELEMETRY_ENABLED=false` is set.
- `DO_NOT_TRACK=1` is set.
- The server runs on a CI runner (`CI`, `GITHUB_ACTIONS` and similar), unless `T3CODE_TELEMETRY_ENABLED=true`.

Events include server heartbeats, thread and turn starts, completed provider requests, MCP tool calls, delegated tasks and feature use (for example fork, archive, pin and rollback). Each event carries the OS, CPU architecture, WSL distro name, T3 Code version, release channel and server mode ([#L209-L241](https://github.com/pingdotgg/t3code/blob/v0.1.12/apps/server/src/telemetry/AnalyticsService.ts#L209-L241)). The [upstream telemetry doc](https://github.com/pingdotgg/t3code/blob/v0.1.12/docs/user/telemetry.md) lists the provider, model, reasoning effort, permission mode, results, durations and token counts. A model that is not in a provider's public catalog is reported as `custom:<provider>`, not by name ([ProviderDimensions.ts#L64-L71](https://github.com/pingdotgg/t3code/blob/v0.1.12/apps/server/src/telemetry/ProviderDimensions.ts#L64-L71)).

### Identifiers

- Every event carries a hash of a random install ID kept in `~/.t3/userdata/anonymous-id` ([Identify.ts#L140](https://github.com/pingdotgg/t3code/blob/v0.1.12/apps/server/src/telemetry/Identify.ts#L140)).
- An `install.owner_observed` event reports a hash of your T3 Connect account ID when the server is linked to one ([#L151-L164](https://github.com/pingdotgg/t3code/blob/v0.1.12/apps/server/src/telemetry/Identify.ts#L151-L164)). Connecting clients add a device ID.

The hashes are plain SHA-256 with no salt ([telemetryId.ts#L8-L9](https://github.com/pingdotgg/t3code/blob/v0.1.12/packages/shared/src/telemetryId.ts#L8-L9)). T3 can match the Connect account hash to the account, since it has the raw ID.

### Agent feedback

v0.1.12 adds agent feedback: agents can report problems with T3 Code to the T3 team. It is off by default under **Settings > Labs > Agent feedback**. Reports are sent as analytics events ([FeedbackStore.ts#L201-L208](https://github.com/pingdotgg/t3code/blob/v0.1.12/apps/server/src/feedback/FeedbackStore.ts#L201-L208)), so the policy and variables above stop them too. With telemetry off, reports stay on the machine.

### Relay traces

T3 Code sends OpenTelemetry traces for T3 Connect connections to Axiom. The release server and web builds include the Axiom endpoint, and the web build also includes an ingest token.

- **Server.** It stops when a policy sets `telemetry: false`, or when `T3CODE_TELEMETRY_ENABLED=false`, `DO_NOT_TRACK=1` or `T3CODE_OTEL_SDK_DISABLED=true` is set ([relayTracing.ts#L21-L27](https://github.com/pingdotgg/t3code/blob/v0.1.12/apps/server/src/cloud/relayTracing.ts#L21-L27)).
- **Desktop app window.** It asks the desktop app, which checks the same policy and variables.
- **Browser on a page your server serves.** It exports only when that server allows relay tracing ([publicConfig.ts#L73-L79](https://github.com/pingdotgg/t3code/blob/v0.1.12/apps/web/src/cloud/publicConfig.ts#L73-L79)), so the server's policy covers it.
- **Hosted web app** (`app.t3.codes`). It always exports when its build has the Axiom config. No setting reaches it. Block the host to stop it.
- **Mobile app.** Turn off **Send connection traces** in **Settings > About T3 Code > Diagnostics**. The server's policy and variables do not reach the phone.

### Mobile crash reports

The iOS and Android apps send crash reports and startup timings through the T3 Connect relay. Turn them off in the app under **Settings > About T3 Code > Diagnostics > Send crash reports**.

## What this repo cannot turn off

- **The phone app.** Use its two switches, described above.
- **The hosted web app on other devices.** The hosts block only covers the machine you run the script on. Run the script on each computer where you use `app.t3.codes`, or block the two hosts on your network's DNS.
- **The T3 Connect relay itself.** If you connect through `relay.t3.codes`, the relay sees your IP address and connection metadata. That is how the feature works. To avoid it, connect directly over your LAN or a VPN.
- **New hosts in later versions.** The hosts block names the two hosts used today. The policy does not depend on host names, so the server and desktop app stay covered. Recheck the browser config after major updates:

  ```sh
  grep -rhoE 'https://[a-z0-9.-]*axiom[a-z0-9./-]*' <t3-install-dir>/client/assets | sort -u
  ```

**Publish agent activity** (**Settings > Sharing & access**) sends project and thread titles to the relay for phone notifications. It is a feature, not telemetry, and has its own switch.

## Changes in v0.1.12

Upstream merged these on 2026-11-24 and 2026-11-25, after posts criticizing the telemetry:

- The server no longer reads `~/.codex/auth.json` or `~/.claude.json`, and no longer sends hashed OpenAI or Claude account IDs. In v0.1.11 it read both files at every startup, even with telemetry off, and used them for the event ID ([#42299](https://github.com/pingdotgg/t3code/pull/42299)).
- `DO_NOT_TRACK=1` turns analytics and relay traces off ([#42629](https://github.com/pingdotgg/t3code/pull/42629)). CI runners send nothing by default ([#42630](https://github.com/pingdotgg/t3code/pull/42630)).
- A browser on a page your server serves follows the server's opt-out. In v0.1.11 it always exported relay traces ([#42356](https://github.com/pingdotgg/t3code/pull/42356)).
- The mobile app has a **Send connection traces** switch ([#42415](https://github.com/pingdotgg/t3code/pull/42415)).
- Custom model names are no longer sent ([#42465](https://github.com/pingdotgg/t3code/pull/42465)).

Analytics are still on by default, and Settings still has no switch for them. [PR #42625](https://github.com/pingdotgg/t3code/pull/42625), which adds one, is open.

## macOS and Windows

The script is Linux only. On other systems, do the same steps by hand. The policy file paths come from the upstream [managed policy doc](https://github.com/pingdotgg/t3code/blob/v0.1.12/docs/user/managed-policy.md).

Policy file contents:

```json
{
  "schemaVersion": 1,
  "organization": "t3code-no-telemetry",
  "limits": { "telemetry": false }
}
```

**macOS**

1. Save the policy as `/Library/Application Support/T3Code/policy.json`, owned by root.
2. Add `export T3CODE_TELEMETRY_ENABLED=false` and `export T3CODE_OTEL_SDK_DISABLED=true` to `~/.zshrc`.
3. Add these lines to `/etc/hosts`:

   ```
   0.0.0.0 ingest.t3.codes
   :: ingest.t3.codes
   0.0.0.0 us-east-1.aws.edge.axiom.co
   :: us-east-1.aws.edge.axiom.co
   ```

4. Quit and reopen T3 Code.

**Windows**

1. Save the policy as `C:\Program Files\T3Code\policy.json`.
2. Set `T3CODE_TELEMETRY_ENABLED=false` and `T3CODE_OTEL_SDK_DISABLED=true` as user environment variables.
3. Add the same four lines to `C:\Windows\System32\drivers\etc\hosts`.
4. Quit and reopen T3 Code.

T3 Code refuses to start if it finds both a native policy (a macOS profile or Windows registry values) and a policy file. Use one or the other.

## Verify

- `t3 policy show` says the running server loaded the policy.
- On Linux, the server process has the variables:

  ```sh
  tr '\0' '\n' < "/proc/$(pgrep -f 't3 serve' | head -1)/environ" | grep -E 'T3CODE_TELEMETRY|OTEL_SDK'
  ```

- `getent hosts ingest.t3.codes` prints `::`.

## Background

- [PR #174](https://github.com/pingdotgg/t3code/pull/174), merged 2026-03-06, added PostHog analytics on by default.
- [Issue #1397](https://github.com/pingdotgg/t3code/issues/1397) asked for an opt-out. It was closed as not planned on 2026-06-23. A maintainer wrote: "We will not make this easier to opt out of, it is crucial for us to know how many users we have".
- [PR #15028](https://github.com/pingdotgg/t3code/pull/15028) moved sending to `ingest.t3.codes`. [PR #21506](https://github.com/pingdotgg/t3code/pull/21506) added the install ID and owner IDs.
- Four PRs that added a Settings toggle were closed without merging: [#13458](https://github.com/pingdotgg/t3code/pull/13458), [#16763](https://github.com/pingdotgg/t3code/pull/16763), [#21248](https://github.com/pingdotgg/t3code/pull/21248) and [#27077](https://github.com/pingdotgg/t3code/pull/27077).

## License

[AGPL-3.0](LICENSE)
