# NGINX Proxy Manager + AdGuard + Wildcard Certificate

**Project Cerberus Homelab · Internal Reverse Proxy, DNS and Certificate Standardisation**

---

## Overview

This project covers the design and implementation of a cleaner internal access model across the homelab, moving away from raw IP addresses and ports toward a properly structured reverse proxy layer with internal DNS rewrites and a trusted wildcard certificate.

The goal was not just to make services accessible by name. The objective was to build something repeatable, a pattern that could be applied to any new internal service without having to rebuild the access logic from scratch each time.

By the end of the project, the following was working cleanly:

```
https://wazuh.cerberus.home.arpa
https://adguard.cerberus.home.arpa
```

Instead of:

```
https://10.0.20.6:8443
http://10.0.20.53:3000
```

---

## Environment

| Component | Detail |
|---|---|
| Proxmox Host | DEKU |
| NGINX Proxy Manager LXC | 10.0.20.7 |
| Wazuh Server | 10.0.20.6 (port 8443, HTTPS backend) |
| AdGuard Home | 10.0.20.53 (port 3000, HTTP backend) |
| Primary FQDN | wazuh.cerberus.home.arpa |
| Additional FQDN | adguard.cerberus.home.arpa |
| Wildcard Certificate Scope | *.cerberus.home.arpa |
| Certificate Tool | mkcert (via Chocolatey on Windows) |

---

## Why This Was Needed

The starting point was a working but manual setup. Wazuh had been configured through a CLI-based NGINX reverse proxy directly on the NGINX server. That worked, but it became clear quickly that managing everything through hand-written `.conf` files was going to become more painful than helpful as more internal services were added.

At the same time, the environment was using `.home.arpa` as the internal domain, which ruled out Let's Encrypt. That meant certificate handling needed its own solution, one that could cover multiple internal services under the same namespace without generating a separate certificate for each one.

The three problems to solve were:

1. Move from CLI NGINX config management to a proper web-managed reverse proxy
2. Standardise internal DNS so services are reachable by FQDN rather than IP and port
3. Implement a wildcard certificate that could be reused across all internal services under `*.cerberus.home.arpa`

---

## What Was Built

### DNS Layer, AdGuard Home Rewrites

AdGuard Home was used as the internal DNS control plane. Two rewrite entries were configured to point both FQDNs at the NGINX Proxy Manager frontend IP rather than directly at the backend services:

```
wazuh.cerberus.home.arpa   → 10.0.20.7
adguard.cerberus.home.arpa → 10.0.20.7
```

This means client devices always hit the reverse proxy first. The proxy then routes traffic to the correct backend based on the hostname in the request.

---

### Reverse Proxy Layer, NGINX Proxy Manager

The old CLI-based Wazuh NGINX configuration was retired cleanly before setting up NPM. This included removing the old `.conf` file, the old certificate and key material, and confirming there was no conflicting config still trying to handle the Wazuh hostname.

NGINX Proxy Manager then became the single point of control for both proxy hosts.

**Wazuh proxy host:**

| Setting | Value |
|---|---|
| Domain | wazuh.cerberus.home.arpa |
| Forward IP | 10.0.20.6 |
| Forward Port | 8443 |
| Backend Scheme | HTTPS |

**AdGuard proxy host:**

| Setting | Value |
|---|---|
| Domain | adguard.cerberus.home.arpa |
| Forward IP | 10.0.20.53 |
| Forward Port | 3000 |
| Backend Scheme | HTTP |

---

### Certificate Layer, mkcert Wildcard

A wildcard certificate was generated for `*.cerberus.home.arpa` using mkcert, installed via Chocolatey and PowerShell on Windows.

```powershell
# Install mkcert
choco install mkcert

# Install and trust the local CA
mkcert -install

# Generate the wildcard certificate
mkcert "*.cerberus.home.arpa"
```

The generated certificate and key were imported into NGINX Proxy Manager and assigned to the relevant proxy hosts. This gave a single reusable certificate covering all first-level subdomains under the internal namespace.

---

## Troubleshooting, What Broke and Why

### Issue 1, Wazuh Login Loop

After Wazuh was configured behind NPM, the login page loaded correctly but authentication would not complete. Entering credentials returned the browser to the login screen with no error.

**Investigation path:**

- Checked NPM proxy host config, backend details looked correct
- Reviewed the Details tab (backend connection) vs the SSL tab (browser-facing connection)
- Found that although the backend was configured as HTTPS, the NPM host list showed HTTP Only on the frontend
- This meant the connection path was: `Browser → HTTP → NPM → HTTPS → Wazuh`

