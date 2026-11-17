# 🔒 Certbot SSL Certificate Automation for HAProxy

A practical Ubuntu and Debian guide for issuing wildcard certificates with DNS verification, preparing HAProxy-compatible PEM bundles, automating DNS-01 validation through Cloudflare, and reloading HAProxy safely after renewal.

> [!IMPORTANT]
> All domains, addresses, certificate names, paths, and credentials in this public document are examples. Replace them with approved values from your environment.

> [!CAUTION]
> Never publish Cloudflare API tokens, private keys, Certbot credential files, HAProxy PEM bundles, or the contents of <code>/etc/letsencrypt/</code>.

## Table of Contents

- [Nginx and HAProxy Difference](#nginx-and-haproxy-difference)
- [Methods at a Glance](#methods-at-a-glance)
- [Install Certbot and HAProxy](#install-certbot-and-haproxy)
- [Step 1 — Wildcard Certificate with Manual DNS Verification](#step-1--wildcard-certificate-with-manual-dns-verification)
- [Step 2 — Install the Certificate in HAProxy](#step-2--install-the-certificate-in-haproxy)
- [Step 3 — Automated DNS Verification with Cloudflare](#step-3--automated-dns-verification-with-cloudflare)
- [Automated HAProxy Deployment After Renewal](#automated-haproxy-deployment-after-renewal)
- [Certificate Verification](#certificate-verification)
- [Troubleshooting](#troubleshooting)
- [Acceptance Checklist](#acceptance-checklist)

## Nginx and HAProxy Difference

The official Certbot Nginx plugin can validate a domain and edit Nginx automatically. Certbot does not include a first-party built-in HAProxy installer, so this guide obtains certificates with <code>certonly</code>, creates a PEM bundle containing the private key and certificate chain, validates the HAProxy configuration, and then reloads HAProxy.

> [!NOTE]
> A third-party HAProxy plugin exists, but the standard Certbot plus deploy-hook workflow is used here to keep certificate handling explicit and portable.

## Methods at a Glance

| Method | Validation | Wildcard | HAProxy Installation | Automatic Renewal |
|---|---|---:|---|---:|
| Manual Certbot | DNS-01 with manually created TXT records | Yes | Manual PEM bundle | No, unless DNS hooks are added |
| Cloudflare DNS plugin | Automated DNS-01 through the Cloudflare API | Yes | Deploy hook creates the PEM bundle | Yes |
| Standalone Certbot | HTTP-01 over public port 80 | No | Manual PEM bundle | Possible, but may conflict with HAProxy port 80 |

> [!TIP]
> DNS-01 is preferred for HAProxy because it supports wildcard certificates and does not require stopping HAProxy or exposing a temporary standalone listener on port 80.

## Install Certbot and HAProxy

~~~bash
sudo apt update
sudo apt install -y certbot haproxy
~~~

Verify both installations:

~~~bash
certbot --version
haproxy -v
sudo systemctl --no-pager status haproxy
~~~

> [!WARNING]
> The <code>--register-unsafely-without-email</code> option disables expiry and account-recovery notifications. Use external renewal monitoring when no email is registered.

## Step 1 — Wildcard Certificate with Manual DNS Verification

Issue a certificate for the root domain and all first-level subdomains:

~~~bash
sudo certbot certonly \
  --manual \
  --preferred-challenges dns \
  --register-unsafely-without-email \
  --agree-tos \
  -d "example.com" \
  -d "*.example.com"
~~~

Certbot requests one or more DNS TXT records:

~~~text
Name:  _acme-challenge.example.com
Type:  TXT
Value: <ACME_VALIDATION_VALUE>
~~~

Create every requested TXT value and verify public propagation before pressing Enter:

~~~bash
dig +short TXT _acme-challenge.example.com @1.1.1.1
dig +short TXT _acme-challenge.example.com @8.8.8.8
~~~

> [!WARNING]
> Keep all TXT values active until Certbot completes every challenge. Removing the first value too early can make combined root-and-wildcard validation fail.

Inspect the issued certificate:

~~~bash
sudo certbot certificates

sudo openssl x509 \
  -in /etc/letsencrypt/live/example.com/fullchain.pem \
  -noout \
  -subject \
  -issuer \
  -dates \
  -ext subjectAltName
~~~

> [!IMPORTANT]
> Interactive manual DNS certificates do not support unattended renewal without authentication hooks. Cloudflare automation in Step 3 is the preferred production workflow.

## Step 2 — Install the Certificate in HAProxy

HAProxy commonly loads the private key, leaf certificate, and intermediate chain from one PEM bundle.

### Create the Certificate Directory

~~~bash
sudo install \
  -d \
  -o root \
  -g haproxy \
  -m 0750 \
  /etc/haproxy/certs
~~~

### Build the HAProxy PEM Bundle

Create the bundle in the order expected by HAProxy: private key followed by the full certificate chain.

~~~bash
sudo sh -c '
  umask 027
  cat \
    /etc/letsencrypt/live/example.com/privkey.pem \
    /etc/letsencrypt/live/example.com/fullchain.pem \
    > /etc/haproxy/certs/example.com.pem
'
~~~

Set restrictive ownership and permissions:

~~~bash
sudo chown root:haproxy /etc/haproxy/certs/example.com.pem
sudo chmod 0640 /etc/haproxy/certs/example.com.pem
~~~

> [!CAUTION]
> The HAProxy PEM bundle contains the private key. Never commit it to Git, copy it into public documentation, or expose it to untrusted users.

### Configure HAProxy

~~~haproxy
frontend public_http
    bind :80
    mode http
    http-request redirect scheme https code 301

frontend public_https
    bind :443 ssl crt /etc/haproxy/certs/example.com.pem alpn h2,http/1.1
    mode http
    option forwardfor
    default_backend application_backend

backend application_backend
    mode http
    balance roundrobin
    server app01 192.0.2.31:8080 check
    server app02 192.0.2.32:8080 check
~~~

All addresses above are from the RFC 5737 documentation range.

Validate the configuration before reload:

~~~bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
sudo systemctl --no-pager status haproxy
~~~

Verify the listening ports:

~~~bash
sudo ss -lntp | grep -E ':(80|443)\b'
~~~

## Step 3 — Automated DNS Verification with Cloudflare

The Cloudflare plugin creates and removes DNS-01 TXT records automatically. It supports wildcard certificates and unattended renewal without interrupting HAProxy traffic.

### Install the Plugin

~~~bash
sudo apt update
sudo apt install -y python3-certbot-dns-cloudflare
sudo certbot plugins | grep -A3 dns-cloudflare
~~~

### Create a Restricted API Token

Use only a zone-scoped token:

~~~text
Permission: Zone / DNS / Edit
Resource:   Include / Specific zone / example.com
~~~

> [!CAUTION]
> Do not use the Cloudflare Global API Key. Restrict the API token to the minimum required DNS zone and permission.

Create and protect the credentials file:

~~~bash
sudo install -d -o root -g root -m 0700 /root/.secrets/certbot
sudo nano /root/.secrets/certbot/cloudflare.ini
~~~

~~~ini
dns_cloudflare_api_token = <CLOUDFLARE_API_TOKEN>
~~~

~~~bash
sudo chown root:root /root/.secrets/certbot/cloudflare.ini
sudo chmod 0600 /root/.secrets/certbot/cloudflare.ini
sudo stat -c '%A %U:%G %n' /root/.secrets/certbot/cloudflare.ini
~~~

Expected permissions:

~~~text
-rw------- root:root /root/.secrets/certbot/cloudflare.ini
~~~

### Issue the Wildcard Certificate

~~~bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/certbot/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 60 \
  --register-unsafely-without-email \
  --agree-tos \
  -d "example.com" \
  -d "*.example.com"
~~~

> [!NOTE]
> Certbot stores the credentials-file path in its renewal configuration, not the token itself. The protected credentials file must remain available for future renewals.

## Automated HAProxy Deployment After Renewal

A deploy hook rebuilds the HAProxy PEM bundle only after successful issuance or renewal.

### Create the Deploy Hook

~~~bash
sudo nano /usr/local/sbin/deploy-certbot-haproxy
~~~

Add:

~~~bash
#!/usr/bin/env bash
set -Eeuo pipefail

: "${RENEWED_LINEAGE:?RENEWED_LINEAGE is required}"

cert_name="$(basename "$RENEWED_LINEAGE")"
cert_dir="/etc/haproxy/certs"
target="${cert_dir}/${cert_name}.pem"
backup="${target}.previous"
temporary="$(mktemp "${cert_dir}/.${cert_name}.XXXXXX")"

cleanup() {
  rm -f "$temporary"
}
trap cleanup EXIT

install -d -o root -g haproxy -m 0750 "$cert_dir"

cat \
  "${RENEWED_LINEAGE}/privkey.pem" \
  "${RENEWED_LINEAGE}/fullchain.pem" \
  > "$temporary"

chown root:haproxy "$temporary"
chmod 0640 "$temporary"

openssl pkey -in "$temporary" -noout
openssl x509 -in "$temporary" -noout -subject -issuer -dates

if [[ -f "$target" ]]; then
  cp -a "$target" "$backup"
fi

install -o root -g haproxy -m 0640 "$temporary" "$target"

if ! haproxy -c -f /etc/haproxy/haproxy.cfg; then
  if [[ -f "$backup" ]]; then
    mv -f "$backup" "$target"
  else
    rm -f "$target"
  fi
  echo "HAProxy configuration validation failed; previous certificate restored." >&2
  exit 1
fi

systemctl reload haproxy
rm -f "$backup"

echo "HAProxy certificate deployed: $target"
~~~

Secure the script:

~~~bash
sudo chown root:root /usr/local/sbin/deploy-certbot-haproxy
sudo chmod 0750 /usr/local/sbin/deploy-certbot-haproxy
~~~

### Test the Hook

~~~bash
sudo env \
  RENEWED_LINEAGE=/etc/letsencrypt/live/example.com \
  /usr/local/sbin/deploy-certbot-haproxy
~~~

Test automatic renewal:

~~~bash
sudo certbot renew \
  --dry-run \
  --deploy-hook /usr/local/sbin/deploy-certbot-haproxy
~~~

> [!IMPORTANT]
> HAProxy configuration validation occurs before reload. If validation fails, the previous PEM file is restored.

### Persist the Deploy Hook

~~~bash
sudo install \
  -d \
  -o root \
  -g root \
  -m 0755 \
  /etc/letsencrypt/renewal-hooks/deploy

sudo ln -sfn \
  /usr/local/sbin/deploy-certbot-haproxy \
  /etc/letsencrypt/renewal-hooks/deploy/20-haproxy
~~~

Check the Certbot timer:

~~~bash
sudo systemctl status certbot.timer --no-pager
sudo systemctl list-timers --all | grep certbot
~~~

## Certificate Verification

Verify the PEM file:

~~~bash
sudo stat -c '%A %U:%G %n' /etc/haproxy/certs/example.com.pem
sudo openssl pkey -in /etc/haproxy/certs/example.com.pem -noout
sudo openssl x509 -in /etc/haproxy/certs/example.com.pem -noout -subject -issuer -dates
~~~

Validate HAProxy and inspect the live endpoint:

~~~bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg

openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  -showcerts \
  </dev/null 2>/dev/null \
  | grep -E 'Verification|Verify return code'
~~~

A healthy endpoint should report:

~~~text
Verification: OK
Verify return code: 0 (ok)
~~~

Inspect the live certificate:

~~~bash
echo | openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  2>/dev/null \
  | openssl x509 \
      -noout \
      -subject \
      -issuer \
      -dates \
      -fingerprint \
      -sha256 \
      -ext subjectAltName
~~~

## Troubleshooting

### HAProxy Cannot Load the PEM File

~~~bash
sudo openssl pkey -in /etc/haproxy/certs/example.com.pem -noout
sudo openssl x509 -in /etc/haproxy/certs/example.com.pem -noout -subject -issuer
~~~

Confirm that the bundle contains the private key followed by the full certificate chain.

### HAProxy Configuration Validation Fails

~~~bash
sudo haproxy -c -V -f /etc/haproxy/haproxy.cfg
sudo journalctl -u haproxy -n 100 --no-pager
~~~

Do not reload or restart HAProxy until validation succeeds.

### Cloudflare Validation Fails

~~~bash
sudo stat -c '%A %U:%G %n' /root/.secrets/certbot/cloudflare.ini
sudo certbot plugins
dig +trace _acme-challenge.example.com TXT
~~~

Verify the token scope, selected DNS zone, credentials syntax, and propagation time. Never print the token value.

### HAProxy Still Serves the Old Certificate

~~~bash
sudo stat /etc/haproxy/certs/example.com.pem
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
~~~

Compare the live certificate fingerprint with the renewed certificate fingerprint.

## Acceptance Checklist

- [ ] Certbot and HAProxy are installed.
- [ ] Wildcard issuance uses DNS-01.
- [ ] Manual TXT values are publicly visible before validation continues.
- [ ] No email address is registered with the ACME account.
- [ ] HAProxy receives a PEM bundle containing the private key and full chain.
- [ ] The PEM bundle is owned by <code>root:haproxy</code> with mode <code>0640</code>.
- [ ] HAProxy configuration validation succeeds before every reload.
- [ ] The Cloudflare token is restricted to DNS editing for one zone.
- [ ] The Cloudflare credentials file is owned by root with mode <code>0600</code>.
- [ ] No API token, private key, certificate bundle, or real internal identifier is committed.
- [ ] The live endpoint reports <code>Verify return code: 0 (ok)</code>.
- [ ] <code>certbot renew --dry-run</code> succeeds with the HAProxy deploy hook.
- [ ] Renewal failures are monitored externally.

## References

- [Certbot User Guide](https://eff-certbot.readthedocs.io/en/stable/using.html)
- [Certbot DNS Cloudflare Plugin](https://certbot-dns-cloudflare.readthedocs.io/en/stable/)
- [Certbot Instructions for HAProxy](https://certbot.eff.org/instructions)
- [HAProxy Configuration Manual](https://docs.haproxy.org/)
