# Incident Response Exercise: Red Team Attack Chain

A full attack chain I ran against an isolated lab environment, starting with nothing but a name on LinkedIn and ending with a simulated ransomware deployment. Built for a TAFE Cert IV Cyber Security incident response unit, my assessor reviewed both the red and blue playbooks for this exercise and called them excellent.

**Attack flow:** Recon → Phishing → Initial Access → Lateral Movement (employee) → Lateral Movement (file server) → Offline Cracking → Privilege Escalation → Impact

## Objective

Simulate a realistic external attacker compromising a small company network. No access to start, full control of the admin account by the end, showing the whole chain from start to finish (first foothold to impact).

## Tools and environment

- Kali Linux (attacker machine)
- Target: a small multi-host lab network (Linux endpoints and a file server)
- Python script (vibe-coded, credential harvesting page)
- Postfix mail relay (simulated phishing email inside the lab)
- SSH / SCP (lateral movement and exfiltration)
- zip2john, John the Ripper, CUPP (offline password cracking and wordlist generation)
- Simulated LinkedIn profile (OSINT reconnaissance)

## Attack chain

**1. Reconnaissance/OSINT.** No access to anything at the start. I overheard disgruntled IT staff venting about their recent layoff, and about a worker named Roy, a careless IT employee who kept their job despite their lack of care and knowledge for cybersecurity. After checking, a quick OSINT search on public records (without using any specific OSINT tools or dorking) returned a LinkedIn profile belonging to an employee which gave up their full name and company email, enough to build a believable spear-phishing email from.

![Redacted LinkedIn profile showing the target's name, company and email](screenshots/01-osint-linkedin.png)

**2. Credential harvesting.** Vibe coded a Python script to serve a fake company login page and log whatever got typed into it. Hosted it locally and sent a spoofed email from a lookalike domain, made to look like an internal IT notice, with a link to the page.

**3. Initial access.** The employee opened the email and entered their credentials. The script picked them up in real time.

**4. Lateral movement.** Used the harvested login to SSH into the employee's own account.

![SSH login into the employee account using the stolen credentials](screenshots/02-ssh-lateral-1.png)

**5. Data discovery.** Found a personal notes file sitting on the employee's desktop with more internal credentials in it, including access details for the company's shared file server. A common real-world representation of poor password security by someone keeping passwords "all in one place to access easily."

**6. Lateral movement (file server).** Used those creds to SSH into the file server.

**7. Data discovery and exfiltration.** Turned up an old password-protected archive on the server that ended up being a legacy backup of company-wide credentials. Pulled it back to the attacker machine over SCP.

**8. Offline password cracking.** Pulled the hash out with zip2john. Brute-forcing it as-is would've taken forever, so instead I built a wordlist out of what I'd already picked up during the OSINT and notes-file steps, names, dates, the kind of phrases someone reuses, and ran it through John the Ripper. Cracked it in a few minutes. That wordlist gave up a master list of credentials for the whole organisation, admin account included.

![zip2john extracting the hash, John the Ripper cracking it, and the unzipped master password list](screenshots/03-password-cracked.png)

**9. Privilege escalation.** Tried the admin credentials from the cracked list over SSH. Still valid. Guess nobody ever got around to rotating that archive's password.

**10. Data discovery (admin host).** Found a sensitive file on the admin's desktop with confidential company data in it.

**11. Impact.** To demonstrate actual impact rather than just access, I encrypted the sensitive file and folder with a password-protected zip run remotely over SSH, deleted the original, and left a ransom note. Full chain from zero access to data destruction, proven end to end.

![The ransom note left behind on the victim's desktop](screenshots/04-ransom-note.png)

## Findings

The whole chain comes down to a handful of avoidable failures:
- No email filtering to catch the phishing attempt
- Sensitive credentials sitting in plain text in a personal notes file
- A legacy credential backup left on a shared server and never deleted
- Password reuse between a low-privilege account and the admin account
- No SSH key rotation, no MFA

Any one of those fixed on its own would've broken the chain somewhere.

## Reflection

What stuck with me most was how little of this came down to actual exploits. I didn't touch a single piece of vulnerable software. Everything after that first phishing email worked because a person reused a password, kept creds in a text file, or nobody ever got around to rotating an old backup.

If I ran it again I'd try to catch the phishing stage itself rather than only the stuff that happens after someone's already compromised. The blue team side of this exercise (see `blue-team.md`) showed that stage leaves zero trace on the victim's system, which is honestly the part that worries me most looking back at it.
