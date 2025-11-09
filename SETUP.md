# Home Lab Setup — Samba AD DC + Win11 RSAT (libvirt/KVM)

This guide reproduces the lab from this repo:

- Host: Ubuntu with libvirt/KVM (virt-manager or Cockpit optional)

- dc01: Ubuntu Server promoted to Samba Active Directory Domain Controller for LAB.LOCAL

- mgmt01: Windows 11 Pro with RSAT (ADUC/DNS/GPMC)

- Optional: join a Linux workstation

- Network: libvirt default NAT (virbr0, 192.168.122.0/24)

No containers for AD: Samba ships its own KDC. Wazuh (if you add it) can be Docker, but the DC and clients are VMs.

## 1) Host prep (Ubuntu)
``` bash
# Install virtualization stack + Cockpit
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients virt-manager \
  bridge-utils dnsmasq-base cloud-image-utils cockpit cockpit-machines

# Enable daemons
sudo systemctl enable --now libvirtd
sudo systemctl enable --now cockpit.socket

# allow your user to manage VMs without sudo
sudo usermod -aG libvirt,kvm $USER

# Create ISO storage
sudo mkdir -p /var/lib/libvirt/images/isos
sudo chown libvirt-qemu:kvm /var/lib/libvirt/images/isos
```
Cockpit (GUI): https://<host>:9090 → Virtual Machines

Default network: virbr0 (NAT 192.168.122.0/24).


## 2) Get media
``` bash

cd /var/lib/libvirt/images/isos
sudo curl -fLO https://releases.ubuntu.com/24.04.3/ubuntu-24.04.3-live-server-amd64.iso
```
Windows 11 ISO
- From a Windows PC: use Microsoft Media Creation Tool or download the Windows 11 ISO from the Microsoft Software Download Page.
- Move it to the host:
  ``` bash
  scp Win11_23H2_English_x64.iso user@<host>:/var/lib/libvirt/images/isos/


## 3) Create VMs
Open https://<host>:9090 -> Virtual Machines -> Create VM

## 3A) dc01 (Ubuntu Server)
- Installation source: /var/lib/libvirt/images/isos/ubuntu-24.04.3-live-server-amd64.iso
- CPU/RAM/Disk: 4vCPU /4GB /30 GB
- Network: default (virbr0)
- Complete Ubuntu install (hostname dc01), boot the VM, then inside dc01:
```
sudo apt update
sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```
## 3B) mgmt01 (Windows RSAT Box)
- Installation source: your WIndows 11 ISO
- CPU/RAM/Disk: 4 vCPU/ 8GB/ 80 - 100 GB
- Network: default (virbr0)
- Complete WIndows Setup ( Pro edition)

## IP addressing (read this before joining)
Libvirts virb0 uses DHCP, so your IPs will not match mine
- Find the DC IP:
```
# from the host
sudo virsh domifaddr dc01 || true

# or inside dc01
ip -4 addr show | awk '/192\.168\.122\./{print $2}'
```
- Easiest approch: keep DHCP and poiunt every clients's DNS to dc01's current IP.
- If you want a stable IP:
    - Reservation (recommended):
    ```
    # get dc01 MAC
  sudo virsh domiflist dc01
  # dump/edit network
  sudo virsh net-dumpxml default > /tmp/default.xml
  # add inside <dhcp>:
  # <host mac='52:54:00:aa:bb:cc' name='dc01' ip='192.168.122.5'/>
  sudo virsh net-define /tmp/default.xml
  sudo virsh net-destroy default
  sudo virsh net-start default
- Static inside dc01 (netplan): set 192.168.122.5/24, gatway 192.168.122.1, keep DNS 127.0.0.1 after AD provision

## 4) Promote dc01 to Samba AD DC
```
sudo apt update
sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent

# Stop services & neutralize any old smb.conf before provisioning
sudo systemctl stop smbd nmbd winbind samba-ad-dc || true
[ -f /etc/samba/smb.conf ] && sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.bak

# Provision the domain
sudo samba-tool domain provision \
  --use-rfc2307 \
  --realm=LAB.LOCAL \
  --domain=LAB \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL \
  --dns-forwarder=1.1.1.1 \
  --adminpass='<StrongPassword>'

# kerberos config from Samba
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

# Make dc01 use itself for DNS
sudo systemctl disable --now systemd-resolved || true
echo -e "nameserver 127.0.0.1\nsearch lab.local" | sudo tee /etc/resolv.conf

# Start AD daemon
sudo systemctl unmask samba-ad-dc || true
sudo systemctl enable --now samba-ad-dc
```
Validate 
```bash
host -t SRV _ldap._tcp.lab.local
host -t SRV _kerberos._tcp.lab.local
host -t A dc01.lab.local

kinit Administrator
klist

sudo samba-tool domain level show
```

