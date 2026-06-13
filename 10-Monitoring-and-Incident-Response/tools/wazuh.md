# Wazuh

> An open-source SIEM and XDR platform for collecting, analyzing, and alerting on security events.

## Overview

Wazuh centralizes security data from endpoints, applications, cloud workloads, and network devices. It can match incoming events against detection rules, store alerts, and present them through a dashboard.

Its architecture includes:

- **Wazuh agent:** Collects endpoint events.
- **Wazuh server:** Decodes events and applies detection rules.
- **Wazuh indexer:** Stores and searches alerts.
- **Wazuh dashboard:** Supports investigation and visualization.

Wazuh also accepts agentless data from sources such as syslog-enabled network devices.

## Key Features

- Log collection and security event analysis
- Custom detection rules and alerting
- File integrity monitoring
- Vulnerability detection
- Configuration assessment
- Endpoint and cloud workload monitoring
- Dashboard, search, and API access

## Use Cases

- Centralize application and operating-system security logs.
- Detect repeated authentication failures or suspicious privilege use.
- Monitor important files for unauthorized changes.
- Investigate related events across multiple endpoints.
- Build a small security monitoring lab without commercial SIEM licensing.

## Installation

For a lab or small environment, Wazuh provides an all-in-one installation assistant. Check the current [Quickstart documentation](https://documentation.wazuh.com/current/quickstart.html) before installation because supported platforms and package versions change.

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

The quickstart requires a supported 64-bit Linux host. Production deployments should use the architecture and sizing guidance from the official documentation rather than assuming the all-in-one configuration will scale.

## Application Log Integration

Write structured application events to a dedicated file:

```json
{"event_name":"authentication.failed","actor_id":"usr_123","source_ip":"203.0.113.10","outcome":"denied"}
```

Configure a Wazuh agent to collect one-line JSON records:

```xml
<localfile>
  <location>/var/log/example/security-events.json</location>
  <log_format>json</log_format>
</localfile>
```

After collection, create and test rules for the application's stable event fields. Do not send passwords, tokens, cookies, or unnecessary personal data to Wazuh.

## Advantages

- Free and open-source platform
- Combines collection, analysis, indexing, and dashboards
- Supports endpoints and agentless log sources
- Custom rules allow application-specific detections
- Suitable for laboratories and production deployments with appropriate design

## Limitations

- Requires operational work for deployment, upgrades, backups, and rule tuning.
- High event volume needs careful sizing and retention planning.
- Default rules do not understand custom application events automatically.
- A SIEM does not replace incident-response ownership or runbooks.
- Poorly designed alerts can create substantial noise.

## Alternatives

- **Elastic Security:** Flexible search and analytics built on the Elastic Stack.
- **Splunk Enterprise Security:** Commercial SIEM with a large integration ecosystem.
- **Grafana Loki:** Log aggregation that can support alerting, but is not a full SIEM by itself.

## Resources

- [Official website](https://wazuh.com/)
- [Documentation](https://documentation.wazuh.com/current/)
- [Architecture](https://documentation.wazuh.com/current/getting-started/architecture.html)
- [Quickstart](https://documentation.wazuh.com/current/quickstart.html)
- [GitHub organization](https://github.com/wazuh)

## License

Wazuh is free and open source. Its components use the GNU General Public License v2 and Apache License 2.0.

## Tags

`siem` `xdr` `log-analysis` `monitoring` `incident-response` `open-source`

---

**Contributed by:** [Diogo Fernandes](https://github.com/diogux)
**Last Updated:** 2026-06-13
