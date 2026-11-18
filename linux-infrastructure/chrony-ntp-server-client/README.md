<p align="center">
  <strong>English</strong> |
  <a href="./README.fa.md">فارسی</a>
</p>

# 🕒 Enterprise Chrony NTP Server and Client Setup

A production-oriented guide for deploying two redundant internal Chrony servers and configuring Linux clients to use them as centralized time sources.

> [!IMPORTANT]
> All addresses in this public document use the RFC 5737 documentation range. Replace them with the correct values for your environment before deployment.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [1. Configure the Chrony Servers](#1-configure-the-chrony-servers)
- [2. Configure the Chrony Clients](#2-configure-the-chrony-clients)
- [3. Understanding the Health Output](#3-understanding-the-health-output)
- [4. Basic Troubleshooting](#4-basic-troubleshooting)
- [5. Final Verification](#5-final-verification)
- [Acceptance Checklist](#acceptance-checklist)
- [References](#references)

## Architecture Overview

~~~mermaid
flowchart TD
    U["Public NTP Sources"]
    P["Primary Chrony Server<br/>192.0.2.10"]
    S["Secondary Chrony Server<br/>192.0.2.11"]
    C["Linux Clients<br/>192.0.2.0/24"]

    U --> P
    U --> S
    P --> C
    S --> C
~~~

| Layer | Address or Source | Role |
|---|---|---|
| Upstream | Public NTP sources | Provide reference time to the internal Chrony servers |
| Primary NTP server | <code>192.0.2.10</code> | Preferred internal time source for Linux clients |
| Secondary NTP server | <code>192.0.2.11</code> | Redundant internal time source for Linux clients |
| Client network | <code>192.0.2.0/24</code> | Linux systems synchronized from the internal servers |

> [!NOTE]
> The example permits NTP clients from <code>192.0.2.0/24</code>. Replace this documentation subnet with the authorized client network in the target environment.

## 1. Configure the Chrony Servers

Perform the following steps on both Chrony servers.

### Install Chrony

~~~bash
sudo apt update
sudo apt install -y chrony
~~~

Back up the current configuration before making changes:

~~~bash
sudo cp /etc/chrony/chrony.conf /etc/chrony/chrony.conf.bak
sudo nano /etc/chrony/chrony.conf
~~~

Comment out or remove the existing default <code>server</code> and <code>pool</code> entries, then add:

~~~conf
# External NTP sources
server 217.218.127.128 iburst
server ntp.time.ir iburst
pool 2.ir.pool.ntp.org iburst maxsources 4
pool 0.asia.pool.ntp.org iburst maxsources 4

# Allow NTP clients only from the authorized internal network
allow 192.0.2.0/24

# Correct large offsets during the first three updates
makestep 1.0 3

# Synchronize the hardware clock
rtcsync
~~~

### Configuration Notes

| Directive | Description |
|---|---|
| <code>iburst</code> | Sends an initial burst of requests so synchronization can be established faster |
| <code>maxsources 4</code> | Limits the sources selected from a pool; it applies to <code>pool</code> entries, not fixed <code>server</code> entries |
| <code>allow 192.0.2.0/24</code> | Permits only the specified client subnet to query this server over NTP |
| <code>makestep 1.0 3</code> | Steps the clock when the offset is greater than one second during the first three updates |
| <code>rtcsync</code> | Periodically synchronizes the hardware real-time clock from the system clock |

> [!WARNING]
> Never leave the <code>allow</code> directive unrestricted. Permit only the network ranges that are authorized to consume the NTP service.

### Allow NTP Through the Firewall

NTP uses UDP port <code>123</code>. If UFW is enabled, permit only the authorized client network:

~~~bash
sudo ufw allow from 192.0.2.0/24 to any port 123 proto udp
~~~

> [!TIP]
> If another host firewall is used, create an equivalent inbound rule for UDP/123 and restrict its source to the approved client networks.

### Start the Service

~~~bash
sudo systemctl enable chrony
sudo systemctl restart chrony
sudo systemctl --no-pager status chrony
~~~

### Validate the Servers

~~~bash
chronyc sources -v
chronyc tracking
chronyc sourcestats -v
chronyc selectdata
chronyc clients
~~~

> [!NOTE]
> <code>chronyc clients</code> begins listing systems only after the clients have sent NTP requests to the server.

## 2. Configure the Chrony Clients

Perform the following steps on every Linux client.

### Install Chrony

~~~bash
sudo apt update
sudo apt install -y chrony
~~~

Back up the current configuration:

~~~bash
sudo cp /etc/chrony/chrony.conf /etc/chrony/chrony.conf.bak
sudo nano /etc/chrony/chrony.conf
~~~

Comment out or remove existing external <code>server</code> and <code>pool</code> entries, then configure the internal Chrony servers:

~~~conf
# Internal Chrony servers
server 192.0.2.10 iburst prefer
server 192.0.2.11 iburst

# Correct large offsets during the first three updates
makestep 1.0 3

# Synchronize the hardware clock
rtcsync
~~~

> [!NOTE]
> The primary server is preferred while it remains healthy. Chrony can select the secondary server when the preferred source is unavailable or unsuitable.

> [!IMPORTANT]
> Clients do not require an <code>allow</code> directive because they consume time and are not intended to serve NTP to other systems.

### Start the Service

~~~bash
sudo systemctl enable chrony
sudo systemctl restart chrony
sudo systemctl --no-pager status chrony
~~~

### Validate the Clients

~~~bash
chronyc sources -v
chronyc tracking
chronyc sourcestats -v
chronyc selectdata
timedatectl status
~~~

A healthy source selection can look like:

~~~text
^* 192.0.2.10
^+ 192.0.2.11
~~~

The selected source can change according to reachability and time-source quality.

## 3. Understanding the Health Output

### <code>chronyc sources -v</code>

| Symbol or Field | Meaning |
|---|---|
| <code>^*</code> | Source currently selected for synchronization |
| <code>^+</code> | Healthy acceptable source that may be combined with the selected source |
| <code>^-</code> | Valid source that is not currently selected |
| <code>^?</code> | Unreachable source or a source with insufficient measurements |
| <code>Reach</code> | Reachability register; it should normally increase toward <code>377</code> after successful polls |

### <code>chronyc tracking</code>

A healthy system should normally show:

- <code>Leap status : Normal</code>
- A valid <code>Reference ID</code>
- A non-zero <code>Stratum</code>
- Small system-time and last-offset values

### <code>timedatectl status</code>

Verify that the system reports:

~~~text
System clock synchronized: yes
NTP service: active
~~~

## 4. Basic Troubleshooting

### Sources Remain Unreachable

If sources remain marked with <code>^?</code>, verify DNS resolution, routing, firewall policy, and UDP port <code>123</code>:

~~~bash
getent hosts ntp.time.ir
getent hosts 2.ir.pool.ntp.org
sudo ss -ulpn | grep ':123'
~~~

Review the Chrony service and its recent logs:

~~~bash
sudo systemctl status chrony
sudo journalctl -u chrony -n 100 --no-pager
~~~

> [!CAUTION]
> Do not disable the firewall as a troubleshooting shortcut. Test UDP/123 explicitly and keep the source-network restriction in place.

### Restore the Previous Configuration

If the new configuration causes a problem, restore the backup and restart Chrony:

~~~bash
sudo cp /etc/chrony/chrony.conf.bak /etc/chrony/chrony.conf
sudo systemctl restart chrony
~~~

## 5. Final Verification

### On the Chrony Servers

~~~bash
chronyc sources -v
chronyc tracking
chronyc clients
~~~

### On the Chrony Clients

~~~bash
chronyc sources -v
chronyc tracking
timedatectl status
~~~

## Acceptance Checklist

- [ ] Both Chrony servers have valid upstream NTP sources.
- [ ] The Chrony service is active on both servers and all clients.
- [ ] Clients can reach the internal servers over UDP port <code>123</code>.
- [ ] One source is marked with <code>^*</code> on each client.
- [ ] The secondary source remains available when the primary is unavailable or unsuitable.
- [ ] <code>chronyc tracking</code> reports <code>Leap status : Normal</code>.
- [ ] <code>timedatectl status</code> reports that the system clock is synchronized.
- [ ] <code>chronyc clients</code> lists the expected clients on the servers.

> [!TIP]
> The deployment is healthy when both internal servers have valid upstream sources, clients can reach UDP/123, one source is selected with <code>^*</code>, and tracking reports a normal leap status.

## References

- [Chrony Configuration Reference](https://chrony-project.org/doc/latest/chrony.conf.html)
- [Chrony Documentation](https://chrony-project.org/documentation.html)
