# 🔒 Certbot SSL Certificate Issuance and Automation

A practical Ubuntu and Debian guide for installing Certbot, issuing wildcard certificates with DNS verification, integrating certificates with Nginx, and automating DNS-01 validation through Cloudflare.

> [!IMPORTANT]
> All domains, DNS records, certificate names, paths, and credentials in this public document are examples. Replace them with approved values from your environment.

> [!CAUTION]
> Never publish Cloudflare API tokens, private keys, Certbot credential files, or the contents of <code>/etc/letsencrypt/</code>. Only placeholder values are used in this guide.

## Table of Contents

- [Methods at a Glance](#methods-at-a-glance)
- [Prerequisites](#prerequisites)
- [Install Certbot and the Nginx Plugin](#install-certbot-and-the-nginx-plugin)
- [Step 1 — Wildcard Certificate with Manual DNS Verification](#step-1--wildcard-certificate-with-manual-dns-verification)
- [Step 2 — Nginx Certificate with Automatic Installation](#step-2--nginx-certificate-with-automatic-installation)
- [Step 3 — Automated DNS Verification with Cloudflare](#step-3--automated-dns-verification-with-cloudflare)
- [Certificate Verification](#certificate-verification)
- [Renewal and Operations](#renewal-and-operations)
- [Troubleshooting](#troubleshooting)
- [Acceptance Checklist](#acceptance-checklist)

## Methods at a Glance

| Method | Validation | Wildcard | Nginx Configuration | Automatic Renewal |
|---|---|---:|---|---:|
| Manual Certbot | DNS-01 with a manually created TXT record | Yes | Not modified | No, unless DNS hooks are added |
| Certbot Nginx plugin | HTTP-01 over public port 80 | No | Automatically updated | Yes |
| Cloudflare DNS plugin | Automated DNS-01 through the Cloudflare API | Yes | Manual or separate installation | Yes |

> [!NOTE]
> A DNS record pointing a hostname to the Nginx server is not the same as a DNS-01 challenge. The Nginx plugin normally validates through HTTP-01 on TCP port 80. DNS-01 is required for wildcard certificates.

## Prerequisites

- A public domain under your control
- Correct system time
- Root or sudo access
- Public DNS access for the requested names
- Nginx installed for Step 2
- Cloudflare authoritative DNS and a restricted API token for Step 3

Verify the system time:

~~~bash
timedatectl status
~~~

## Install Certbot and the Nginx Plugin

Install Certbot and the Nginx integration package:

~~~bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
~~~

Verify the installation:

~~~bash
certbot --version
sudo certbot plugins
~~~

The plugin list should include <code>nginx</code>.

> [!WARNING]
> The <code>--register-unsafely-without-email</code> option disables expiry and account-recovery notifications. It is used here because the workflow explicitly avoids email registration, but monitored production environments should have an alternative renewal-alerting mechanism.

## Step 1 — Wildcard Certificate with Manual DNS Verification

This method obtains one certificate for both <code>example.com</code> and <code>*.example.com</code>. Wildcard issuance requires DNS-01 validation.

Run Certbot without email registration:

~~~bash
sudo certbot certonly \
  --manual \
  --preferred-challenges dns \
  --register-unsafely-without-email \
  --agree-tos \
  -d "example.com" \
  -d "*.example.com"
~~~

Certbot displays a DNS TXT record similar to:

~~~text
Name:  _acme-challenge.example.com
Type:  TXT
Value: <ACME_VALIDATION_VALUE>
~~~

Create the TXT record in the authoritative DNS zone. Before pressing Enter in Certbot, confirm that the value is publicly visible:

~~~bash
dig +short TXT _acme-challenge.example.com @1.1.1.1
dig +short TXT _acme-challenge.example.com @8.8.8.8
~~~

Continue only after the expected TXT value appears from public resolvers.

> [!WARNING]
> Certbot may request more than one TXT value when the root and wildcard names are validated together. Keep every active value until all challenges finish.

After successful issuance, the certificate is stored under:

~~~text
/etc/letsencrypt/live/example.com/fullchain.pem
/etc/letsencrypt/live/example.com/privkey.pem
/etc/letsencrypt/live/example.com/cert.pem
/etc/letsencrypt/live/example.com/chain.pem
~~~

Inspect the certificate:

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
> A certificate created with the interactive manual plugin cannot renew automatically unless authentication hooks are provided. Without hooks, repeat the same command and create new TXT values during renewal.

## Step 2 — Nginx Certificate with Automatic Installation

The Nginx plugin validates the domain with HTTP-01 and then edits the matching Nginx server block to enable HTTPS.

### Prepare Public DNS

Create an A or AAAA record that points the example hostname to the public Nginx endpoint:

~~~text
app.example.com  A  192.0.2.20
~~~

The address above is from the RFC 5737 documentation range and must be replaced.

Confirm public resolution:

~~~bash
dig +short app.example.com A
dig +short app.example.com AAAA
~~~

### Prepare the Nginx Server Block

Example HTTP configuration:

~~~nginx
server {
    listen 80;
    listen [::]:80;

    server_name app.example.com;

    root /var/www/app;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
~~~

Validate and reload Nginx:

~~~bash
sudo nginx -t
sudo systemctl reload nginx
~~~

Ensure TCP port 80 is reachable from the Internet before continuing.

### Issue and Install the Certificate

~~~bash
sudo certbot --nginx \
  --register-unsafely-without-email \
  --agree-tos \
  --redirect \
  -d "app.example.com"
~~~

This command:

- Completes HTTP-01 validation through Nginx
- Obtains the certificate
- Updates the matching server block
- Adds an HTTP-to-HTTPS redirect
- Creates a renewable Certbot configuration

Validate the result:

~~~bash
sudo nginx -t
sudo systemctl --no-pager status nginx
sudo certbot certificates
curl -I https://app.example.com/
~~~

> [!NOTE]
> The Nginx authenticator does not issue wildcard certificates because it uses HTTP-01. Use manual DNS-01 or an automated DNS provider plugin for wildcard names.

## Step 3 — Automated DNS Verification with Cloudflare

The Cloudflare plugin creates and removes the required <code>_acme-challenge</code> TXT records through the Cloudflare API. It is useful for wildcard certificates, servers without public port 80, and unattended renewal.

### Install the Cloudflare Plugin

~~~bash
sudo apt update
sudo apt install -y python3-certbot-dns-cloudflare
~~~

Confirm that Certbot detects the plugin:

~~~bash
sudo certbot plugins | grep -A3 dns-cloudflare
~~~

### Create a Restricted API Token

Create a Cloudflare API token restricted to:

~~~text
Permission: Zone / DNS / Edit
Resource:   Include / Specific zone / example.com
~~~

> [!CAUTION]
> Do not use the Cloudflare Global API Key. Use a restricted API token scoped only to the required DNS zone.

Create a protected credentials directory:

~~~bash
sudo install -d -o root -g root -m 0700 /root/.secrets/certbot
~~~

Create the credentials file:

~~~bash
sudo nano /root/.secrets/certbot/cloudflare.ini
~~~

Add only the placeholder-formatted token entry:

~~~ini
dns_cloudflare_api_token = <CLOUDFLARE_API_TOKEN>
~~~

Restrict access:

~~~bash
sudo chown root:root /root/.secrets/certbot/cloudflare.ini
sudo chmod 600 /root/.secrets/certbot/cloudflare.ini
~~~

Verify the permissions without displaying the secret:

~~~bash
sudo stat -c '%A %U:%G %n' /root/.secrets/certbot/cloudflare.ini
~~~

Expected permissions:

~~~text
-rw------- root:root /root/.secrets/certbot/cloudflare.ini
~~~

### Issue a Wildcard Certificate Automatically

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

The propagation option gives public DNS resolvers time to observe the temporary TXT records before ACME validation begins.

> [!NOTE]
> Certbot stores the credentials-file path in the renewal configuration, not the token value. The protected INI file must remain available for future renewals.

### Optional Nginx Configuration

Point Nginx to the issued certificate:

~~~nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    root /var/www/example;
    index index.html;
}
~~~

Validate and reload:

~~~bash
sudo nginx -t
sudo systemctl reload nginx
~~~

## Certificate Verification

List all Certbot-managed certificates:

~~~bash
sudo certbot certificates
~~~

Inspect the certificate identity and validity:

~~~bash
sudo openssl x509 \
  -in /etc/letsencrypt/live/example.com/fullchain.pem \
  -noout \
  -subject \
  -issuer \
  -serial \
  -dates \
  -fingerprint \
  -sha256 \
  -ext subjectAltName
~~~

Verify the live TLS endpoint:

~~~bash
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

## Renewal and Operations

Test renewal without replacing the production certificate:

~~~bash
sudo certbot renew --dry-run
~~~

Inspect the Certbot timer:

~~~bash
sudo systemctl status certbot.timer --no-pager
sudo systemctl list-timers --all | grep certbot
~~~

Review renewal configuration files without printing credentials:

~~~bash
sudo ls -l /etc/letsencrypt/renewal/
~~~

> [!IMPORTANT]
> The Nginx and Cloudflare plugin workflows support unattended renewal. The interactive manual DNS workflow does not, unless custom authentication and cleanup hooks automate the TXT records.

### Reload Nginx After Renewal

Most packaged Certbot installations handle the required deployment workflow. A deploy hook can be tested explicitly:

~~~bash
sudo certbot renew \
  --dry-run \
  --deploy-hook "systemctl reload nginx"
~~~

## Troubleshooting

### DNS TXT Record Is Not Visible

~~~bash
dig +trace _acme-challenge.example.com TXT
dig +short TXT _acme-challenge.example.com @1.1.1.1
~~~

Wait for propagation and verify that the TXT record was created in the authoritative DNS zone.

### Nginx Validation Fails

~~~bash
sudo nginx -t
sudo ss -lntp | grep ':80'
curl -I http://app.example.com/
sudo journalctl -u nginx -n 100 --no-pager
~~~

Confirm that public DNS points to the correct endpoint and that inbound TCP/80 is not blocked.

### Cloudflare Reports Invalid Credentials

~~~bash
sudo stat -c '%A %U:%G %n' /root/.secrets/certbot/cloudflare.ini
sudo certbot plugins
~~~

Check the token scope, selected zone, credentials-file syntax, and installed plugin version. Never print the token into logs or terminal history.

### Certificate Was Issued but Nginx Still Uses the Old One

~~~bash
sudo nginx -T | grep -E 'ssl_certificate|ssl_certificate_key'
sudo nginx -t
sudo systemctl reload nginx
~~~

Confirm that Nginx references the correct <code>fullchain.pem</code> and <code>privkey.pem</code> paths.

## Acceptance Checklist

- [ ] Certbot and the Nginx plugin are installed.
- [ ] Manual wildcard issuance uses DNS-01.
- [ ] Public TXT values are verified before continuing manual validation.
- [ ] The Nginx workflow uses a valid public DNS record and HTTP-01 on port 80.
- [ ] No email address is registered with the ACME account.
- [ ] The Cloudflare token is restricted to DNS editing for one zone.
- [ ] The Cloudflare credentials file is owned by root with mode <code>0600</code>.
- [ ] No API token, private key, or real internal identifier is committed.
- [ ] Certificate SANs and expiration dates are verified.
- [ ] The live endpoint reports <code>Verify return code: 0 (ok)</code>.
- [ ] <code>certbot renew --dry-run</code> succeeds for automated workflows.
- [ ] Renewal failures are monitored through an external alerting mechanism.

## References

- [Certbot User Guide](https://eff-certbot.readthedocs.io/en/stable/using.html)
- [Certbot DNS Cloudflare Plugin](https://certbot-dns-cloudflare.readthedocs.io/en/stable/)
- [Certbot Nginx Instructions](https://certbot.eff.org/instructions)
