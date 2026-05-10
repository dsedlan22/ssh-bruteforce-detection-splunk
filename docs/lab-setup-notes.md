# Lab Setup Notes

Supplementary notes documenting setup decisions and trade-offs not covered in the main README.

## VM Resource Allocation

| VM | RAM | CPU | Disk | Purpose |
|---|---|---|---|---|
| Kali-Attacker | 3 GB | 2 cores | 80 GB (sparse) | Pre-built Kali VMware image |
| Target-Ubuntu | 2 GB | 2 cores | 20 GB | Lightweight target |
| Splunk-Server | 4 GB | 2 cores | 40 GB | Splunk requires more RAM |

Total host RAM consumption: approximately 13 GB on a 16 GB system.

## Network Configuration

All three VMs use VMware NAT networking on the same subnet (192.168.254.0/24). NAT provides:
- Internet access for package downloads
- Inter-VM connectivity without host network exposure
- Simpler setup than Bridged or Host-Only modes

Static IPs were configured via Netplan on both Ubuntu hosts to prevent DHCP-induced IP changes between reboots, which would break the Forwarder's hardcoded destination.

## Security Trade-offs in the Lab

Several intentional security weaknesses were introduced to make brute-force attacks viable:

- PAM `pam_pwquality.so` dictionary check disabled, allowing weak passwords
- No `fail2ban` or rate-limiting on SSH (production systems should always have this)
- No SSH key-only authentication
- No 2FA on any user account
- All users use default `/bin/sh` shell (cosmetic only, no security impact)

These configurations must never be applied to production systems.

## Production-Realistic Choices

Despite the deliberate weaknesses above, several production-grade choices were preserved:

- Splunk runs as dedicated `splunk` user, not root
- Universal Forwarder runs as dedicated `splunkfwd` user
- Forwarder permissions granted via `adm` group membership rather than direct file ACLs
- Systemd-managed boot persistence for the Forwarder
- Static IPs via Netplan (not manual `ip` commands)
- Splunk index isolation (`linux_logs` rather than `main`)

## Splunk Sourcetype Considerations

The `linux_secure` sourcetype was used for all auth.log events. While Splunk auto-extracts host, source, sourcetype, pid, and process fields, it does not extract `user` or `src_ip` fields by default. These must be extracted at search time using `rex` or via persistent field extractions in `props.conf` and `transforms.conf`.

For the lab, search-time `rex` was sufficient. In a production deployment, persistent field extractions would be configured to optimize search performance and enable saved-search alerting.

## Wordlist Strategy Rationale

A subset of RockYou (top 500 + 2 manual additions) was chosen over the full 14.3 million entries because:

1. Full RockYou would require approximately 40 days of brute-force at 4 attempts per second, which is impractical for a lab demo
2. The top 500 covers the vast majority of opportunistic attacks against weak passwords
3. Manual additions of `password123` and `admin123` simulate a targeted attacker with prior intelligence
4. The resulting attack volume (1,664 events over 84 minutes) provides sufficient detection signal without excessive runtime

## Tools and Versions

| Tool | Version | Notes |
|---|---|---|
| VMware Workstation | (host) | Virtualization platform |
| Kali Linux | 2025.x | Pre-built VMware image |
| Ubuntu Server | 24.04.4 LTS | Both Target and Splunk hosts |
| Splunk Enterprise | 10.2.3 | Free license (500 MB/day indexing) |
| Splunk Universal Forwarder | 10.2.3 | Matched Enterprise version |
| Metasploit Framework | 6.4.116-dev | Bundled with Kali |
| Nmap | 7.98 | Bundled with Kali |
| OpenSSH | 9.6p1 (Ubuntu 24.04) | Target SSH server |