**Root cause:**

For a session and cookie-based application like Wazuh, a mixed HTTP/HTTPS path is enough to cause authentication to fail. The session cookie set by the backend over HTTPS was not being carried correctly back to the browser over HTTP.

**Fix:**

Applied the wildcard certificate to the Wazuh proxy host in the SSL tab, making the frontend HTTPS. The login loop cleared immediately after this change.

**Key lesson:**

> The Details tab in NPM controls how it talks to the backend. The SSL tab controls how the browser talks to NPM. They are independent settings and both matter.

---

### Issue 2, AdGuard Backend Scheme

When first configuring the AdGuard proxy host, NPM was set to forward to the AdGuard backend over HTTPS. This caused connection failures.

Checking the listening state on the AdGuard LXC confirmed it was listening on TCP port 3000 but speaking plain HTTP, not HTTPS.

```bash
ss -tulpen | grep 3000
# tcp LISTEN ... *:3000 ... users:(("AdGuardHome",...))
```

**Fix:**

Corrected the NPM proxy host backend scheme from HTTPS to HTTP.

**Key lesson:**

> The port tells you where a service is listening. The scheme tells you how it is speaking. These are not the same thing and assuming one from the other will cause issues.

---

### Issue 3, Browser Security Warning After Fix

Even after the wildcard certificate was applied and Wazuh login was working correctly, the browser was still showing a security warning indicating the site was not secure.

Testing in incognito and private sessions looked cleaner, which pointed to browser state rather than a live infrastructure issue.

**Investigation using DevTools:**

- Console and Network tabs showed 401 Unauthorized API calls on the login page, these were expected, as the login page calls protected endpoints before authentication completes
- Security tab showed the certificate was valid and trusted, but included a message indicating that active content with certificate errors had previously been allowed to run

**Root cause:**

The browser was remembering earlier testing sessions when the site had certificate problems. Even though the live configuration was now correct, the browser was still treating the origin as tainted.

**Fix:**

Full browser cache clear for all time, followed by a browser restart. After this the site loaded as fully secure, the wildcard certificate displayed correctly, and the warning was gone.

**Key lesson:**

> Browser state can actively mislead troubleshooting. If the infrastructure looks correct but the browser is still complaining, check whether the browser itself is the problem before continuing to dig into the config.

---

## Final Working State

| Service | FQDN | Status |
|---|---|---|
| Wazuh | https://wazuh.cerberus.home.arpa | ✅ Secure, login working |
| AdGuard | https://adguard.cerberus.home.arpa | ✅ Secure, accessible |

Both services:
- Reachable via FQDN through AdGuard DNS rewrites
- Routed through NGINX Proxy Manager
- Serving frontend HTTPS via the `*.cerberus.home.arpa` wildcard certificate
- Showing as fully secure in the browser

---

## Reusable Pattern, Onboarding a New Internal Service

The output of this project is not just two working proxy hosts. It is a repeatable pattern that can be applied to any new internal service:

1. Add a DNS rewrite in AdGuard pointing the new FQDN at `10.0.20.7`
2. Create a new proxy host in NPM with the correct forward IP, port and backend scheme
3. Assign the existing `*.cerberus.home.arpa` wildcard certificate in the SSL tab
4. Verify the service is accessible by FQDN over HTTPS

No new certificate required. No CLI config file to write. No port to remember.

---

## Skills Demonstrated

- Reverse proxy architecture and NGINX Proxy Manager configuration
- Internal DNS design and rewrite management using AdGuard Home
- Internal PKI and wildcard certificate creation using mkcert
- HTTPS troubleshooting across frontend and backend layers
- Browser DevTools-based investigation including Console, Network and Security tabs
- Layered fault isolation across DNS, reverse proxy, certificates, application config and browser state
- Structured troubleshooting methodology, each issue identified, isolated and resolved independently
- Building reusable patterns rather than one-off fixes

---

## Full Documentation

The complete project writeup including full context, environment details, and extended troubleshooting narrative is available in the [Project Cerberus Notion workspace](https://www.notion.so/339fe4fa7fd380739e48d52210a6848d).

---

## Related Projects

- [Project Cerberus, Infrastructure Overview](https://jburke-labs.github.io/project-cerberus)
- [WireGuard VPN, Double NAT Deployment](https://github.com/jburke-labs/wireguard-vpn-deployment)
- [Snipe-IT Replatform, Docker Host Migration](https://github.com/jburke-labs/snipe-it-replatform)
- [Wazuh SOC Automation Pipeline](https://github.com/jburke-labs/wazuh-soc-automation)
