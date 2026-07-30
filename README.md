# Building a Security Onion SIEM

> A practical, sanitized write-up of deploying Security Onion on bare metal and integrating FortiGate, Bitdefender GravityZone, Elastic Fleet, and secure remote access.

## Overview

This project documents the process of standing up a standalone Security Onion SIEM and integrating multiple security data sources into it.

The objective was not simply to install a SIEM, but to build a functional monitoring environment where security telemetry from different sources could be received, parsed, searched, and validated.

The deployment covered:

- Security Onion on dedicated hardware
- FortiGate firewall log ingestion over TCP syslog
- Bitdefender GravityZone Event Push integration
- Elastic Fleet / Elastic Agent integrations
- Security Onion firewall and network configuration
- Secure remote SOC access through a reverse tunnel
- Troubleshooting of routing, listeners, ingestion, services, and Salt

> **Security note:** This repository is intentionally sanitized. Real IP addresses, hostnames, domains, API keys, credentials, tunnel identifiers, organization names, and internal network details have been removed or replaced with placeholders.

---

## Architecture

```text
                         ┌─────────────────────┐
                         │ Bitdefender         │
                         │ GravityZone         │
                         └─────────┬───────────┘
                                   │
                              Event Push
                                HTTPS
                                   │
                                   ▼
┌──────────────┐            ┌────────────────────────┐
│              │ TCP Syslog │                        │
│   FortiGate  ├───────────►│    Security Onion      │
│              │            │                        │
└──────────────┘            │  Fleet / Elastic Agent │
                            │  Elasticsearch         │
                            │  Kibana / SOC          │
                            │  Suricata / Zeek       │
                            └───────────┬────────────┘
                                        │
                                        │ HTTPS
                                        ▼
                              ┌─────────────────────┐
                              │   Reverse Tunnel    │
                              └──────────┬──────────┘
                                         │
                                         ▼
                                   Remote Analyst
```

The important lesson from this architecture was that SIEM integration is not only a logging problem. It is also a networking, firewall, transport, parsing, and validation problem.

A useful way to think about every integration is:

```text
Source Configuration
        ↓
Network Reachability
        ↓
Security Onion Firewall
        ↓
Listening Service
        ↓
Fleet / Elastic Agent
        ↓
Integration / Parser
        ↓
Elasticsearch Data Stream
        ↓
Kibana / SOC
```

---

## 1. Security Onion Deployment

Security Onion was deployed in **Standalone** mode on dedicated physical hardware.

Using bare metal simplified the final network design because the SIEM could participate directly on the required networks without depending on hypervisor NAT or additional virtual-network translation.

### Installation process

1. Prepared the Security Onion installation media.
2. Installed the operating system on the dedicated host.
3. Planned the network-interface roles before completing setup.
4. Assigned a static address to the management interface.
5. Selected the standalone deployment type.
6. Configured administrative access and network settings.
7. Allowed the initial Security Onion configuration and Salt states to complete.
8. Verified that the SOC interface was reachable from an authorized workstation.

A useful first health check was:

```bash
sudo so-status
```

Before integrating any external source, I confirmed that the base Security Onion services were healthy.

---

## 2. Network Planning

Before configuring FortiGate or GravityZone, I mapped the expected communication path for each source.

```text
SOURCE
   ↓
SOURCE NETWORK
   ↓
SECURITY ONION INTERFACE
   ↓
PROTOCOL
   ↓
DESTINATION PORT
   ↓
INTEGRATION
```

This became especially important because the Security Onion host had multiple network interfaces.

A source being able to reach one interface does not necessarily mean it can reach another. The destination address therefore had to correspond to an interface reachable from the source network.

Useful checks included:

```bash
ip addr
```

```bash
ip route
```

For TCP integrations, packet capture was particularly useful:

```bash
sudo tcpdump -i any tcp port <PORT>
```

This provides an important troubleshooting boundary:

- **No packets:** investigate the source, routing, destination address, or upstream firewall.
- **Packets arrive but no events:** investigate the Security Onion firewall, listener, Elastic Agent, integration, or parser.

---

## 3. Security Onion Firewall

One easy-to-miss part of external log ingestion is Security Onion's own firewall.

Opening a port on an upstream firewall does not automatically mean Security Onion will accept the connection.

For an external source, both of these have to be considered:

```text
Network / Perimeter Rules
          +
Security Onion Host Firewall
```

The source host or network must be authorized for the service it needs to reach.

Conceptually:

