# wisdombusara.com, VPS + Cloudflare Runbook

Deploying a static site (no build step, no server-side code) to a VPS, with the domain on Cloudflare. Follow top to bottom the first time; use the **Updating** and **Troubleshooting** sections thereafter.

---

## 0. What's in the bundle

Extract `wisdombusara-site.tar.gz` and you get the web root:

```
index.html            # the site (self-contained: inline CSS + JS)
robots.txt
sitemap.xml
assets/
  three.min.js        # vendored locally: no third-party runtime calls
```

Notes:
- There is **no CNAME file** and you don't need one. That file is a GitHub Pages artifact; it does nothing on a VPS.
- The only external calls the page makes are to Google Fonts (styling) and `api.github.com` (the live repo panel, from the visitor's browser). Everything else is served by you. Self-hosting the fonts is an optional hardening step at the end.

---

## 1. Prerequisites

- A VPS you can SSH into as root or a sudo user (Ubuntu/Debian assumed below).
- `wisdombusara.com` already added to your Cloudflare account (the domain's nameservers point to Cloudflare).
- Your VPS public IP address. Call it `VPS_IP` below.

---

## 2. Put the files on the server

From your local machine, copy the bundle up:

```bash
scp wisdombusara-site.tar.gz you@VPS_IP:/tmp/
```

Then on the VPS:

```bash
sudo mkdir -p /var/www/wisdombusara
sudo tar xzf /tmp/wisdombusara-site.tar.gz -C /var/www/wisdombusara
sudo chown -R www-data:www-data /var/www/wisdombusara
ls -R /var/www/wisdombusara     # sanity check: index.html, assets/three.min.js, etc.
```

---

## 3. Install nginx

```bash
sudo apt update && sudo apt install -y nginx
```

---

## 4. Get a Cloudflare Origin Certificate

This secures the Cloudflare-to-your-server hop. It's free, valid for 15 years, and trusted by Cloudflare (so it works with the strict SSL mode in step 6). No renewals to worry about.

1. Cloudflare dashboard → your domain → **SSL/TLS → Origin Server → Create Certificate**.
2. Leave defaults (RSA, hostnames `wisdombusara.com, *.wisdombusara.com`). Create.
3. You'll see two text blocks: the **Origin Certificate** and the **Private Key**. Copy each onto the VPS:

```bash
sudo mkdir -p /etc/ssl/cloudflare
sudo nano /etc/ssl/cloudflare/wisdombusara.com.pem   # paste the Origin Certificate
sudo nano /etc/ssl/cloudflare/wisdombusara.com.key   # paste the Private Key
sudo chmod 600 /etc/ssl/cloudflare/wisdombusara.com.key
```

---

## 5. nginx server block

Create `/etc/nginx/sites-available/wisdombusara`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name wisdombusara.com www.wisdombusara.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name wisdombusara.com www.wisdombusara.com;

    root /var/www/wisdombusara;
    index index.html;

    ssl_certificate     /etc/ssl/cloudflare/wisdombusara.com.pem;
    ssl_certificate_key /etc/ssl/cloudflare/wisdombusara.com.key;

    # compression (Cloudflare also compresses at the edge; this helps origin fetches)
    gzip on;
    gzip_min_length 1024;
    gzip_types text/css application/javascript application/json image/svg+xml;

    # security headers (inherited by all locations below, since those use no add_header)
    server_tokens off;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # cache the fingerprint-free way: short for html, long for static assets
    location = /index.html { expires 5m; }
    location /assets/     { expires 1y; }

    location / { try_files $uri $uri/ =404; }
}
```

> On nginx 1.25.1+ you may see a deprecation notice for `listen ... http2`. It still works; if you prefer, drop `http2` from the `listen` lines and add `http2 on;` inside the `server` block.

Enable it, drop the default site, test, reload:

```bash
sudo ln -s /etc/nginx/sites-available/wisdombusara /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

---

## 6. Cloudflare DNS and TLS

**DNS** (dashboard → your domain → DNS → Records). Add two records, both **Proxied** (orange cloud on):

| Type | Name | Content | Proxy |
|------|------|---------|-------|
| A    | `@`  | `VPS_IP` | Proxied |
| A    | `www`| `VPS_IP` | Proxied |

**TLS** (SSL/TLS → Overview): set the encryption mode to **Full (strict)**.
This is the important one. *Flexible* would leave the Cloudflare-to-origin hop unencrypted and cause redirect loops with the config above.

**Edge settings** (SSL/TLS → Edge Certificates): turn on **Always Use HTTPS** and **Automatic HTTPS Rewrites**. (Speed → Optimization: Brotli on, optional.)

Give DNS a couple of minutes, then load `https://wisdombusara.com`.

---

## 7. Firewall, lock the origin to Cloudflare

Because the records are proxied, all real traffic reaches you *from Cloudflare's IPs*. Allow only those to hit 80/443, so nobody can reach your VPS IP directly and bypass Cloudflare:

```bash
sudo ufw allow OpenSSH
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do
  sudo ufw allow proto tcp from $ip to any port 80,443;
done
for ip in $(curl -s https://www.cloudflare.com/ips-v6); do
  sudo ufw allow proto tcp from $ip to any port 80,443;
done
sudo ufw --force enable
sudo ufw status
```

> Keep an SSH session open while enabling the firewall, in case you need to fix a rule.

---

## 8. Verify

```bash
# origin is serving locally
curl -I http://127.0.0.1/ -H "Host: wisdombusara.com"      # expect 301 to https
# through Cloudflare
curl -I https://wisdombusara.com                            # expect 200
curl -s https://wisdombusara.com/assets/three.min.js | head -c 40   # expect the three.js license header
```

In a browser, confirm: the page loads over HTTPS, the dark/light toggle works, ⌘K opens the command palette, project filters work, and the GitHub panel populates. If the 3D hero mesh doesn't appear, that's non-fatal (it degrades silently), check `/assets/three.min.js` returns 200.

---

## 9. Updating the site later

Static files, so updates are simple. To replace just the page:

```bash
scp index.html you@VPS_IP:/tmp/ && \
  ssh you@VPS_IP 'sudo mv /tmp/index.html /var/www/wisdombusara/index.html && sudo chown www-data:www-data /var/www/wisdombusara/index.html'
```

Or re-extract the whole bundle over the existing root (step 2). No nginx reload is needed for content changes.

After updating, purge the edge cache so visitors get the new version immediately: Cloudflare dashboard → Caching → Configuration → **Purge Everything** (or purge the single URL). The `index.html` cache is only 5 minutes anyway, so it will refresh on its own shortly regardless.

---

## 10. Troubleshooting

| Symptom | Likely cause | Fix |
|--------|--------------|-----|
| Cloudflare **521** (web server is down) | nginx not running, or firewall is blocking Cloudflare | `sudo systemctl status nginx`; confirm ufw rules include the Cloudflare IP ranges (re-run step 7) |
| Cloudflare **522** (timeout) | Port 443 not reachable from Cloudflare | Check ufw allows 443 from Cloudflare IPs; check the VPS provider's own firewall/security group |
| Cloudflare **525 / 526** (SSL handshake / invalid cert) | Origin cert missing/mismatched while mode is Full (strict) | Confirm the `.pem` and `.key` are installed and the paths in nginx match; ensure you used a Cloudflare **Origin Certificate** |
| Redirect loop (`ERR_TOO_MANY_REDIRECTS`) | SSL mode set to Flexible | Set SSL/TLS mode to **Full (strict)** |
| Site loads but no fonts / no 3D | Google Fonts or `/assets/three.min.js` not loading | Non-fatal; verify the asset returns 200 and that no CSP blocks fonts.googleapis.com |
| Old version still showing after update | Edge cache | Purge Cloudflare cache (step 9) |

---

## 11. Optional hardening

- **Self-host the fonts.** Download the Bricolage Grotesque and IBM Plex Mono `woff2` files, drop them in `assets/fonts/`, replace the Google Fonts `<link>` in `index.html` with local `@font-face` rules. Removes the last third-party call and speeds first paint.
- **Cloudflare Tunnel instead of open ports.** Install `cloudflared` and run a named tunnel to `http://localhost:80`. Your origin then needs **no inbound ports open at all** and its IP stays hidden. This replaces steps 6–7's exposure model entirely; the strongest option for your security posture.
- **fail2ban** on the SSH port, and key-only SSH auth if not already.
- **HSTS caution:** the config enables `Strict-Transport-Security` for a year. Only keep it once HTTPS is confirmed working, since browsers will refuse plain HTTP to the domain for that period.
