# Security policy

## Reporting a vulnerability

Please report security issues privately through GitHub's **Report a vulnerability** button on the Security tab of this repository. Do not open a public issue for them.

## Keeping your setup safe

- Never post your ML API token, Home Assistant URL, IP addresses or printer serial in issues, logs or screenshots.
- Keep the ML container's port 3333 reachable from your LAN only.
- Snapshots in `/config/www/obico` are served at `/local/obico/` without authentication. If Home Assistant is reachable from the internet, block that path on your reverse proxy or tunnel.
- If a token leaks, generate a new one and update both the ML stack's `.env` and the `p1s_ml_auth_header` secret.