```text
HOST GROUP
"Who is allowed?"
       +
PORT / SERVICE
"What can they reach?"
```

For the FortiGate integration, the trusted firewall source/network had to be permitted to reach the TCP listener used by the Fortinet integration.

A useful read-only check during troubleshooting was:

```bash
sudo iptables -nvL
```

The important point is to use Security Onion's supported configuration mechanisms for persistent firewall changes rather than treating generated firewall state as the primary configuration source.

---

## 4. FortiGate Log Integration

The FortiGate integration used **TCP syslog**.

The ingestion path was:

```text
FortiGate
    │
    │ TCP Syslog
    ▼
Security Onion
    │
    ▼
Elastic Agent / Fleet
    │
    ▼
Fortinet Integration
    │
    ▼
Elasticsearch
    │
    ▼
Kibana / SOC
```

### 4.1 Add the Fortinet integration

In Fleet, I added the Fortinet/FortiGate integration to the appropriate agent policy and configured its firewall log input as a TCP listener.

A sanitized representation is:

```text
Listen Address: 0.0.0.0
Protocol:       TCP
Port:           <SYSLOG_PORT>
```

The exact port is intentionally omitted. The important requirement is consistency across:

- the Fleet integration;
- Security Onion firewall configuration;
- any network firewall or ACL in the path;
- and the FortiGate syslog destination.

### 4.2 Allow the FortiGate source

The FortiGate address or trusted source network was permitted through the appropriate Security Onion firewall configuration.

The logic was:

```text
FortiGate / Trusted Source
           ↓
Security Onion Host Group
           ↓
Allowed TCP Service
           ↓
Fortinet Listener
```

### 4.3 Confirm the listener

Before repeatedly changing the firewall configuration, I verified that something was actually listening on the configured port:

```bash
sudo ss -tlnp | grep <SYSLOG_PORT>
```

A listener should appear on the expected address/port.

If no listener exists, the problem is further up the ingestion stack and should be investigated in Fleet/Elastic Agent before changing the source device again.

### 4.4 Configure FortiGate

A sanitized FortiOS example looks like:

```text
config log syslogd setting
    set status enable
    set server "<SECURITY_ONION_IP>"
    set port <SYSLOG_PORT>
    set mode reliable
end
```

A useful vendor-specific detail is that FortiGate uses:

```text
set mode reliable
```

for reliable/TCP syslog mode.

The destination address should be the Security Onion interface reachable from the firewall's network path.

### 4.5 Select log categories

Depending on monitoring requirements, useful categories may include:

- traffic and denied traffic;
- authentication activity;
- administrative and system events;
- VPN events;
- IPS;
- antivirus;
- web filtering;
- application control.

For initial onboarding, collecting broadly can make validation easier. The scope can then be tuned based on usefulness, storage, and noise.

---

## 5. Troubleshooting FortiGate Ingestion

I found it more effective to troubleshoot from the bottom of the stack upward.

### Is Security Onion listening?

```bash
sudo ss -tlnp | grep <SYSLOG_PORT>
```

If there is no listener, investigate the Fleet integration or Elastic Agent.

### Are packets reaching Security Onion?

```bash
sudo tcpdump -i any tcp port <SYSLOG_PORT>
```

If packets are absent, investigate:

- routing;
- destination IP/interface;
- ACLs;
- upstream firewall rules;
- source configuration.

### Is TCP being established?

For TCP syslog, the expected handshake is:

```text
SYN → SYN-ACK → ACK
```

Repeated SYN packets without a completed handshake suggest that the host may be reachable while the destination service is unavailable or blocked.

### Are events reaching Elastic?

Packet delivery alone is not enough.

The final validation was performed in Kibana/Discover by checking the relevant Fortinet data stream/dataset and confirming that live events were being parsed.

The integration was only considered successful once the complete path worked:

```text
FortiGate
   ✓
Network
   ✓
Security Onion Firewall
   ✓
TCP Listener
   ✓
Elastic Agent
   ✓
Fortinet Integration
   ✓
Elasticsearch
```

---

## 6. Bitdefender GravityZone Event Push

Bitdefender GravityZone uses a different integration model.

Unlike FortiGate syslog, GravityZone's **Event Push Service** sends events to a configured listener. API access is used to configure and manage that push integration.

```text
GravityZone API Configuration
           ↓
GravityZone Event Push
           ↓
          HTTPS
           ↓
Security Onion / Elastic Integration
           ↓
Elastic Agent
           ↓
Elasticsearch
           ↓
Kibana
```

