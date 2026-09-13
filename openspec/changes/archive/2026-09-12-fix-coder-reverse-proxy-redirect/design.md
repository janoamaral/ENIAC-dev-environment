## Context

The VPS topology is: Internet -> Caddy (Docker, host network, ports 80/443, TLS via ACME) -> HTTP -> Coder (systemd, `127.0.0.1:3000`). The env template `ansible/roles/coder/templates/coder.env.j2` currently renders `CODER_REDIRECT_TO_ACCESS_URL=true`. Because Caddy forwards over plain HTTP, Coder sees non-HTTPS requests and redirects them to `CODER_ACCESS_URL`; the public request already is that URL, so the browser loops (`307 -> https://<coder_domain>`, via Caddy) until the client gives up. Confirmed fixed in production by flipping the flag to `false` and restarting Coder. The restart-on-change plumbing already exists: the template task notifies the `Restart Coder` handler (ansible/roles/coder/tasks/main.yml:48, ansible/roles/coder/handlers/main.yml).

## Goals / Non-Goals

**Goals:**
- Generated `/etc/coder.d/coder.env` contains `CODER_REDIRECT_TO_ACCESS_URL=false`.
- Public endpoint `https://<coder_domain>/` serves the Coder UI without a redirect loop.
- README documents the Caddy/Coder responsibility split and why the flag is `false`.
- Coder restarts when the env file changes (already satisfied by existing handler — verify, don't rebuild).

**Non-Goals:**
- Enabling TLS in Coder (`CODER_TLS_ENABLE` stays `false`).
- Adding forwarding headers to the Caddyfile (Caddy's reverse_proxy defaults suffice; no evidence they're needed).
- Changing ports, firewall, PostgreSQL, Docker, or any other role.
- Wildcard workspace app domains.

## Decisions

**1. Flip the flag in the template; do nothing else to config.**
Root cause is a single line: `CODER_REDIRECT_TO_ACCESS_URL=true` (coder.env.j2:4). Alternatives considered and rejected: enabling Coder TLS (duplicates Caddy's job, breaks the single-TLS-endpoint design); adding `X-Forwarded-Proto` handling in Caddy (would mask the symptom with extra config; Caddy's docs for this setup don't require it and the working production config didn't need it). The confirmed-working production configuration is exactly the current template with the flag set to `false` — adopt it verbatim.

**2. Keep `CODER_ACCESS_URL=https://{{ coder_domain }}`.**
Coder still needs to know its public URL for links, OAuth callbacks, and workspace app routing; it just must not *redirect* to it.

**3. Reuse the existing `Restart Coder` handler.**
The env template task already `notify: Restart Coder`, so the playbook run applies and restarts in one shot. No new handler, no handler edits.

**4. README documents the responsibility split.**
A short section stating: Caddy owns 80/443, TLS certs, and the HTTP->HTTPS redirect; Coder listens on `127.0.0.1:3000` only, no TLS, no forced access-URL redirect — and that `CODER_REDIRECT_TO_ACCESS_URL=false` is intentional for exactly this reason. Also update the sample env snippet in "Important configuration files", which currently omits the flag.

## Risks / Trade-offs

- [Direct requests to `127.0.0.1:3000` are served over plain HTTP without redirect] → Acceptable: the listener is localhost-only and this path bypasses the proxy on purpose; not a security regression.
- [If Coder is ever moved to serve publicly without Caddy, no redirect to HTTPS exists] → Documented in README; re-enable the flag if the topology ever changes.
- [Redirect-loop class bugs can recur if someone re-adds the flag or enables Coder TLS] → Spec requirement + README note pin the expected behavior.
