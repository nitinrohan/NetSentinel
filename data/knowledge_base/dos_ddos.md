# Denial of Service (DoS) and Distributed Denial of Service (DDoS)

## What it is
A Denial of Service attack tries to make a service unavailable to legitimate users by exhausting a resource — bandwidth, connection tables, CPU, or memory. A DoS comes from a single source; a DDoS (Distributed DoS) comes from many sources at once, often a botnet, which makes it far harder to block by IP. In MITRE ATT&CK this is T1498 (Network Denial of Service). Common variants in the CICIDS2017 dataset include DoS Hulk, DoS GoldenEye, DoS Slowloris, DoS Slowhttptest, and DDoS.

## How it looks on the network
DoS/DDoS traffic is the opposite shape of a port scan: loud and heavy. Expect a very high volume of packets or connections to a single destination service, sustained over time. Flood variants (Hulk, GoldenEye, DDoS) show huge packet and byte counts. Slow attacks (Slowloris, Slowhttptest) are sneakier — they hold many connections open with trickle-slow requests to exhaust the server's connection pool, so the byte volume is low but the open-connection count is abnormally high.

## Why it matters
DoS/DDoS directly impacts availability — one of the three pillars of security (confidentiality, integrity, availability). It can also be used as a smokescreen to distract defenders while another intrusion happens elsewhere. DDoS is especially serious because the distributed sources defeat simple IP blocking.

## Analyst response / playbook
1. Identify the targeted service and confirm impact (is it actually degraded or down?).
2. Characterize the attack: single source (DoS) vs many sources (DDoS); flood vs slow.
3. For floods: apply rate limiting, upstream filtering, or engage a DDoS mitigation/CDN provider.
4. For slow attacks: tune server connection timeouts and limits; they are not stopped by bandwidth filtering.
5. Watch for parallel activity — DoS is sometimes a cover for a separate intrusion.