It is useful to distinguish between:

1. the API credentials used to manage/configure Event Push; and
2. the collector/listener configuration used when events are delivered.

Credentials used for either side should never be committed to a public repository.

---

## 7. GravityZone API Access

An API key with the permissions required for the Event Push Service was created from the GravityZone console.

The general workflow was:

```text
GravityZone Control Center
        ↓
Account / API Configuration
        ↓
Generate API Key
        ↓
Enable Required API Access
```

The key should be treated like a password.

Throughout this repository it is represented as:

```text
<GRAVITYZONE_API_KEY>
```

rather than showing a real value.

---

## 8. Configure the Bitdefender Integration in Fleet

The Bitdefender integration was added to the appropriate Fleet policy on the Security Onion side.

An important troubleshooting lesson was that:

> An integration being installed and healthy in Fleet does not mean the upstream product is already sending events.

Both sides have to be configured.

```text
Elastic Integration Installed  ✓

GravityZone Event Push Set      ?
```

This distinction prevented unnecessary changes to the Elastic side when the missing configuration was actually upstream.

---

## 9. Configure GravityZone Event Push

GravityZone provides API operations for managing Event Push settings.

Conceptually, the configuration defines:

- whether Event Push is enabled;
- the service type;
- the destination URL;
- authorization information;
- TLS/certificate requirements;
- subscribed event types.

A sanitized example:

```json
{
  "jsonrpc": "2.0",
  "method": "setPushEventSettings",
  "params": {
    "status": 1,
    "serviceType": "jsonRPC",
    "serviceSettings": {
      "url": "https://<PUBLIC_EVENT_COLLECTOR>",
      "authorization": "<AUTHORIZATION_VALUE>",
      "requireValidSslCertificate": true
    },
    "subscribeToEventTypes": {
      "<EVENT_TYPE>": true
    }
  },
  "id": "<REQUEST_ID>"
}
```

The actual API key and authorization values are intentionally excluded.

For initial testing, I enabled a broad set of useful event categories so I could first prove the pipeline worked. Event selection could then be refined later.

```text
Collect broadly
      ↓
Confirm ingestion
      ↓
Understand telemetry
      ↓
Reduce unnecessary events
      ↓
Improve signal-to-noise
```

---

## 10. Validate GravityZone Event Push

Validation was performed at multiple layers.

First, the Fleet integration had to be healthy.

Next, GravityZone's current Event Push configuration could be checked using the relevant API operation, such as:

```text
getPushEventSettings
```

A test push can then help confirm delivery.

Finally, I verified that actual Bitdefender events were appearing in Elasticsearch/Kibana under the expected integration data stream.

The complete path was:

```text
GravityZone
    ✓
Event Push
    ✓
HTTPS / Network Path
    ✓
Security Onion
    ✓
Elastic Integration
    ✓
Elasticsearch
```

One particularly useful troubleshooting message indicated that the Event Push settings had not yet been configured.

The key lesson was:

> Security Onion can have a correctly installed integration while GravityZone is still not configured to send anything.

That shifted the troubleshooting workflow from repeatedly changing Elastic settings to verifying the upstream Event Push configuration.

---

## 11. Secure Remote SOC Access

Remote access to the SOC interface was provided through a reverse tunnel rather than directly exposing the Security Onion host through an inbound edge-firewall rule.

```text
Remote Analyst
      │
     HTTPS
      ▼
Public DNS
      │
      ▼
Reverse Tunnel
      │
      ▼
Security Onion Web Interface
```

The public repository intentionally does not contain:

- the real hostname;
- tunnel ID;
- internal destination address;
- credentials;
- certificates;
- organization-specific access rules.

### Troubleshooting lesson

A remote dashboard outage does not automatically mean the tunnel itself has failed.

The troubleshooting path became:

```text
DNS
 ↓
Tunnel
 ↓
Local Connectivity
 ↓
Security Onion Host
 ↓
Web Proxy
 ↓
SOC Service
```

Useful checks included:

```bash
sudo so-status
```

```bash
docker ps
```

```bash
systemctl status <service>
```

This helped avoid troubleshooting the wrong layer.

---

## 12. Salt / Grid Troubleshooting

Security Onion relies on Salt for configuration management and orchestration.

When Salt becomes unhealthy, symptoms can appear elsewhere in the platform:

- configuration changes may not apply;
- highstate may not complete;
- Grid components may report unhealthy;
- services may appear inconsistent.

