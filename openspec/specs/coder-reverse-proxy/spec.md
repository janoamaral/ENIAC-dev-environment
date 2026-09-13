# coder-reverse-proxy Specification

## Purpose

Define the Caddy/Coder reverse-proxy split: Caddy owns public ports, TLS, and the HTTP-to-HTTPS redirect, while Coder listens only on localhost without TLS and must not redirect to its access URL behind the proxy. TBD.

## Requirements

### Requirement: Caddy is the sole TLS termination point
The Ansible provisioning SHALL configure Caddy to own public ports 80 and 443, manage TLS certificates, redirect public HTTP to HTTPS, and proxy to Coder over localhost HTTP. Coder SHALL NOT be configured to enable TLS or listen on a public address.

#### Scenario: Generated Coder environment keeps TLS disabled
- **WHEN** the playbook renders `/etc/coder.d/coder.env` from `ansible/roles/coder/templates/coder.env.j2`
- **THEN** the file contains `CODER_TLS_ENABLE=false` and `CODER_HTTP_ADDRESS=127.0.0.1:3000`, and Caddy proxies to that localhost address

#### Scenario: Caddyfile stays the public entry point
- **WHEN** the playbook renders the Caddyfile
- **THEN** the site block for `{{ coder_domain }}` proxies to Coder on localhost and contains no additional forwarding headers

### Requirement: Coder must not redirect to its access URL behind the proxy
The generated Coder environment SHALL set `CODER_REDIRECT_TO_ACCESS_URL=false` while keeping `CODER_ACCESS_URL=https://{{ coder_domain }}`, because Caddy already performs the public HTTP-to-HTTPS redirect.

#### Scenario: Public HTTPS request is served, not redirected
- **WHEN** a client requests `https://<coder_domain>/` through Caddy
- **THEN** Coder responds with the normal UI/login/setup response instead of `307` pointing back to `CODER_ACCESS_URL`

#### Scenario: No redirect loop through the proxy chain
- **WHEN** `curl -IL https://<coder_domain>/` follows redirects through the public endpoint
- **THEN** the request terminates on a real response and never repeats a `307` to the same URL

#### Scenario: Environment file contains the setting after provisioning
- **WHEN** the playbook has been applied to the VPS
- **THEN** `grep CODER_REDIRECT_TO_ACCESS_URL /etc/coder.d/coder.env` outputs `CODER_REDIRECT_TO_ACCESS_URL=false`

### Requirement: Coder restarts when its environment file changes
The playbook SHALL restart the Coder service whenever the rendered `/etc/coder.d/coder.env` changes, via the existing `Restart Coder` handler.

#### Scenario: Env change triggers restart
- **WHEN** a playbook run changes the content of `/etc/coder.d/coder.env`
- **THEN** the `Restart Coder` handler runs and the service is active afterwards

### Requirement: Reverse-proxy responsibility split is documented
The Ansible README SHALL document that Caddy owns public ports, TLS certificates, and the HTTP-to-HTTPS redirect, while Coder listens only on localhost, does not manage TLS, and does not force redirects to `CODER_ACCESS_URL` — and that `CODER_REDIRECT_TO_ACCESS_URL=false` is intentional for this reason.

#### Scenario: README explains the intentional flag value
- **WHEN** a reader consults the Ansible README
- **THEN** it states the Caddy/Coder responsibility split and explains that `CODER_REDIRECT_TO_ACCESS_URL=false` is deliberate because Caddy already handles the public HTTP-to-HTTPS redirect
