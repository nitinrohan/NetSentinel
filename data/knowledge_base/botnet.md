# Botnet Activity

## What it is
A botnet is a network of compromised machines ("bots") under the remote control of an attacker (the "bot herder") via command-and-control (C2) infrastructure. Infected hosts periodically beacon out to a C2 server to receive instructions, which can include launching DDoS attacks, sending spam, stealing data, or spreading further. In MITRE ATT&CK, C2 behavior maps to the TA0011 Command and Control tactic. Botnet traffic appears as the "Bot" class in the CICIDS2017 dataset.

## How it looks on the network
The signature of botnet C2 is beaconing: regular, periodic, low-volume connections from an internal host to the same external destination, often at suspiciously even time intervals. Individual connections look small and unremarkable — which is exactly why botnets are hard to detect and why a model with few examples of them tends to miss them. The signal is in the pattern over time, not in any single loud connection.

## Why it matters
A bot on your network means a host is already compromised and under external control. It is both an active threat (it can be told to do damage at any moment) and evidence of an existing breach that needs remediation. Bots are also stepping stones for lateral movement deeper into the network.

## Analyst response / playbook
1. Identify the internal host that is beaconing and the external C2 destination.
2. Check threat intelligence for the C2 domain/IP reputation.
3. Isolate the compromised host from the network to cut off C2.
4. Investigate how it was infected (phishing, drive-by, unpatched service) and hunt for other infected hosts showing the same beacon pattern.
5. Reimage the host rather than just cleaning it — assume full compromise.
