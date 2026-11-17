# DevOps

Production-ready DevOps projects, cloud-native infrastructure, automation, CI/CD, observability, networking, and storage.

This repository is a structured portfolio of hands-on infrastructure projects. Each project will include sanitized source files, configuration examples, architectural decisions, deployment steps, validation procedures, and operational notes.

> **Repository status:** Initial structure created. Project content will be added progressively.

## Repository Map

| Area | Projects |
|---|---|
| [Kubernetes](./kubernetes/) | RKE2 HA, Cilium & Hubble, Envoy Gateway, Calico, HPA/VPA, Velero migration |
| [CI/CD](./ci-cd/) | GitLab upgrade, GitLab Runner, Argo CD & Kustomize, SonarQube & Trivy |
| [Observability](./observability/) | VictoriaMetrics, Grafana alerting, Zabbix, Elasticsearch snapshots |
| [Storage](./storage/) | Longhorn, Rook Ceph, MinIO object storage |
| [Networking](./networking/) | HAProxy & Keepalived, Envoy Proxy, IPVS for SIP, NGINX |
| [Automation](./automation/) | Ansible infrastructure maintenance, Bash & Python scripts |
| [Platform Services](./platform-services/) | Nexus Repository, FreeIPA SSH policy, RabbitMQ |

## Project Standards

Every project will gradually contain:

- Project overview and production use case
- Architecture and design decisions
- Sanitized configuration and source files
- Installation and deployment instructions
- Validation and troubleshooting commands
- Security and operational considerations
- Lessons learned

## Security

All examples in this repository must be sanitized before publication. Internal IP addresses, domains, credentials, certificates, private keys, kubeconfigs, tokens, and organization-specific data must never be committed.

See [SECURITY.md](./SECURITY.md) for repository security rules.

## Author

**Shobeyr Moshashaee** — Senior DevOps Engineer  
[GitHub](https://github.com/shobeyrmoshashaee) · [LinkedIn](https://www.linkedin.com/in/shobeyr-moshashaee-2583772b2)
