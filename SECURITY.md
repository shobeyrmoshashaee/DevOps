# Security Policy

This is a public DevOps portfolio repository. Never commit production secrets or organization-specific sensitive data.

## Prohibited Content

- Passwords, API keys, access tokens, and private credentials
- TLS private keys, SSH private keys, and signing keys
- Kubeconfig files and service-account tokens
- Internal domains, hostnames, IP addresses, and registry endpoints
- Customer, employee, or organization-confidential information
- Unredacted logs, backups, database dumps, or environment files

## Sanitization Rules

Use documentation-only examples such as:

- `example.com` for domains
- `192.0.2.0/24`, `198.51.100.0/24`, and `203.0.113.0/24` for public documentation IPs
- `registry.example.com` for private registries
- `.env.example` for required environment variables

Every file must be reviewed and sanitized before it is pushed.
