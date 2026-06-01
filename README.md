

## What is this?
I built a small Security Operations Center (SOC) lab at home. It’s my first real hands‑on project in cybersecurity. I wanted to understand how SIEM works, how to detect a brute‑force attack, and what it feels like to respond to an incident.

## Why did I do this?
I’m aiming for a junior SOC analyst role. I believe the best way to learn is to break things and then fix them. Reading is important, but configuring everything myself taught me much more.

## Tools & environment

 **VirtualBox**  Running three VMs (Kali, CentOS, POP_OS) 
 **Wazuh** (Docker)  The SIEM – collects logs, triggers alerts 
 **Kali Linux**  Attacker machine (nmap, hydra for SSH brute force) 
 **CentOS 7**  Victim machine with Wazuh agent 
 **GitHub**  Where I store my configs and document my journey 

## What I did step by step

1. **Set up the virtual network** – each VM has two adapters: NAT (for internet) and Host‑Only (so they can talk to each other). I used `vboxnet0` with IPs: Kali `192.168.56.102`, CentOS `192.168.56.103`.

2. **Installed Wazuh on POP_OS** – used the official docker‑compose. After a few tries (and fixing a port conflict) the manager, indexer and dashboard were all running.

3. **Installed Wazuh agent on CentOS** – at first it failed because I typed `upd` instead of `udp` in the protocol setting. Reading `/var/ossec/logs/ossec.log` saved me.

4. **Registered the agent** – I had a conflict with the default name `localhost.localdomain`. After deleting the old agent in the manager and cleaning `client.keys` on CentOS, the agent finally connected.

5. **Launched an attack** – from Kali I used `hydra` to brute‑force SSH on CentOS. The logs `/var/log/secure` on CentOS immediately filled with `Failed password` entries.

6. **Saw alerts in Wazuh** – in the dashboard I found rule **5758** ("Maximum authentication attempts exceeded", level 8). I was super excited – it really worked!

7. **Tried to enable active response** – I configured `firewall-drop` in the manager. Unfortunately I couldn’t make it work. I suspect that inside the Docker container (running on VirtualBox) iptables doesn’t have enough privileges. I also tested `host-deny` – it didn’t block the IP either. But I understood the concept and I know how it should work in a normal environment.

## What worked well 

- Full Wazuh stack (manager, indexer, dashboard) running correctly.
- CentOS agent connected to the manager (`Connected to server` in logs).
- Attack logs are generated and visible on the victim.
- Alerts in the dashboard – I can see the attacker’s IP, MITRE T1110, and the description.
- I learned to read logs, debug network issues, and navigate the Wazuh interface.

## What didn’t work and why 

- **Active response (automatic IP blocking)** – despite correct configuration, the `firewall-drop` script never executed on CentOS. After hours of debugging I think it’s a permission issue with iptables inside the Docker container (running on a VirtualBox VM). In a real, non‑containerised environment it would probably work.
- **Journald problems** – Wazuh agent on CentOS kept throwing `Failed to get message from the journal`. I switched the log source to the plain file `/var/log/secure` and everything started working.
- **Initial connection refused** – the host firewall (`ufw`) was blocking port 1514/tcp. After opening it the agent connected.

## What I learned

- Installing and configuring Wazuh (agent, manager, dashboard).
- Reading logs like a detective – where to look for errors.
- VirtualBox networking – NAT vs Host‑Only vs NAT Network.
- The importance of small details – one typo (`upd` instead of `udp`) cost me an hour.
- How to document my work – I now write down everything I try, even the failures.

## Next steps

- Make active response work in a non‑Docker environment.
- Add Suricata (IDS) and forward its logs to Wazuh.
- Use Metasploitable as a more vulnerable target.


## Screenshots

nmap_scan.png – port scanning

hydra_attack.png – brute force attack from Kali.

agent_wazuh_CentOS.png – agent online.

attack_detection.png – alert in dashboard.

ossec_rule_configuration.png (poprawiona nazwa) – firewall-drop configuration for rule 5758.

iptables_check.png – empty iptables.

ossec_error.png –  agent error.




## How to reproduce (short version)

1. Clone this repo.
2. Create three VMs in VirtualBox with Host‑Only + NAT.
3. Install Docker and run the Wazuh single‑node stack.
4. Install Wazuh agent on CentOS, point it to the host’s Host‑Only IP (192.168.56.1).
5. From Kali run `hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://<CentOS-IP>`.
6. Watch alerts appear in the Wazuh dashboard.

Detailed config files are in the `configs/` folder.

## Final thoughts

This project is not perfect – the automatic blocking part is still missing. I now understand the full pipeline: attack → logs → alert → (attempted) response.

## Contact

e02.ignys@gmail.com
