# Web Attacks and Brute-Force Attacks

## Web attacks (SQL injection, XSS, brute force on web apps)
Web attacks target the application layer of a website or web service rather than the network itself. The CICIDS2017 dataset includes Web Attack - Brute Force, Web Attack - XSS, and Web Attack - SQL Injection.

- SQL Injection (MITRE T1190, Exploit Public-Facing Application) sends crafted input that breaks out of a database query, letting an attacker read, modify, or destroy data, or bypass authentication. On the network it can look like normal HTTP, so detection often relies on payload inspection — which is why a flow-based model with very few SQL-injection examples catches so few of them.
- Cross-Site Scripting (XSS) injects malicious scripts into pages viewed by other users, used to steal sessions or credentials.

## Brute-force attacks (FTP-Patator, SSH-Patator)
A brute-force attack repeatedly guesses credentials against a service until it finds a valid one. CICIDS2017 includes FTP-Patator (brute force against FTP) and SSH-Patator (brute force against SSH). In MITRE ATT&CK this is T1110 (Brute Force).

On the network, brute force shows as many rapid, repeated connection attempts to the same authentication service from one source, with a high rate of failed logins. This is a relatively loud and learnable pattern, which is why models detect patators well.

## Why it matters
Web attacks can lead directly to data breaches and full application compromise. Successful brute force gives an attacker valid credentials — legitimate-looking access that is hard to spot afterward.

## Analyst response / playbook
1. For brute force: identify the targeted account and source; lock or rate-limit the account, block the source, enforce MFA.
2. For web attacks: pull the web server and WAF logs; look for the specific payloads and whether any request succeeded (HTTP 200 with abnormal response size).
3. Confirm whether sensitive data was accessed or any login eventually succeeded.
4. Patch the vulnerable endpoint; add input validation and a WAF rule for the observed payload.
