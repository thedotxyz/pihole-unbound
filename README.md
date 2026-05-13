# Pi-hole + Unbound Recursive DNS

[![Pi-hole](https://img.shields.io/badge/Pi--hole-DNS%20sinkhole-96060C?style=for-the-badge&logo=pi-hole&logoColor=white)](https://pi-hole.net)
[![Unbound](https://img.shields.io/badge/Unbound-Recursive%20DNS-005A9C?style=for-the-badge)](https://nlnetlabs.nl/projects/unbound/)
[![Proxmox LXC](https://img.shields.io/badge/Proxmox-LXC-E57000?style=for-the-badge&logo=proxmox&logoColor=white)](https://proxmox.com)
[![Debian](https://img.shields.io/badge/Debian-12%20%7C%2013-A81D33?style=for-the-badge&logo=debian&logoColor=white)](https://www.debian.org)
[![Raspberry Pi](https://img.shields.io/badge/Raspberry%20Pi-supported-A22846?style=for-the-badge&logo=raspberrypi&logoColor=white)](https://www.raspberrypi.com)

Lightweight setup for running **Pi-hole** with **Unbound** as a local recursive DNS resolver.
 
This guide supports:

- Proxmox VE with an unprivileged Debian LXC container
- Debian 12 Bookworm
- Debian 13 Trixie
- Raspberry Pi OS / Debian-based systems

Part of the [pixelkeep homelab](https://github.com/pixelkeep/homelab-network) setup.

---

## What this setup does

Pi-hole provides DNS filtering, blocklists, and the web interface.

Unbound provides local recursive DNS resolution and DNSSEC validation.

The target architecture is:

```
LAN clients
   |
   | DNS :53
   v
Pi-hole
   |
   | upstream DNS: 127.0.0.1#5335
   v
Unbound
   |
   | recursive DNS queries
   v
Root / TLD / authoritative DNS servers
```

## Why Pi-hole + Unbound instead of Pi-hole alone

Pi-hole alone requires an upstream DNS provider — typically a public resolver like Cloudflare (`1.1.1.1`) or Google (`8.8.8.8`). This means every DNS query that is not blocked by Pi-hole is forwarded to a third-party server. That third party can log, analyse, and profile all domains your network resolves.

Unbound eliminates that dependency. Instead of forwarding queries to a public resolver, Unbound resolves DNS recursively — it queries the root nameservers directly and follows the chain to the authoritative nameserver for each domain. No query leaves your network to a third party.

Combined benefits:

- **Privacy** — no third-party DNS provider sees your query history
- **DNSSEC** — Unbound validates DNSSEC signatures end-to-end, preventing DNS spoofing
- **Ad blocking** — Pi-hole handles filtering before Unbound resolves anything
- **No dependency** — works without internet access to a public resolver, as long as root servers are reachable
- **Low latency** — Unbound caches resolved records locally; repeat queries resolve instantly

The only trade-off: Unbound's first resolution for an uncached domain takes slightly longer than forwarding to a cached public resolver. For a homelab this is imperceptible.

---

## Key references

- Pi-hole documentation: <https://docs.pi-hole.net/>
- Pi-hole Unbound guide: <https://docs.pi-hole.net/guides/dns/unbound/>
- DNSSEC overview: <https://www.icann.org/resources/pages/dnssec-what-is-it-why-important-2019-03-05-en>
- Unbound documentation: <https://unbound.docs.nlnetlabs.nl/>
- Homelab network design: <https://github.com/pixelkeep/homelab-network>
- Raspberry Pi OS setup: <https://github.com/pixelkeep/rpi-setup>

---

## Contents

- [What this setup does](#what-this-setup-does)
- [Why Pi-hole + Unbound instead of Pi-hole alone](#why-pi-hole--unbound-instead-of-pi-hole-alone)
- [Key references](#key-references)
- [Prerequisites](#prerequisites)
- [Proxmox LXC preparation](#proxmox-lxc-preparation)
  * [Select the target storage](#select-the-target-storage)
  * [Configure system locale](#configure-system-locale)
  * [Create an administrative user](#create-an-administrative-user)
  * [Enable SSH access](#enable-ssh-access)
  * [Configure a static IP address](#configure-a-static-ip-address)
  * [Validate basic connectivity](#validate-basic-connectivity)
  * [Continue installation over SSH](#continue-installation-over-ssh)
- [Debian or Raspberry Pi preparation](#debian-or-raspberry-pi-preparation)
- [Pre-flight checks](#pre-flight-checks)
- [Install Pi-hole](#install-pi-hole)
- [Install and configure Unbound](#install-and-configure-unbound)
- [Debian Bullseye / Bookworm / Trixie resolvconf fix](#debian-bullseye--bookworm--trixie-resolvconf-fix)
- [Validate and restart Unbound](#validate-and-restart-unbound)
- [Point Pi-hole to Unbound](#point-pi-hole-to-unbound)
- [Verify Pi-hole is using Unbound](#verify-pi-hole-is-using-unbound)
- [Quick validation](#quick-validation)
- [Security hardening](#security-hardening)
  * [Network exposure baseline](#network-exposure-baseline)
  * [Administrative user](#administrative-user)
  * [SSH hardening](#ssh-hardening)
  * [Firewall guidance](#firewall-guidance)
  * [Optional local firewall with UFW](#optional-local-firewall-with-ufw)
  * [Proxmox LXC-specific hardening](#proxmox-lxc-specific-hardening)
  * [Proxmox firewall recommendation](#proxmox-firewall-recommendation)
  * [Raspberry Pi-specific hardening](#raspberry-pi-specific-hardening)
  * [Pi-hole NTP behavior](#pi-hole-ntp-behavior)
  * [Fail2ban](#fail2ban)
- [Automatic Debian security updates](#automatic-debian-security-updates)
- [Pi-hole maintenance and updates](#pi-hole-maintenance-and-updates)
- [Pi-hole update procedure](#pi-hole-update-procedure)
- [Backup recommendation](#backup-recommendation)
- [Security validation checklist](#security-validation-checklist)
- [Sources](#sources)

---

## Prerequisites

This guide supports the following deployment models:

```
Proxmox VE    Debian 13 or Debian 12 unprivileged LXC
Raspberry Pi  Raspberry Pi OS based on Debian
Debian        Debian 13 Trixie or Debian 12 Bookworm
```

For Proxmox VE, the recommended container profile is:

```
Type:        Unprivileged LXC
OS:          Debian 13 standard template
CPU:         1 vCPU
RAM:         512 MB minimum, 1 GB recommended
Disk:        4 GB minimum, 8 GB recommended
Network:     Bridged network with static IP or DHCP reservation
VLAN tag:    20  (Servers VLAN — adjust to match your network design)
```

For Raspberry Pi or standalone Debian systems:

```
OS:          Raspberry Pi OS or Debian 12/13
RAM:         512 MB minimum, 1 GB recommended
Disk:        4 GB minimum, 8 GB recommended
Network:     Static IP or DHCP reservation
```

Before installing, make sure:

- The host or container has a static IP address or DHCP reservation.
- No other service is already listening on port 53.
- Outbound DNS traffic to the internet on TCP/UDP port 53 is allowed.
- The system can reach the DNS root servers directly.
- You have root access or sudo permissions.
- Pi-hole DHCP is disabled unless you explicitly want Pi-hole to act as DHCP server.
- The Pi-hole admin interface is not exposed directly to the internet.

Deployment guidance:

- Use a native Debian LXC on Proxmox VE.
- Do not use Docker inside LXC unless you have a specific operational reason.
- Keep the system dedicated to Pi-hole and Unbound.
- Let Pi-hole handle filtering.
- Let Unbound handle recursive DNS and DNSSEC validation.

---

## Proxmox LXC preparation

If you are installing on Proxmox VE, use a lightweight Debian 13 unprivileged LXC container.

Update the Proxmox template list:

```bash
pveam update
```

List available Debian templates:

```bash
pveam available --section system | grep debian
```

Download the Debian 13 standard template:

```bash
pveam download local debian-13-standard_13.1-2_amd64.tar.zst
```

### Select the target storage

By default, this guide uses `local-lvm` for the container root disk:

```
--rootfs local-lvm:8
```

List your available Proxmox storage backends first:

```bash
pvesm status
```

Use the storage ID that fits your setup. The number `8` means an 8 GB root disk, which is sufficient for Pi-hole and Unbound.

Create a new unprivileged LXC container:

```bash
pct create 110 local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst \
  --hostname lxc-pihole01 \
  --cores 1 \
  --memory 512 \
  --swap 512 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,tag=20,ip=dhcp,type=veth \
  --unprivileged 1 \
  --features nesting=1 \
  --onboot 1 \
  --start 1
```

> `tag=20` assigns the container to the Servers VLAN (192.168.20.0/24). Adjust to match your VLAN design. If you use a different VLAN or no VLAN tagging, omit the `tag=` parameter.

Enter the container:

```bash
pct enter 110
```

Update the container and install base packages:

```bash
apt update
apt upgrade -y
apt install -y curl wget dnsutils sudo ca-certificates unbound openssh-server locales
```

### Configure system locale

Some minimal Debian LXC templates show locale warnings. Fix this before continuing:

```bash
sed -i 's/^# *en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
locale-gen
update-locale LANG=en_US.UTF-8
source /etc/default/locale
locale
```

Expected result:

```
LANG=en_US.UTF-8
```

### Create an administrative user

Do not use root SSH login for the installation.

```bash
adduser piholeadmin
usermod -aG sudo piholeadmin
```

Validate:

```bash
groups piholeadmin
```

Expected result should include `sudo`.

### Enable SSH access

```bash
systemctl enable --now ssh
systemctl status ssh --no-pager
```

### Configure a static IP address

For DNS infrastructure, use a stable IP address. Prefer a DHCP reservation on your router — see the [homelab network README](https://github.com/pixelkeep/homelab-network) for UniFi DHCP reservation instructions.

To configure a static IP directly in Proxmox:

```bash
exit
```

From the Proxmox host:

```bash
pct stop 110
pct set 110 --net0 name=eth0,bridge=vmbr0,tag=20,ip=192.168.20.11/24,gw=192.168.20.1,type=veth
pct start 110
```

Adjust the IP address, gateway, and VLAN tag to match your network.

### Validate basic connectivity

```bash
pct enter 110
ip a
ip route
ping -c 3 1.1.1.1
ping -c 3 debian.org
```

### Continue installation over SSH

```bash
exit
ssh piholeadmin@192.168.20.11
sudo whoami
```

Expected result: `root`

---

## Debian or Raspberry Pi preparation

Update the system and install required base packages:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl wget dnsutils sudo ca-certificates unbound locales
```

Configure the default locale if needed:

```bash
sudo sed -i 's/^# *en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
sudo locale-gen
sudo update-locale LANG=en_US.UTF-8
```

Validate basic connectivity:

```bash
ip a
ip route
ping -c 3 1.1.1.1
ping -c 3 debian.org
```

---

## Pre-flight checks

Check if anything is already using DNS port 53:

```bash
sudo ss -tulpn | grep ':53 ' || true
```

Check whether the system can reach the DNS root servers directly:

```bash
dig @198.41.0.4 . NS +norec +time=3
dig @198.41.0.4 . NS +norec +tcp +time=3
```

If these tests fail, your ISP, router, or firewall may be blocking direct DNS traffic. Fix that before continuing.

---

## Install Pi-hole

Log in over SSH using the administrative user:

```bash
ssh piholeadmin@192.168.20.11
```

Download the official Pi-hole installer:

```bash
curl -fsSL https://install.pi-hole.net -o /tmp/pihole-install.sh
ls -lh /tmp/pihole-install.sh
head -n 5 /tmp/pihole-install.sh
```

### Fix terminal line drawing issues (optional)

If the installer shows broken characters (`q`, `x`, `l`, `k`), set the terminal environment first:

```bash
export TERM=xterm-256color
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
export NCURSES_NO_UTF8_ACS=1
```

Start the installer:

```bash
sudo bash /tmp/pihole-install.sh
```

During installation:

- Choose the network interface used by the container.
- Confirm the static IP address.
- Select any temporary upstream DNS provider (e.g. `1.1.1.1`). This will be replaced by Unbound.
- Enable the web interface.
- Enable query logging according to your privacy requirements.

After installation, set the Pi-hole web admin password:

```bash
sudo pihole setpassword
```

Check status:

```bash
pihole status
```

---

## Install and configure Unbound

Install Unbound if not already installed:

```bash
sudo apt update
sudo apt install -y unbound dnsutils
```

Create the Pi-hole specific Unbound configuration:

```bash
sudo tee /etc/unbound/unbound.conf.d/pi-hole.conf > /dev/null <<'EOF'
server:
    verbosity: 0

    interface: 127.0.0.1
    port: 5335

    do-ip4: yes
    do-ip6: no
    do-udp: yes
    do-tcp: yes

    # Recommended EDNS buffer size to avoid fragmentation issues
    edns-buffer-size: 1232

    # Security hardening
    harden-glue: yes
    harden-dnssec-stripped: yes
    harden-below-nxdomain: yes
    harden-referral-path: yes
    use-caps-for-id: no

    # Privacy and efficiency
    qname-minimisation: yes
    prefetch: yes
    aggressive-nsec: yes

    # Hide local resolver identity
    hide-identity: yes
    hide-version: yes

    # Keep small for lightweight LXC usage
    num-threads: 1
    so-rcvbuf: 1m

    # Block private address ranges from appearing in public DNS responses
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10
    private-address: 192.0.2.0/24
    private-address: 198.51.100.0/24
    private-address: 203.0.113.0/24
    private-address: 255.255.255.255/32
    private-address: 2001:db8::/32
EOF
```

---

## Debian Bullseye / Bookworm / Trixie resolvconf fix

On modern Debian releases, `unbound-resolvconf.service` may create unwanted resolver configuration. Disable it:

```bash
sudo systemctl disable --now unbound-resolvconf.service 2>/dev/null || true

if [ -f /etc/resolvconf.conf ]; then
  sudo sed -Ei 's/^unbound_conf=/#unbound_conf=/' /etc/resolvconf.conf
fi

sudo rm -f /etc/unbound/unbound.conf.d/resolvconf_resolvers.conf
```

---

## Validate and restart Unbound

```bash
sudo unbound-checkconf
sudo systemctl restart unbound
sudo systemctl enable unbound
```

Check that Unbound listens only on localhost port 5335:

```bash
sudo ss -tulpn | grep 5335
```

Expected: `127.0.0.1:5335` — **not** `0.0.0.0:5335`. If Unbound listens on the LAN IP, fix the configuration before continuing.

Test recursive resolution:

```bash
dig pi-hole.net @127.0.0.1 -p 5335
```

Test DNSSEC validation:

```bash
dig fail01.dnssec.works @127.0.0.1 -p 5335
dig +ad dnssec.works @127.0.0.1 -p 5335
```

Expected result:

```
fail01.dnssec.works → SERVFAIL   (invalid DNSSEC, correctly rejected)
dnssec.works        → NOERROR with ad flag  (valid DNSSEC, correctly accepted)
```

---

## Point Pi-hole to Unbound

Open the Pi-hole admin UI:

```
http://192.168.20.11/admin
```

Go to:

```
Settings > DNS
```

Configure:

```
Custom DNS server: 127.0.0.1#5335
```

Disable all other upstream DNS providers. Save the settings.

Flush Pi-hole DNS cache:

```bash
sudo pihole reloaddns
```

---

## Verify Pi-hole is using Unbound

From a client machine, query Pi-hole:

```bash
dig pi-hole.net @192.168.20.11
```

On the Pi-hole host, check the live query log:

```bash
sudo pihole tail
```

You should see client queries arriving at Pi-hole, which forwards allowed domains to Unbound on `127.0.0.1#5335`.

---

## Quick validation

```bash
# Test Unbound directly
dig pi-hole.net @127.0.0.1 -p 5335

# Test DNSSEC
dig fail01.dnssec.works @127.0.0.1 -p 5335
dig +ad dnssec.works @127.0.0.1 -p 5335

# Test Pi-hole
dig pi-hole.net @127.0.0.1
dig pi-hole.net @192.168.20.11

# Live query log
sudo pihole tail
```

Expected DNSSEC results:

```
fail01.dnssec.works → SERVFAIL
dnssec.works        → NOERROR with ad flag
```

---

## Security hardening

Security baseline:

```
Internet exposure:      none
SSH:                    admin network only
Pi-hole web UI:         admin network only
DNS port 53:            trusted LAN/VLAN clients only
Unbound port 5335:      localhost only
Root SSH login:         disabled
Updates:                Debian security updates automatic
Pi-hole updates:        manual after backup/snapshot
```

### Network exposure baseline

Recommended exposure:

```
Service        Port        Source
DNS            53/tcp      Trusted LAN/VLAN clients only
DNS            53/udp      Trusted LAN/VLAN clients only
Pi-hole UI     80/tcp      Admin network only
Pi-hole UI     443/tcp     Admin network only, if enabled
SSH            22/tcp      Admin network only
Unbound        5335/tcp    Localhost only — never exposed externally
Unbound        5335/udp    Localhost only — never exposed externally
NTP            123/udp     Only if Pi-hole NTP server is intentionally enabled
DHCP           67/udp      Only if Pi-hole DHCP server is intentionally enabled
```

Validate Unbound is localhost-only:

```bash
sudo ss -tulpn | grep 5335
```

Expected: `127.0.0.1:5335`

### Administrative user

```bash
sudo adduser piholeadmin
sudo usermod -aG sudo piholeadmin
groups piholeadmin
```

### SSH hardening

Disable root SSH login:

```bash
sudo tee /etc/ssh/sshd_config.d/99-disable-root-login.conf > /dev/null <<'EOF'
PermitRootLogin no
EOF

sudo sshd -t
sudo systemctl reload ssh
```

Validate:

```bash
sshd -T | grep permitrootlogin
```

Expected: `permitrootlogin no`

Copy your SSH key to the admin user and only then disable password authentication:

```bash
ssh-copy-id piholeadmin@192.168.20.11
```

Validate key login from a new terminal first, then:

```bash
sudo tee /etc/ssh/sshd_config.d/99-disable-password-login.conf > /dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
EOF

sudo sshd -t
sudo systemctl reload ssh
```

> Keep an active SSH session open while testing SSH changes. If SSH access breaks on Proxmox LXC, use `pct enter <CTID>` from the Proxmox host to recover.

### Firewall guidance

Preferred control points:

- Router/firewall: allow trusted client VLANs to reach Pi-hole on port 53.
- Router/firewall: block internet access to SSH and Pi-hole web interface.
- Proxmox firewall: restrict LXC access to trusted admin and LAN/VLAN networks.
- Local UFW: optional, useful for Raspberry Pi or standalone Debian.

Do not expose these ports from the internet:

```
22/tcp  53/tcp  53/udp  80/tcp  443/tcp  5335/tcp  5335/udp
```

### Optional local firewall with UFW

```bash
sudo apt update
sudo apt install -y ufw

sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH from admin network
sudo ufw allow from 192.168.10.0/24 to any port 22 proto tcp
sudo ufw allow from 192.168.20.0/24 to any port 22 proto tcp

# Pi-hole web UI from admin/trusted network
sudo ufw allow from 192.168.10.0/24 to any port 80 proto tcp
sudo ufw allow from 192.168.30.0/24 to any port 80 proto tcp

# DNS from all trusted VLANs
sudo ufw allow from 192.168.10.0/24 to any port 53
sudo ufw allow from 192.168.20.0/24 to any port 53
sudo ufw allow from 192.168.30.0/24 to any port 53
sudo ufw allow from 192.168.40.0/24 to any port 53

sudo ufw enable
sudo ufw status verbose
```

Adjust subnet ranges to match your VLAN design. See the [homelab network README](https://github.com/pixelkeep/homelab-network) for the full VLAN overview.

Do not allow inbound access to Unbound port `5335`.

### Proxmox LXC-specific hardening

Validate unprivileged container:

```bash
pct config 110 | grep unprivileged
```

Expected: `unprivileged: 1`

```bash
pct config 110 | grep features
```

Expected includes: `features: nesting=1`

### Proxmox firewall recommendation

```
Inbound default:  DROP
Outbound default: ACCEPT

ALLOW  tcp  192.168.10.0/24  → PIHOLE_IP  port 22    (SSH from Management)
ALLOW  tcp  192.168.10.0/24  → PIHOLE_IP  port 80    (Pi-hole UI from Management)
ALLOW  tcp  192.168.30.0/24  → PIHOLE_IP  port 80    (Pi-hole UI from Trusted LAN)
ALLOW  udp  192.168.10.0/24  → PIHOLE_IP  port 53    (DNS from Management)
ALLOW  udp  192.168.20.0/24  → PIHOLE_IP  port 53    (DNS from Servers)
ALLOW  udp  192.168.30.0/24  → PIHOLE_IP  port 53    (DNS from Trusted LAN)
ALLOW  udp  192.168.40.0/24  → PIHOLE_IP  port 53    (DNS from IoT)
ALLOW  tcp  192.168.10.0/24  → PIHOLE_IP  port 53
ALLOW  tcp  192.168.20.0/24  → PIHOLE_IP  port 53
ALLOW  tcp  192.168.30.0/24  → PIHOLE_IP  port 53
ALLOW  tcp  192.168.40.0/24  → PIHOLE_IP  port 53
DROP   all  any              → PIHOLE_IP
```

### Raspberry Pi-specific hardening

- Keep Raspberry Pi OS updated.
- Disable unused services.
- Do not expose SSH or the Pi-hole web interface to the internet.
- Use SSH keys where possible.
- Use a reliable wired connection where possible.
- Use a reliable power supply.
- Use a DHCP reservation or static IP.
- Make regular backups of the SD card or boot disk.

Check enabled services:

```bash
systemctl list-unit-files --state=enabled
sudo ss -tulpn
```

### Pi-hole NTP behavior

Pi-hole v6 includes NTP functionality. For Proxmox LXC, let the Proxmox host manage system time and disable Pi-hole's NTP sync inside the container:

```bash
sudo pihole-FTL --config ntp.sync.active false
sudo systemctl restart pihole-FTL
```

Validate:

```bash
sudo pihole-FTL --config ntp.sync.active
```

Expected: `false`

Optionally disable the Pi-hole NTP server:

```bash
sudo pihole-FTL --config ntp.ipv4.active false
sudo pihole-FTL --config ntp.ipv6.active false
sudo systemctl restart pihole-FTL
```

### Fail2ban

Fail2ban is not required for the default deployment model. Preferred controls in order:

1. Do not expose SSH to the internet
2. Restrict SSH to trusted admin networks
3. Disable root SSH login
4. Prefer SSH keys over passwords
5. Use Proxmox, router, or firewall rules for access control
6. Fail2ban as optional additional control

Do not make a Proxmox LXC container privileged only to support Fail2ban.

---

## Automatic Debian security updates

Pi-hole is not updated by Debian package updates, but system packages (`unbound`, `openssh-server`, `ca-certificates`, libraries) should receive security updates automatically.

```bash
sudo apt update
sudo apt install -y unattended-upgrades apt-listchanges
```

Create the APT periodic update configuration:

```bash
sudo tee /etc/apt/apt.conf.d/20auto-upgrades > /dev/null <<'EOF'
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
EOF
```

Create a local policy:

```bash
sudo tee /etc/apt/apt.conf.d/52unattended-upgrades-local > /dev/null <<'EOF'
Unattended-Upgrade::Automatic-Reboot "false";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::SyslogEnable "true";
EOF
```

Validate:

```bash
sudo unattended-upgrade --dry-run --debug
systemctl status apt-daily.timer apt-daily-upgrade.timer --no-pager
sudo tail -n 100 /var/log/unattended-upgrades/unattended-upgrades.log
```

---

## Pi-hole maintenance and updates

Pi-hole is not updated by `unattended-upgrades`. Manage it manually.

```bash
# Check installed versions
pihole version

# Check status
pihole status

# Update blocklists
sudo pihole updateGravity
# short form:
sudo pihole -g
```

---

## Pi-hole update procedure

Before updating, create a backup or snapshot.

For Proxmox LXC:

```bash
pct snapshot 110 pre-pihole-update-$(date +%Y%m%d)
```

For Raspberry Pi: export Pi-hole configuration via the web interface using Teleporter, or back up `/etc/pihole` and `/etc/unbound`.

Then update:

```bash
sudo pihole -up
```

After the update, validate:

```bash
pihole status
sudo systemctl status pihole-FTL --no-pager
sudo systemctl status unbound --no-pager
dig pi-hole.net @127.0.0.1
dig pi-hole.net @192.168.20.11
dig pi-hole.net @127.0.0.1 -p 5335
dig fail01.dnssec.works @127.0.0.1 -p 5335
dig +ad dnssec.works @127.0.0.1 -p 5335
```

---

## Backup recommendation

For Proxmox LXC:

```
Backup frequency:   daily or weekly
Mode:               snapshot
Retention:          at least 3 recent backups
Scope:              LXC container including root disk
```

Before major changes:

```bash
pct snapshot 110 before-major-change-$(date +%Y%m%d)
```

For Raspberry Pi — use one or more of:

- SD card image backup
- Filesystem backup
- Pi-hole Teleporter export (web UI)
- Backup of `/etc/pihole` and `/etc/unbound`

Major changes that warrant a backup first:

- Pi-hole version update
- Debian release upgrade
- Unbound configuration change
- Network or VLAN change
- Firewall policy change

---

## Security validation checklist

Run after installation and after major changes.

```bash
# Check listening ports
sudo ss -tulpn

# Validate Unbound is localhost-only
sudo ss -tulpn | grep 5335

# Pi-hole and Unbound status
pihole status
sudo systemctl status unbound --no-pager

# SSH root login policy
sshd -T | grep permitrootlogin

# DNS through Pi-hole
dig pi-hole.net @192.168.20.11

# Unbound directly
dig pi-hole.net @127.0.0.1 -p 5335

# DNSSEC validation
dig fail01.dnssec.works @127.0.0.1 -p 5335
dig +ad dnssec.works @127.0.0.1 -p 5335

# Live Pi-hole log
sudo pihole tail

# Recent Unbound log
journalctl -u unbound -n 100 --no-pager

# Recent SSH log
journalctl -u ssh -n 100 --no-pager

# Automatic updates status
systemctl status apt-daily.timer apt-daily-upgrade.timer --no-pager
sudo tail -n 50 /var/log/unattended-upgrades/unattended-upgrades.log
```

---

## Sources

- Pi-hole documentation: <https://docs.pi-hole.net/>
- Pi-hole Unbound integration guide: <https://docs.pi-hole.net/guides/dns/unbound/>
- Unbound documentation: <https://unbound.docs.nlnetlabs.nl/>
- DNSSEC explained (ICANN): <https://www.icann.org/resources/pages/dnssec-what-is-it-why-important-2019-03-05-en>
- Debian unattended-upgrades: <https://wiki.debian.org/UnattendedUpgrades>
- Proxmox VE documentation: <https://pve.proxmox.com/pve-docs/>
- Homelab network design: <https://github.com/pixelkeep/homelab-network>
- Raspberry Pi OS setup: <https://github.com/pixelkeep/rpi-setup>
- Caddy reverse proxy setup: <https://github.com/pixelkeep/caddy-pixelkeep>
