# Linux-Network-Troubleshooting-Security-Hardening-Lab
A hands-on lab that simulates real network faults on Linux, diagnoses them with a fixed layered method, traces them through system logs, and then hardens the servers against attack.

Skills demonstrated: Linux networking, fault isolation, log investigation, firewall configuration, SSH hardening, open-port auditing.

Table of Contents
Lab Topology
Tools Used
Lab Setup
Phase 1: Networking and Connectivity
Phase 2: Troubleshooting Scenarios
Phase 3: Log Investigation
Phase 4: Security Hardening
Results
Repository Structure
Key Learnings
Lab Topology

Show Image

VM	Role	OS	Subnet A (192.168.10.0/24)	Subnet B (192.168.20.0/24)
VM1	Gateway / Firewall	Ubuntu Server	192.168.10.1	192.168.20.1
VM2	App Server (Nginx, SSH, dnsmasq)	Ubuntu Server		192.168.20.10
VM3	Client / "Attacker"	Ubuntu / Kali	192.168.10.20	

Update the IPs above if you used different addresses.

Tools Used

iproute2 (ip), ss, ping, traceroute, tcpdump, dig, nslookup, resolvectl, journalctl, nftables / UFW, dnsmasq, Nginx, OpenSSH, Fail2ban, nmap, Bash, netplan

Lab Setup
Hypervisor: VirtualBox / VMware / Multipass (choose one)
Networks: two host-only networks, with VM1 attached to both
Configs: all working configuration files are stored in configs/
Phase 1: Networking and Connectivity
Task	Command / File	Verification
Static IPs	netplan (/etc/netplan/*.yaml)	ip a, ip route
IP forwarding on VM1	sysctl -w net.ipv4.ip_forward=1	Client can reach App Server across subnets
DNS server	dnsmasq on VM2	dig @192.168.20.10 app.lab, nslookup
Listening ports	ss -tulnp	Baseline list saved in evidence/
VLAN interface	ip link add link eth0 name eth0.10 type vlan id 10	ip -d link show eth0.10
Bonding (optional)	netplan bond config	cat /proc/net/bonding/bond0
Phase 2: Troubleshooting Scenarios

Method used for every fault:

Interface -> IP -> Gateway -> DNS -> Firewall -> Application

Each scenario has its own write-up in scenarios/ with symptom, diagnosis steps, root cause, fix, and proof.

#	Fault Injected	Symptom	Key Tools	Write-up
1	Wrong default gateway	Local ping works, remote fails	ip route, traceroute	01-wrong-gateway.md
2	DNS server removed	Ping by IP works, by name fails	dig, resolvectl status	02-dns-failure.md
3	Firewall blocks port 80	Connection timeout	nft list ruleset, tcpdump	03-firewall-block.md
4	Nginx stopped	Connection refused	systemctl status, ss -tlnp	04-service-down.md
5	Wrong netmask	Partial connectivity	ip a, ping	05-wrong-netmask.md

Timeout vs. refusal (captured with tcpdump -i eth0 port 80):

Timeout: SYN packets are sent with no reply, which usually means a firewall is dropping them.
Refused: the server answers with RST, meaning the host is reachable but nothing is listening.

Packet captures are stored in evidence/.

Phase 3: Log Investigation
Reviewed service logs: journalctl -u ssh, journalctl -u nginx, journalctl -p err
Reviewed /var/log/auth.log and /var/log/syslog
Generated failed SSH logins and an nmap scan from VM3, then located them in the logs
Added a firewall LOG rule for dropped packets and traced them in journalctl -k
Wrote a Bash script that counts failed SSH logins per source IP: scripts/failed_logins.sh

Example script output:

Failed SSH logins by IP
   47  192.168.10.20
    3  192.168.10.35
Phase 4: Security Hardening
Area	Change
SSH	Key-only login, PermitRootLogin no, PasswordAuthentication no
Firewall	Default deny incoming; allow only SSH, HTTP, DNS as needed
Fail2ban	Automatically bans the attacker VM after repeated failed logins
Audit	Open ports checked before and after with ss and nmap
Results
Metric	Before	After
Open TCP/UDP ports on App Server	X	Y
SSH password logins	Enabled	Disabled
Brute-force attempts blocked	0	Banned by Fail2ban after N attempts

Replace X, Y, and N with your real measured values. Evidence is in evidence/.

Repository Structure
linux-network-troubleshooting-lab/
├── README.md
├── topology.png
├── configs/        # netplan, nftables, sshd_config, fail2ban, dnsmasq
├── scenarios/      # one .md per fault: symptom, diagnosis, fix
├── scripts/        # failed-login counter, port audit
└── evidence/       # screenshots, tcpdump captures, log snippets
Key Learnings
A fixed, layer-by-layer method finds faults faster than guessing.
A timeout and a refusal point to completely different causes (firewall vs. service).
Logs confirm what tools suggest; use both.
Hardening should be measured (before/after audits), not just applied.
How to Reproduce
Create the three VMs and two host-only networks as shown in the topology.
Apply the configs from configs/.
Follow the scenario files in scenarios/ to break and fix each fault.
Run scripts/failed_logins.sh after generating failed logins from VM3.
Author

Your Name | LinkedIn | GitHub
