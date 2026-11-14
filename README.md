# OPNsense Wazuh Integration

The goal of this project is to ship [OPNsense](https://opnsense.org/) logs into Wazuh over syslog, without installing an agent on the firewall, and turn them into decoded, alertable events instead of noise.

* Syslog listener block for the Wazuh manager (UDP 514, restricted to the firewall IP)
* Firewall (`filterlog`) and `sshd` lines decoded by the stock Wazuh ruleset, no extra work needed
* Custom decoder and rules for web GUI logins (failed, successful, brute force)
* Rule 87702 (repeated blocks from one source) lowered from level 10 to 6, so internet scanners hitting the WAN show on the dashboard without mailing you every few minutes

## Related repositories

| Repo | Why |
|---|---|
| [Wazuh-Antimalware-Stack](https://github.com/stylersnico/Wazuh-Antimalware-Stack) | the Wazuh manager setup these rules plug into. Its **Security detections** dashboard has a *Firewall (OPNsense)* panel built on this repo's rules (block bursts, web GUI and SSH logins by source IP) |
| [Naxsi-Wazuh-Integration](https://github.com/stylersnico/Naxsi-Wazuh-Integration) | same idea for the Naxsi WAF on the reverse proxy behind OPNsense |
| [Custom-Naxsi-Crowdsec-Configs](https://github.com/stylersnico/Custom-Naxsi-Crowdsec-Configs) | the reverse proxy itself (Nginx + Naxsi + CrowdSec) |

None is required: this repo works on any Wazuh manager. Without the stack's dashboard, find the events in *Threat Hunting* with `rule.groups: opnsense or rule.groups: pfsense`.

--------

## Structure

```
manager/
└── syslog_remote.conf      # <remote> syslog listener for ossec.conf
decoders/
└── opnsense_decoders.xml   # web GUI login lines (child of the stock "audit" decoder)
rules/
└── opnsense_rules.xml      # web GUI login rules + level override of 87702
```

--------

## What you get

| Log | Decoded by | Rule | Level | Mail (default level 10+) |
|---|---|---|---|---|
| `filterlog` packet passed | stock `pf` decoder | 87700 | 0 | no |
| `filterlog` packet blocked | stock `pf` decoder | 87701 | 5, not stored (`no_log`) | no |
| 18 blocks in 45 s from one IP | stock `pf` decoder | 87702 (overridden) | 6 | no |
| `sshd` failed login | stock `sshd` decoder | 5760, 5710... | 5 | no |
| `sshd` brute force | stock `sshd` decoder | 5712, 5763... | 10 | yes |
| Web GUI login failed | `opnsense-audit-login` | 100601 | 5 | no |
| Web GUI login successful | `opnsense-audit-login` | 100602 | 3 | no |
| 5 failed web GUI logins in 5 min from one IP | `opnsense-audit-login` | 100603 | 10 | yes |

Everything else OPNsense sends (`configd.py`, `dhcpd`, `unbound`...) is received and dropped unless a stock Wazuh rule matches it. Logs are not archived (`logall` stays off), so a busy firewall costs almost nothing on the indexer.

--------

## Installation

### 1. Manager side

Paste the contents of `manager/syslog_remote.conf` at the end of `/var/ossec/etc/ossec.conf` and replace `192.168.1.1` with the IP OPNsense sends from (its LAN address, usually).

Copy the decoder and the rules:
```bash
cp decoders/opnsense_decoders.xml /var/ossec/etc/decoders/
cp rules/opnsense_rules.xml /var/ossec/etc/rules/
chown wazuh:wazuh /var/ossec/etc/decoders/opnsense_decoders.xml /var/ossec/etc/rules/opnsense_rules.xml
chmod 660 /var/ossec/etc/decoders/opnsense_decoders.xml /var/ossec/etc/rules/opnsense_rules.xml
```

> :warning: **Rule IDs 100600-100603 are custom-range placeholders.** If you already have other local rules in that range, renumber before deploying - `wazuh-analysisd` will refuse to start on a collision.

Check the configuration, then restart the manager:
```bash
/var/ossec/bin/wazuh-analysisd -t
```
```bash
systemctl restart wazuh-manager
```

Open UDP 514 from OPNsense to the manager if a firewall sits in between. The manager log should show:
```
wazuh-remoted: INFO: Remote syslog allowed from: '192.168.1.1'
wazuh-remoted: INFO: Started (pid: 1234). Listening on port 514/UDP (syslog).
```

### 2. OPNsense side

*System → Settings → Logging → Remote*, click **+**:

| Field | Value |
|---|---|
| Enabled | checked |
| Transport | UDP(4) |
| Applications | empty (= all) |
| Levels | empty (= all) |
| Facilities | empty (= all) |
| Hostname | IP of the Wazuh manager |
| Port | 514 |
| RFC5424 | **unchecked** (Wazuh expects classic BSD syslog) |
| Description | Wazuh |

**Save**, then **Apply**.

Optional: to see packets dropped by the default deny rule, check *Firewall → Settings → Advanced → Log packets matched from the default block rules*. For your own rules, tick **Log packets that are handled by this rule** in each rule.

--------

## Validating before you restart in production

Check the packets arrive on the manager:
```bash
tcpdump -n -i any -c 10 udp port 514
```

`wazuh-logtest` lets you replay a log line against the decoder and rules without touching the running service:
```bash
/var/ossec/bin/wazuh-logtest
```

Paste these lines:
```
Nov 14 10:00:00 OPNsense.localdomain audit[1234]: /index.php: Web GUI authentication error for 'root' from 192.168.1.50
Nov 14 10:00:01 OPNsense.localdomain audit[1234]: /index.php: Successful login for user 'root' from: 192.168.1.50 (using Local Database)
Nov 14 10:00:02 OPNsense.localdomain filterlog[5678]: 77,,,fae559338f65e11c53669fc3642c93c2,igb0,match,block,in,4,0x0,,242,54321,0,none,6,tcp,40,203.0.113.30,198.51.100.5,51234,22,0,S,1234567,,1024,,
```

Expected result: rule `100601` (level 5), then `100602` (level 3), then `87701` with `action: block` and `srcip: 203.0.113.30`.

Then test for real: a wrong password on the OPNsense web GUI must show up in *Threat Hunting* with rule `100601`.

--------

## Known gotchas

* **Wazuh takes one protocol per syslog listener.** `<protocol>udp,tcp</protocol>` is accepted by the config check but only TCP ends up listening. Stay on UDP, or declare a second `<remote>` block on another port for TCP.
* **RFC5424 must stay off on OPNsense.** With it, the program name is not parsed and none of the decoders match.
* **The web GUI decoder is a child of the stock `solaris_bsm` decoder.** Wazuh's Solaris BSM decoder claims every message whose program is `audit`, which is what OPNsense uses for its web GUI log. A standalone decoder never gets a chance, so this one is attached to `solaris_bsm` and rules start from its parent rule `6100`.
* **IPv6 `filterlog` lines are not fully decoded.** The stock `pf` decoder assumes the IPv4 column layout: action is right, IP and port fields may be shifted on IPv6.
* **Events carry the manager as agent.** Syslog events show `agent.name` = the manager; the firewall is in `location` (its IP) and `predecoder.hostname`.
* **WireGuard does not log connections.** It has no sessions, only silent handshakes, so nothing reaches syslog. To see VPN connections, tick *Log packets that are handled by this rule* on the WAN rule allowing the WireGuard port: one `filterlog` line per new connection, with the source IP (not the peer name).
