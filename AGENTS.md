# AGENTS.md

## Purpose

Turn off T3 Code telemetry using only T3 Code's own configuration, so users keep the official builds and updates. The README documents what T3 Code sends and which switch covers which part.

## Direction

- Config over patches. Never fork, patch or wrap the T3 Code binary.
- Every claim in the README links to upstream source at a tag. When upstream changes, recheck the claims and move the links to the new tag.
- The script must be safe to rerun, and `--uninstall` must restore files exactly. Never overwrite a policy file the script did not write.
- Plain bash with standard Linux tools. Run `shellcheck t3-no-telemetry` before committing.

## Glossary

- **Managed policy**: T3 Code's admin config file (`/etc/t3code/policy.json` on Linux). `limits.telemetry: false` turns off analytics and relay traces in the server and desktop app, and environment variables cannot turn them back on.
- **Product analytics**: events the server batches to `ingest.t3.codes`, which forwards them to PostHog.
- **Relay traces**: OpenTelemetry spans about T3 Connect connections, sent to Axiom.
- **Marker**: the `# t3-no-telemetry` comment on every line the script adds, which `--uninstall` uses to find them.
