# Port Scanning (Reconnaissance)

## What it is
A port scan is a reconnaissance technique where an attacker sends probes to many ports on a target host to discover which services are running and reachable. It is usually the first stage of an attack — the attacker is mapping the attack surface before choosing an exploit. Common tools include nmap and masscan. In the MITRE ATT&CK framework this maps to T1046 (Network Service Discovery).

## How it looks on the network
Port scans produce a very recognizable traffic shape: a single source address contacts a large number of distinct destination ports in a short window, with many connections that are never completed. In flow features this shows up as a high count of short-lived connections, a high rate of failed or reset connections (SYN sent, no full handshake), and many distinct destination ports per source. Unlike a DoS flood, the total data volume is small — scans are quiet and wide, not loud and heavy.

## Why it matters
A scan on its own does not breach anything, but it is an early-warning signal. Detecting and investigating scans lets a SOC get ahead of an attack before exploitation. Scans against sensitive or internal hosts are higher priority than scans hitting an internet-facing host that is probed constantly.

## Analyst response / playbook
1. Confirm the source IP and whether it is internal or external.
2. Identify which ports/services were probed and whether any are sensitive (RDP, SSH, SMB, databases).
3. Check whether the scan was followed by a connection to an open service (possible exploitation attempt).
4. If internal source: treat as possible lateral movement or a compromised host — escalate.
5. If external and noisy/broad: likely opportunistic internet background noise — rate-limit or block at the firewall, lower priority.