Useful diagnostic commands included:

```bash
sudo systemctl status salt-master
```

```bash
sudo journalctl -u salt-master
```

I also checked whether expected services and ports were actually available before making further changes.

Any local service overrides or restart-policy changes used during troubleshooting should be treated as environment-specific remediation, not as universal Security Onion deployment requirements. Supported platform configuration should be preferred wherever possible.

---

## 13. Troubleshooting Methodology

The most reusable outcome of this project was a structured approach to SIEM integration troubleshooting.

```text
1. Is the source generating events?
             ↓
2. Is the source configured to forward them?
             ↓
3. Is it targeting the correct IP/interface?
             ↓
4. Is the protocol correct?
             ↓
5. Is the destination port correct?
             ↓
6. Can the source reach that destination?
             ↓
7. Does Security Onion permit the source?
             ↓
8. Is a service actually listening?
             ↓
9. Are packets arriving?
             ↓
10. Is Elastic Agent receiving them?
             ↓
11. Is the correct integration parsing them?
             ↓
12. Is the expected data stream populated?
```

Commands I repeatedly found useful:

```bash
ip addr
ip route
sudo ss -tlnp
sudo tcpdump -i any port <PORT>
sudo iptables -nvL
sudo so-status
docker ps
journalctl -u <service>
```

No individual command proves that ingestion works.

Together, they make it possible to identify where the pipeline stops.

---

## 14. Don't Stop at "Connected"

One of the biggest lessons from this project was that connectivity and ingestion are different things.

```text
LEVEL 1  Source can reach SIEM

LEVEL 2  Source can connect to listener

LEVEL 3  Security Onion receives packets

LEVEL 4  Elastic Agent receives events

LEVEL 5  Integration parses events

LEVEL 6  Correct data stream is populated

LEVEL 7  Fields are usable for detection and hunting
```

I only considered an integration complete once actual events could be searched and the relevant fields were being parsed correctly.

A green status indicator is useful.

Searchable, correctly parsed telemetry is proof.

---

## 15. Security and Sanitization

A SIEM write-up can unintentionally disclose a detailed map of an organization's defensive infrastructure.

Before publishing, I removed or generalized:

- IP addresses and internal subnets;
- public and internal domains;
- usernames and credentials;
- API keys and authorization headers;
- tunnel identifiers;
- organization and site names;
- device names and serial numbers;
- certificates and private keys;
- configuration exports;
- screenshots containing sensitive telemetry.

Examples throughout the repository use placeholders:

```text
<SECURITY_ONION_IP>
<FORTIGATE_IP>
<SYSLOG_PORT>
<GRAVITYZONE_API_KEY>
<API_ACCESS_URL>
<PUBLIC_EVENT_COLLECTOR>
<INTERNAL_NETWORK>
```

The goal is to make the methodology useful without publishing the real environment.

---

## Key Takeaways

**SIEM deployment is an integration exercise.** Installing the platform was only the beginning. The real work was getting independent systems to communicate reliably and produce useful telemetry.

**Troubleshoot from the network upward.** Start with routing, firewall rules, ports, listeners, and packets before changing parsing or visualization settings.

**Security Onion's firewall matters.** A listener existing on the host does not automatically mean an external source is permitted to reach it.

**Understand push versus syslog integrations.** FortiGate and GravityZone have different communication models, which changes where routing and firewall requirements apply.

**An installed integration does not mean data is flowing.** Always verify the source configuration independently.

**Packet capture is invaluable.** `tcpdump` repeatedly answered the most useful early question: *is the traffic actually reaching the SIEM?*

**Verify the final data.** The strongest evidence of a working integration is real, correctly parsed telemetry in the expected Elasticsearch data stream.

---

## What I Learned

What started as a Security Onion installation became a practical exercise in:

- SIEM architecture;
- network design;
- TCP/syslog transport;
- API-based security integrations;
- firewall configuration;
- Elastic Fleet and Elastic Agent;
- Linux service troubleshooting;
- packet-level troubleshooting;
- log validation and security monitoring.

The most important principle I took away from the project is:

> **A SIEM integration is only complete when you can prove the full path from the event source to searchable, correctly parsed data.**

---

## Disclaimer

This repository is provided for educational and portfolio purposes.

The examples are based on practical deployment experience but have been sanitized and generalized. Commands and configuration examples should be reviewed against the documentation for the specific Security Onion, FortiOS, Bitdefender GravityZone, and Elastic versions in use before being applied to another environment.
