## 1. Fix the Coder environment template

- [x] 1.1 In `ansible/roles/coder/templates/coder.env.j2`, change `CODER_REDIRECT_TO_ACCESS_URL=true` to `CODER_REDIRECT_TO_ACCESS_URL=false`, leaving `CODER_ACCESS_URL`, `CODER_HTTP_ADDRESS`, `CODER_TLS_ENABLE`, and `CODER_PG_CONNECTION_URL` untouched
- [x] 1.2 Verify the template task still notifies the existing `Restart Coder` handler (no handler changes expected)

## 2. Document the reverse-proxy responsibility split

- [x] 2.1 In `ansible/README.md`, add a section documenting the split: Caddy owns public ports 80/443, TLS certificates, and the HTTP-to-HTTPS redirect; Coder listens only on `127.0.0.1:3000`, does not manage TLS, and does not force redirects to `CODER_ACCESS_URL`
- [x] 2.2 In the same README, state that `CODER_REDIRECT_TO_ACCESS_URL=false` is intentional because Caddy already handles the public HTTP-to-HTTPS redirect
- [x] 2.3 Update the sample `/etc/coder.d/coder.env` snippet in the "Important configuration files" section to include `CODER_REDIRECT_TO_ACCESS_URL=false`

## 3. Apply and validate on the VPS

- [x] 3.1 Run `ansible-playbook site.yml --syntax-check --ask-vault-pass`, then `ansible-playbook site.yml --ask-vault-pass`; confirm the env template task reports changed and the `Restart Coder` handler runs
- [x] 3.2 Verify on the VPS: `sudo grep CODER_REDIRECT_TO_ACCESS_URL /etc/coder.d/coder.env` outputs `CODER_REDIRECT_TO_ACCESS_URL=false`
- [x] 3.3 Verify `sudo systemctl status coder` shows the service active after the restart
- [x] 3.4 Verify `curl -IL https://<coder_domain>/` reaches a real Coder UI/login/setup response without repeating `307` redirects to the same URL
- [x] 3.5 Verify in a browser that `https://<coder_domain>` loads normally with the Caddy-managed certificate
