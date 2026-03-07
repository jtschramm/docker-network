# Overview

I wanted to set up a combination of Caddy for reverse proxy with Authelia and CrowdSec handling security for my self-hosted applications. I was still confused with the available resources for creating this stack, so it took a lot of trial and error to get things working. Hopefully this guide saves someone else that trouble. Credit to [Genie0720](https://github.com/genie0720/genieaj_homelab_stacks/) and their homelab stack for the best explanations.

**Prerequisites:** Docker and Docker Compose v2 installed on your host. Clone this repository and work from within each service's directory as you go.

---

# Initial Setup

Before bringing the stack up, there are a few values that need to be filled in first.

**Authelia secrets:** For each value in the `secrets` section of `authelia/config/configuration.yml`, generate a unique random string with:
```bash
docker run --rm authelia/authelia:latest authelia crypto hash generate argon2 --random --random.length 64 --random.charset alphanumeric
```

**Authelia domain:** Replace all `website.com` entries in `authelia/config/configuration.yml` with your actual domain.

Once those are filled in, bring the full stack up:
```bash
docker compose up -d
```

---

# Caddy

This uses a maintained Docker image that automatically incorporates the `caddy-bouncer` module that CrowdSec needs to communicate with Caddy.

> **Note:** The `.env` file contains a CrowdSec bouncer API key that you won't have until the CrowdSec setup step — leave it placeholder for now and come back to it then.

In the `Caddyfile`, notice a few things:
- The admin port is open so that dashboard services can access it.
- The Authelia rule is bypassed for anything on the LAN and the local Docker IP range.
- The `site_defaults` snippet ensures all services are passed through CrowdSec and Authelia, and logs interactions to the Caddy access log for CrowdSec to monitor.

Don't point traffic at it just yet — we need to set up the other services first.

---

# Authelia

After the stack is up, Authelia will have generated its remaining config files including `config/users_database.yml`. You can now set your username there and generate a password hash with:

```bash
docker run -v ./authelia/configuration.yml:/configuration.yml --rm authelia/authelia:latest authelia crypto hash generate --config /configuration.yml --password 'your-password'
```

> **Note:** Run this command from the root of the repository so the config file path resolves correctly.

Paste the resulting hash into the `password` field in `users_database.yml` (in quotes).

You may also want to adjust access control rules as needed — for example, bypassing 2FA for certain services on your LAN. For email-based 2FA, you can use a Gmail account by generating an app password at: https://myaccount.google.com/apppasswords

Restart Caddy to apply any changes:
```bash
docker compose restart authelia
```

---

# CrowdSec

Once the stack is up, connect CrowdSec to the Caddy bouncer:
```bash
docker exec crowdsec cscli bouncers add caddy
```

Copy the generated API key into the `CROWDSEC_API_KEY` value in `caddy/.env`, then restart Caddy to pick it up:
```bash
docker compose restart caddy
```

> **Important:** If you ever need to restart the full stack with `compose down`, you may need to re-add the bouncer and update the `.env` with the new key, as the old key will be invalidated. Where possible, restart services individually to avoid this (e.g. `docker compose restart caddy`).

If you plan on using a local dashboard such as [CrowdSec Web UI](https://github.com/TheDuffman85/crowdsec-web-ui) or [Homepage](https://gethomepage.dev/), you'll need to whitelist your local IP address in `config/config.yaml` so the dashboard can reach the CrowdSec API. Add your local subnet under the `api.server.trusted_ips` section, for example:

```yaml
api:
  server:
    trusted_ips:
      - 127.0.0.1
      - 192.168.1.0/24
```

When configuring your dashboard, you'll need the local API URL and credentials — these can be found in `crowdsec/config/local_api_credentials.yaml` after the service has started for the first time.

---

# WG-Easy

Visit `http://<your-server-ip>:<port>` to finish setup and create your first tunnel.

> **Note:** If your router's gateway is not `192.168.1.1`, update the `WG_DEFAULT_DNS` or gateway value in the compose file accordingly.

---

# Final Steps

Check CrowdSec's logs to confirm everything is parsing correctly, then you can start enabling services.

1. **DNS:** Point your domain's DNS records at your WAN IP. Cloudflare is a good choice here as it also offers DDoS protection and easy port management.
2. **Port forwarding:** On your router, forward the following ports to your server:
   - `51820/udp` — WireGuard
   - `80/tcp` — HTTP (Caddy handles redirect to HTTPS)
   - `443/tcp` — HTTPS
3. **Split DNS / DNS override:** Set up a local DNS override for `*.website.com` pointing to your server's LAN IP. Without this, clients connected via WireGuard will resolve your domain to your WAN IP and their traffic will hairpin out to the internet and back — this can cause connectivity issues depending on your router's NAT loopback support, and defeats the purpose of the VPN tunnel. Most routers and DNS servers (pfSense, AdGuard, Pi-hole, etc.) support this natively.

**Verification checklist:**
- Access your services from inside the local network — you should pass through without 2FA prompting.
- Connect via WireGuard and confirm the same services are accessible.
- On mobile data (no LAN, no VPN), browse to a protected service — you should be redirected to `auth.website.com`.
- Run the following to confirm CrowdSec is parsing both the Caddy and Authelia logs:

```bash
docker exec crowdsec cscli metrics
```

You should see output similar to this — confirm both sources appear in the `Lines parsed` column. If Authelia logs are missing, wait a minute and retry (it may not have generated any log lines yet):

```
+--------------------------------------------------------------------------------------------------------------------------+
| Acquisition Metrics                                                                                                      |
+--------------------------------+------------+--------------+----------------+------------------------+-------------------+
| Source                         | Lines read | Lines parsed | Lines unparsed | Lines poured to bucket | Lines whitelisted |
+--------------------------------+------------+--------------+----------------+------------------------+-------------------+
| docker:authelia                | 672        | 672          | -              | -                      | -                 |
| file:/var/log/caddy/access.log | 3.20k      | 3.20k        | 4              | 308                    | 2.60k             |
+--------------------------------+------------+--------------+----------------+------------------------+-------------------+
```

You can also verify CrowdSec's overall health using the official [health check documentation](https://docs.crowdsec.net/u/getting_started/health_check/).

That's it — enjoy a more secure homelab!
