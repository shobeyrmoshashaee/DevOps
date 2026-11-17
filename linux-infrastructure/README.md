# 🖥️ Linux Infrastructure

A practical collection of Linux infrastructure engineering, administration, high availability, and production operations.

## Scope

- Linux server installation, configuration, and lifecycle management
- Users, groups, permissions, services, and systemd
- Package management, repositories, patching, and maintenance
- Filesystems, LVM, storage mounting, and capacity management
- Networking, DNS, NTP, routing, firewalling, and troubleshooting
- HAProxy, Keepalived, load balancing, and high availability
- Performance analysis, logs, health checks, and reliability
- Hardening, access controls, backup, and recovery procedures

## Projects

| Project | Description |
|---|---|
| [Chrony NTP Server and Client Setup](./chrony-ntp-server-client/) | Redundant internal Chrony servers, secured client access, health validation, and troubleshooting |
| [Custom CA Certificate Trust](./ca-certificate-trust/) | Install, validate, test, rotate, and remove custom CA certificates on Ubuntu and Debian |
| [Certbot SSL Certificate Automation](./certbot-ssl-certificate-automation/) | Manual wildcard DNS validation, Nginx integration, Cloudflare DNS automation, verification, and renewal |

## Project Organization

Each Linux infrastructure project uses its own subdirectory and includes architecture notes, sanitized configuration, deployment instructions, validation, troubleshooting, and operational guidance.
