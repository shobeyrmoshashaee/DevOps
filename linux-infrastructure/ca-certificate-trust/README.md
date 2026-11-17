<p align="center">
  <strong>English</strong> |
  <a href="./README.fa.md">فارسی</a>
</p>

# 🔐 Install a Custom CA Certificate on Ubuntu and Debian

A practical guide for adding an internal or private Certificate Authority certificate to the operating system trust store on Ubuntu and Debian systems.

> [!IMPORTANT]
> All certificate names, domains, and endpoints in this public document are examples. Replace them with approved values from your environment before deployment.

> [!CAUTION]
> Never copy or publish a private key, password, token, PKCS#12 file, or confidential certificate bundle. This procedure requires only the public CA certificate.

## Table of Contents

- [Trust Store Locations](#trust-store-locations)
- [1. Install the Required Package](#1-install-the-required-package)
- [2. Inspect the CA Certificate](#2-inspect-the-ca-certificate)
- [3. Install the CA Certificate](#3-install-the-ca-certificate)
- [4. Validate the Installation](#4-validate-the-installation)
- [5. Test a TLS Endpoint](#5-test-a-tls-endpoint)
- [6. Remove or Replace the CA Certificate](#6-remove-or-replace-the-ca-certificate)
- [Troubleshooting](#troubleshooting)
- [Acceptance Checklist](#acceptance-checklist)

## Trust Store Locations

| Path | Purpose |
|---|---|
| <code>/usr/local/share/ca-certificates/</code> | Source directory for locally managed CA certificates |
| <code>/etc/ssl/certs/</code> | Generated OpenSSL-compatible certificate links |
| <code>/etc/ssl/certs/ca-certificates.crt</code> | Consolidated operating-system CA bundle |

> [!NOTE]
> Locally managed certificates must use the <code>.crt</code> extension. Store one CA certificate per file.

## 1. Install the Required Package

Ensure the operating-system CA management package and OpenSSL are installed:

~~~bash
sudo apt update
sudo apt install -y ca-certificates openssl
~~~

## 2. Inspect the CA Certificate

The certificate should be PEM encoded and begin and end with these markers:

~~~text
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
~~~

Check the first and last lines without displaying the entire certificate:

~~~bash
head -n 1 example-internal-ca.crt
tail -n 1 example-internal-ca.crt
~~~

Inspect the certificate metadata:

~~~bash
openssl x509 \
  -in example-internal-ca.crt \
  -noout \
  -subject \
  -issuer \
  -dates \
  -fingerprint \
  -sha256
~~~

> [!WARNING]
> Confirm the SHA-256 fingerprint through a trusted channel before installing the certificate. Do not trust a CA file only because its filename looks correct.

### Convert a DER Certificate to PEM

If the certificate is DER encoded, convert it before installation:

~~~bash
openssl x509 \
  -inform DER \
  -in example-internal-ca.der \
  -out example-internal-ca.crt
~~~

Validate the converted certificate:

~~~bash
openssl x509 -in example-internal-ca.crt -noout -subject -issuer
~~~

## 3. Install the CA Certificate

Copy the public CA certificate into the local trust-source directory with safe permissions:

~~~bash
sudo install \
  -o root \
  -g root \
  -m 0644 \
  example-internal-ca.crt \
  /usr/local/share/ca-certificates/example-internal-ca.crt
~~~

Update the operating-system trust store:

~~~bash
sudo update-ca-certificates
~~~

A successful update normally reports output similar to:

~~~text
Updating certificates in /etc/ssl/certs...
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.
~~~

> [!NOTE]
> The command rebuilds the system CA bundle and creates the required links under <code>/etc/ssl/certs/</code>. Do not manually copy files into the generated trust directory.

## 4. Validate the Installation

Inspect the installed source certificate:

~~~bash
openssl x509 \
  -in /usr/local/share/ca-certificates/example-internal-ca.crt \
  -noout \
  -subject \
  -issuer \
  -dates \
  -fingerprint \
  -sha256
~~~

Check for the generated certificate link:

~~~bash
ls -l /etc/ssl/certs/ | grep 'example-internal-ca'
~~~

Confirm that the certificate update completes without errors:

~~~bash
sudo update-ca-certificates --verbose
~~~

> [!TIP]
> Compare the SHA-256 fingerprint of the installed file with the fingerprint supplied by the certificate owner or PKI administrator.

## 5. Test a TLS Endpoint

Use a non-sensitive example endpoint in public documentation:

~~~bash
openssl s_client \
  -connect service.example.com:443 \
  -servername service.example.com \
  -CAfile /etc/ssl/certs/ca-certificates.crt \
  </dev/null 2>/dev/null \
  | grep -E 'Verification|Verify return code'
~~~

A trusted and correctly configured TLS chain should report:

~~~text
Verification: OK
Verify return code: 0 (ok)
~~~

Test the same endpoint with cURL:

~~~bash
curl -Iv https://service.example.com/
~~~

> [!WARNING]
> Installing a CA certificate does not fix hostname mismatches, expired certificates, incomplete server chains, or incorrect system time.

### Reload Applications

Long-running applications may cache the trust store. Restart only the affected service after validating the CA installation:

~~~bash
sudo systemctl restart example-service
sudo systemctl --no-pager status example-service
~~~

> [!NOTE]
> Some applications maintain independent trust stores. Java applications, browsers, Python environments using Certifi, containers, and certain runtimes may require separate certificate configuration.

## 6. Remove or Replace the CA Certificate

Remove the locally installed CA certificate:

~~~bash
sudo rm -f /usr/local/share/ca-certificates/example-internal-ca.crt
~~~

Rebuild the trust store and remove stale generated links:

~~~bash
sudo update-ca-certificates --fresh
~~~

Validate that the certificate is no longer present:

~~~bash
ls -l /etc/ssl/certs/ | grep 'example-internal-ca' || true
~~~

> [!CAUTION]
> Removing a trusted CA can immediately break TLS connections for services that depend on it. Identify affected applications and prepare a rollback before removal or certificate rotation.

### Replace an Expiring CA

Back up the existing public certificate:

~~~bash
sudo cp \
  /usr/local/share/ca-certificates/example-internal-ca.crt \
  /usr/local/share/ca-certificates/example-internal-ca.crt.bak
~~~

Install the approved replacement and refresh the trust store:

~~~bash
sudo install \
  -o root \
  -g root \
  -m 0644 \
  replacement-ca.crt \
  /usr/local/share/ca-certificates/example-internal-ca.crt

sudo update-ca-certificates
~~~

Re-run the fingerprint and endpoint validation before deleting the backup.

## Troubleshooting

### Certificate Is Ignored

Verify the file extension and permissions:

~~~bash
find /usr/local/share/ca-certificates \
  -maxdepth 1 \
  -type f \
  -printf '%M %u:%g %f\n'
~~~

The file must use the <code>.crt</code> extension and contain one PEM certificate.

### Certificate Cannot Be Parsed

~~~bash
openssl x509 \
  -in /usr/local/share/ca-certificates/example-internal-ca.crt \
  -noout \
  -text
~~~

If this fails, confirm the input format and obtain a valid public CA certificate from the authorized source.

### Endpoint Is Still Untrusted

Inspect the server-provided chain:

~~~bash
openssl s_client \
  -connect service.example.com:443 \
  -servername service.example.com \
  -showcerts \
  </dev/null
~~~

Common causes include:

- The wrong CA certificate was installed.
- The server does not provide the required intermediate certificate.
- The endpoint hostname does not match the certificate.
- The certificate is expired or not yet valid.
- The application uses a separate trust store.
- The service has not reloaded its cached trust data.

## Acceptance Checklist

- [ ] The certificate is public CA material and contains no private key.
- [ ] The SHA-256 fingerprint was verified through a trusted channel.
- [ ] The certificate is PEM encoded and uses the <code>.crt</code> extension.
- [ ] The file is installed under <code>/usr/local/share/ca-certificates/</code>.
- [ ] <code>update-ca-certificates</code> completes without errors.
- [ ] The generated trust-store link is present under <code>/etc/ssl/certs/</code>.
- [ ] OpenSSL reports <code>Verify return code: 0 (ok)</code> for the target endpoint.
- [ ] The affected application can connect without disabling TLS verification.
- [ ] The rollback and certificate-rotation procedures are understood.

> [!IMPORTANT]
> Never solve a trust failure by disabling TLS verification in production. Install the correct CA chain and fix the underlying certificate or server configuration.
