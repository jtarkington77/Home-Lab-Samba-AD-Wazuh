# Home Lab: Samba AD DC + Windows RSAT + Wazuh (single-host, KVM/libvirt)

**Stack (final state)**  
- Host: Ubuntu + libvirt/KVM (Cockpit VM plugin), NAT `virbr0` (192.168.122.0/24)
- Core DC: `dc01` (Ubuntu Server, Samba AD DC) — **IP `192.168.122.5`**
- Admin box: `mgmt01` (Windows 11 Pro, RSAT)  
- Workstations: `ws02` (Windows 11), `ws01` (Ubuntu Desktop)  
- SIEM: **Wazuh single-node (manager + indexer + dashboard v4.7.5)** via Docker
- Agents: `mgmt01-agent`, `WIN01-Agent`, `DC01-Agent`, `WS02-Agent` (reinstalled clean)

---

## What this repo contains
- **README:** high-level architecture + summary (this file)  
- **/docs (added next):** exact commands & screenshots for AD + RSAT + Wazuh  
- **/screenshots (added next):** evidence for agents, vulns, and GPO checks

> Secrets are redacted (`<REDACTED>`). Replace lab credentials before sharing screenshots/logs.

---

## Build summary (condensed)
1. **Samba AD on `dc01`**
   - Provisioned with `samba-tool domain provision --realm LAB.LOCAL --domain LAB --dns-backend=SAMBA_INTERNAL`
   - `/etc/krb5.conf` copied from Samba; resolver set to `127.0.0.1`
   - `dns forwarder = 1.1.1.1` in `/etc/samba/smb.conf`
   - Verified: `_ldap._tcp`, `_kerberos._tcp`, `kinit Administrator`, `klist`

2. **RSAT on `mgmt01`**
   - Joined to `LAB.LOCAL` (NIC DNS → `dc01`)
   - RSAT installed: `Get-WindowsCapability -Name RSAT* -Online | Add-WindowsCapability -Online`
   - Users: `jeremy` (Domain Admins + Helpdesk Admins), `lab.svc.wazuh`, `lab.svc.edr` (Domain Users)
   - First GPO linked to a `Workstations` OU (logon banner test)

3. **Wazuh single-node**
   - `git clone https://github.com/wazuh/wazuh-docker.git && cd wazuh-docker/single-node`
   - `docker compose -f generate-indexer-certs.yml run --rm generator && docker compose up -d`
   - Verified Vulnerability Detector in `ossec.log` (“Analyzing… Finished”)
   - Fixed WS02 by purging old agent via API (`?purge=true&force=true`), reinstalling

---

## Next in this repo
- Add `/docs/guide.md` with exact commands (AD, RSAT, Wazuh, agent purge)  
- Add `/screenshots` and link them here