## 5) Join mgmt01 to the domain & install RSAT
- Point DNS at the DC:
    - On magmt01 NIC IPv4 settings, set DNS = <dco1 IP>
- Join the domain:
    - Settings -> System -> About -> Rename this PC (advanced) -> Change...
    - Domain: LAB.LOCAL
    - Credentials: Administrator / the password you set
    - Reboot
- Install RSAT (ADUC,DNS,GPMC)
    Open PowerShell (Admin)
  ``` powershell
  $cap = @(
  'Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0',
  'Rsat.Dns.Tools~~~~0.0.1.0',
  'Rsat.GroupPolicy.Management.Tools~~~~0.0.1.0'
  )
  foreach ($c in $cap) { Add-WindowsCapability -Online -Name $c }
  
  # Launchers
  # ADUC: dsa.msc
  # DNS : dnsmgmt.msc
  # GPMC: gpmc.msc
  
## 6) Baseline domain tasks
**OU layout (ADUC)**
```lua
lab.local
 ├─ _Computers
 │   ├─ Servers
 │   └─ Workstations
 └─ _Users
```
Move mgmt01 -> Servers. (dc01 stays under Domain Controllers.)

## 6B) First GPO (GPMC)
gpmc.msc -> Right-click WOrkstations )U -> Create a GPO.. (e.g., Baseline-workstations) -> Link here.

Suggested starts (configure to taste):
- Security -> Audit Policy (Account Logon, Policy Change, System)
- Remote Desktop /WinRM settings under Administrative Templates
- Local Admin control via COmputer Config -> Preferences -> Local Users and Groups
  **Force & Verify on mgmt01:**
  ```powershell
  gpupdate /force
  gpresult /r

## 7) (Optional) Join an Ubuntu workstation
```bash
sudo apt update
sudo apt install -y realmd sssd-ad sssd-tools adcli samba-common-bin oddjob oddjob-mkhomedir packagekit

realm discover LAB.LOCAL
sudo realm join LAB.LOCAL -U Administrator

realm list
```
## 8) Troubleshooting 
- AD daemon masked
  ```sudo systemctl unmask samba-ad-dc && sudo systemctl enable --now samba-ad-dc```
- Provisioning "server role mismatch"
  You still had ```/etc/samba/smb.conf```. Move it aside before provisioning.
- DNS SERVFAIL on dco01
  Ensure dns forwarder = 1.1.1.1 or 8.8.8.8 and /etc/rosolv.conf = nameserver 127.0.0.1
- Client can't join
  Client DNS must point to dc01 (knot public DNS)
- Kerberos checks
  On dc01: kinit Administrator && klist. On Windows: klist, whoami /groups

## 9) Notes on multi-DC & SYSVOL
Samba AD does not implement DFS-R for SYSVOL. if you add more Samba DCs, replicate SYSVOL with rsync then:
```bash
samba-tool ntacl sysvolreset
samba-tool ntacl sysvolcheck
```
(AD database replication is separate: smaba-tool drs showrepl)


## 10 Wazuh ( A or B )
- A Dedicated VM on libcvirt ( recommended this is what I did)
- B Co-Host on dc01 ( works but shares resources with your DC)
Default ports used by Wazuh
- 1514/TCP (agent <-> manager)
- 55000/TCP (agent registration API)
- 443/TCP (dashboard GUI)
## A) Dedicated VM: wazuh01
1) Create the VM
     - Name: wazuh01
     - ISO: UBuntu 24.04 Server  or 22.04 Server ( See note at end)
     - CPU/RAM/Disk: 4 vCPU / 6-8 GB  / 100 GB
     - Network: default
     - Finish install, then inside wazuh01:
       ``` bash
       sudo apt update
        sudo apt install -y qemu-guest-agent docker.io docker-compose-plugin git
        sudo systemctl enable --now qemu-guest-agent docker
2) Deploy Wazuh (single-node)
   ``` bash
   
    git clone https://github.com/wazuh/wazuh-docker.git
    cd wazuh-docker/single-node
    
    # generate certs, then start the stack
    docker compose -f generate-indexer-certs.yml run --rm generator
    docker compose up -d
    
    # sanity
    docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
3) Get IP from Cockpit

4) Open dashboard from a machine in the lab:
     https://<wazuh IP>: (self-signed cert)
   
## 11) Onboard agents 
In Wazuh Dashboard -> Agents -> Deploy new agent ->  fill out the items and select the correct OS. it will generate a CMD to run and tell you what to run to the endpoint. 


### Important note: Ubuntu 24.04  must be ran through Docker method  if you use Package installer from Wazuh it will fail the OS check as it is not supported on Ubuntu 24  


