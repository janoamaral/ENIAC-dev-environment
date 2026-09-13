## Why

With `CODER_REDIRECT_TO_ACCESS_URL=true`, Coder answers every proxied request with `307 -> CODER_ACCESS_URL`, even when the public request already uses that exact HTTPS URL. Behind Caddy (which terminates TLS and forwards to Coder over plain HTTP on `127.0.0.1:3000`), this produces an infinite redirect loop that makes the Coder UI unreachable through the public endpoint.

## What Changes

- Set `CODER_REDIRECT_TO_ACCESS_URL=false` in `ansible/roles/coder/templates/coder.env.j2` so generated `/etc/coder.d/coder.env` stops forcing redirects to `CODER_ACCESS_URL`.
- Keep `CODER_ACCESS_URL=https://{{ coder_domain }}`, `CODER_HTTP_ADDRESS={{ coder_http_address }}`, and `CODER_TLS_ENABLE=false` unchanged — Caddy remains the TLS termination point and public entry.
- Document the Caddy/Coder reverse-proxy responsibility split in `ansible/README.md`, including why `CODER_REDIRECT_TO_ACCESS_URL=false` is intentional.
- No changes to the Caddyfile template, ports, firewall, or any other role. The existing `Restart Coder` handler already covers the env-file change.

## Capabilities

### New Capabilities
- `coder-reverse-proxy`: Expected behavior of Coder and Caddy when Coder is published through a reverse proxy — Caddy owns TLS and public ports, Coder stays localhost-only and must not issue access-URL redirects that loop behind the proxy.

### Modified Capabilities

(none — `openspec/specs/` only contains `cockpit-provisioning`, which is unaffected)

## Impact

- `ansible/roles/coder/templates/coder.env.j2` — one line changes (`true` -> `false`).
- `ansible/README.md` — new/updated section documenting proxy responsibility and the redirect setting.
- Runtime effect: next `ansible-playbook site.yml --ask-vault-pass` run updates `/etc/coder.d/coder.env` and restarts Coder via the existing handler; the public HTTPS endpoint stops looping.
- No API, dependency, or infrastructure topology changes.
