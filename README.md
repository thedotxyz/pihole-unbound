# Pi-hole + Unbound Recursive DNS

<p align="center">
  <img src="https://img.shields.io/badge/Pi--hole-DNS%20sinkhole-96060C?style=for-the-badge&logo=pi-hole&logoColor=white" alt="Pi-hole">
  <img src="https://img.shields.io/badge/Unbound-Recursive%20DNS-005A9C?style=for-the-badge" alt="Unbound">
  <img src="https://img.shields.io/badge/Proxmox-LXC-E57000?style=for-the-badge&logo=proxmox&logoColor=white" alt="Proxmox LXC">
  <img src="https://img.shields.io/badge/Debian-12%20%7C%2013-A81D33?style=for-the-badge&logo=debian&logoColor=white" alt="Debian">
  <img src="https://img.shields.io/badge/Raspberry%20Pi-supported-A22846?style=for-the-badge&logo=raspberrypi&logoColor=white" alt="Raspberry Pi">
</p>

Lightweight setup for running **Pi-hole** with **Unbound** as a local recursive DNS resolver.

This guide supports:

- Proxmox VE with an unprivileged Debian LXC container
- Debian 12 Bookworm
- Debian 13 Trixie
- Raspberry Pi OS / Debian-based systems

## What this setup does

Pi-hole provides DNS filtering, blocklists and the web interface.

Unbound provides local recursive DNS resolution and DNSSEC validation.

The target architecture is:

```text
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
## Key references

- Pi-hole documentation: https://docs.pi-hole.net/
- Pi-hole Unbound guide: https://docs.pi-hole.net/guides/dns/unbound/
- DNSSEC overview: https://www.icann.org/resources/pages/dnssec-what-is-it-why-important-2019-03-05-en
- Unbound documentation: https://unbound.docs.nlnetlabs.nl/

## Contents

- [Prerequisites](#prerequisites)
- [Proxmox LXC preparation](#proxmox-lxc-preparation)
- [Pre-flight checks](#pre-flight-checks)
- [Install Pi-hole](#install-pi-hole)
- [Install and configure Unbound](#install-and-configure-unbound)
- [Point Pi-hole to Unbound](#point-pi-hole-to-unbound)
- [Verify Pi-hole is using Unbound](#verify-pi-hole-is-using-unbound)
- [Security hardening](#security-hardening)
- [Automatic Debian security updates](#automatic-debian-security-updates)
- [Pi-hole maintenance and updates](#pi-hole-maintenance-and-updates)
- [Backup recommendation](#backup-recommendation)
- [Security validation checklist](#security-validation-checklist)
- [Sources](#sources)

## Prerequisites

This guide supports the following deployment models:

```text
Proxmox VE    Debian 13 or Debian 12 unprivileged LXC
Raspberry Pi Raspberry Pi OS based on Debian
Debian        Debian 13 Trixie or Debian 12 Bookworm
```

For Proxmox VE, the recommended container profile is:

```text
Type:        Unprivileged LXC
OS:          Debian 13 standard template
CPU:         1 vCPU
RAM:         512 MB minimum, 1 GB recommended
Disk:        4 GB minimum, 8 GB recommended
Network:     Bridged network with static IP or DHCP reservation
```

For Raspberry Pi or standalone Debian systems, use:

```text
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

## Proxmox LXC preparation
If you are installing this on Proxmox VE, use a lightweight Debian 13 unprivileged LXC container.

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

```bash
--rootfs local-lvm:8
```

This is the default Proxmox LVM-thin storage on many installations.

If you have a separate SSD/NVMe storage pool, you can place the container there instead. First list available Proxmox storage backends:

```bash
pvesm status
```

Example output may include storage IDs such as:

```text
local
local-lvm
ssd4tb
vmdata
```

Use the storage ID you want for the container root disk.

Examples:

```bash
--rootfs local-lvm:8
```

or:

```bash
--rootfs ssd4tb:8
```

The number `8` means an 8 GB container root disk.
For Pi-hole and Unbound, 8 GB is usually sufficient.

Create a new unprivileged LXC container:

```bash
pct create 110 local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst \
  --hostname DEBPIH01 \
  --cores 1 \
  --memory 512 \
  --swap 512 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp,type=veth \
  --unprivileged 1 \
  --features nesting=1 \
  --onboot 1 \
  --start 1
```

If you want to use another Proxmox storage backend, replace `local-lvm` with your own storage ID.

Example:

```bash
--rootfs ssd4tb:8
```

Enter the container:

```bash
pct enter 110
```

Update the container and install basic packages:

```bash
apt update
apt upgrade -y
apt install -y curl wget dnsutils sudo ca-certificates unbound openssh-server locales
```

### Configure system locale

Some minimal Debian LXC templates may show locale warnings such as:

```text
perl: warning: Setting locale failed.
perl: warning: Falling back to the standard locale ("C").
```

This is not fatal, but it should be fixed before continuing.

Generate and configure the default locale:

```bash
sed -i 's/^# *en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
locale-gen
update-locale LANG=en_US.UTF-8
```

Reload the locale settings:

```bash
source /etc/default/locale
```

Validate the locale configuration:

```bash
locale
```

Expected result:

```text
LANG=en_US.UTF-8
```

### Create an administrative user

Do not use root SSH login for the installation.

Create a dedicated administrative user and grant sudo permissions:

```bash
adduser piholeadmin
usermod -aG sudo piholeadmin
```

Validate the sudo group membership:

```bash
groups piholeadmin
```

Expected result should include:

```text
sudo
```

### Enable SSH access

Enable SSH inside the container.

This is recommended because the interactive Pi-hole installer may not behave correctly when started through `pct enter`.

```bash
systemctl enable --now ssh
```

Validate that SSH is running:

```bash
systemctl status ssh --no-pager
```

### Configure a static IP address

For DNS infrastructure, use a stable IP address.

Either configure a DHCP reservation on your router/firewall, or configure a static IP address in Proxmox.

To configure a static IP address in Proxmox, first exit the container:

```bash
exit
```

Then run this on the Proxmox host:

```bash
pct stop 110
pct set 110 --net0 name=eth0,bridge=vmbr0,ip=192.168.1.30/24,gw=192.168.1.1,type=veth
pct start 110
```

Adjust the IP address and gateway to match your own network.

### Validate basic connectivity

Enter the container again:

```bash
pct enter 110
```

Validate basic network connectivity from inside the container:

```bash
ip a
ip route
ping -c 3 1.1.1.1
ping -c 3 debian.org
```

Exit the container:

```bash
exit
```

### Continue installation over SSH

Continue the installation over SSH using the administrative user.

Example:

```bash
ssh piholeadmin@192.168.1.30
```

Adjust the IP address to match your container.

Validate sudo access:

```bash
sudo whoami
```

Expected result:

```text
root
```

After this, continue with the pre-flight checks and the Pi-hole installation.

## Debian or Raspberry Pi preparation

If you are installing on Raspberry Pi OS or a standalone Debian system, update the system and install the required base packages:

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

After this, continue with the pre-flight checks.

## Pre-flight checks

Run these checks from the SSH session inside the container.

Check if anything is already using DNS port 53:

```bash
sudo ss -tulpn | grep ':53 ' || true
```

Check whether the system can reach the DNS root servers directly:

```bash
dig @198.41.0.4 . NS +norec +time=3
dig @198.41.0.4 . NS +norec +tcp +time=3
dig @198.41.0.4 version.bind CH TXT +time=3
```

If these tests fail, your ISP, router or firewall may be blocking or intercepting direct DNS traffic.

Fix that before continuing.

After this, continue with the Pi-hole installation over SSH.

## Install Pi-hole

Install Pi-hole using the official Pi-hole installer.

Log in over SSH using the administrative user created earlier (if you are not logged in already):

```bash
ssh piholeadmin@192.168.1.30
```

(Don't forget to adjust the IP address to match your container).

If the issue persists, make sure your SSH client uses UTF-8 encoding and an `xterm-256color` terminal type.

Download the official Pi-hole installer:

```bash
curl -fsSL https://install.pi-hole.net -o /tmp/pihole-install.sh
```
Validate that the installer was downloaded:

```bash
ls -lh /tmp/pihole-install.sh
head -n 5 /tmp/pihole-install.sh
```
### (Optional) Fix terminal line drawing issues

If the Pi-hole installer shows broken line drawing characters such as `q`, `x`, `l` and `k`, the issue is usually the SSH terminal emulation, not Pi-hole.

Set the terminal environment before starting the installer:

```bash
export TERM=xterm-256color
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
export NCURSES_NO_UTF8_ACS=1
```

Then start the installer with sudo:

```bash
sudo bash /tmp/pihole-install.sh
```

During installation:

- Choose the network interface used by the container.
- Confirm the static IP address.
- Select any temporary upstream DNS provider, for example Cloudflare `1.1.1.1`. This will be replaced later by Unbound.
- Enable the web interface.
- Enable query logging according to your own privacy requirements.

After the installation, set or change the Pi-hole web admin password:

```bash
sudo pihole setpassword
```
Enter the new password twice when prompted.

Check Pi-hole status:

```bash
pihole status
```

## Install and configure Unbound

Install Unbound if it is not already installed:

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

    # Keep this small for lightweight LXC usage
    num-threads: 1
    so-rcvbuf: 1m

    # Do not return private addresses from public DNS
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

## Debian Bullseye / Bookworm / Trixie resolvconf fix

On modern Debian releases, `unbound-resolvconf.service` may create unwanted resolver configuration.

Disable it and remove the generated resolver file:

```bash
sudo systemctl disable --now unbound-resolvconf.service 2>/dev/null || true

if [ -f /etc/resolvconf.conf ]; then
  sudo sed -Ei 's/^unbound_conf=/#unbound_conf=/' /etc/resolvconf.conf
fi

sudo rm -f /etc/unbound/unbound.conf.d/resolvconf_resolvers.conf
```

## Validate and restart Unbound

Check the Unbound configuration:

```bash
sudo unbound-checkconf
```

Restart Unbound:

```bash
sudo systemctl restart unbound
sudo systemctl enable unbound
```

Check if Unbound is listening on localhost port 5335:

```bash
sudo ss -tulpn | grep 5335
```

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

- `fail01.dnssec.works` should fail DNSSEC validation.
- `dnssec.works` should return a valid answer with the `ad` flag.

## Point Pi-hole to Unbound

Open the Pi-hole admin UI:

```text
http://<PIHOLE-IP>/admin
```

Go to:

```text
Settings > DNS
```

Configure:

```text
Custom DNS server: 127.0.0.1#5335
```

Disable all other upstream DNS providers.

Save the settings.

Flush Pi-hole DNS cache:

```bash
sudo pihole reloaddns
```
### Pi-hole NTP behavior in Proxmox LXC

Pi-hole v6 includes NTP functionality.

In an unprivileged Proxmox LXC container, Pi-hole cannot adjust the system time because the container does not have permission to set kernel time.

For Proxmox LXC, let the Proxmox host manage system time and disable Pi-hole's NTP time sync client inside the container:

```bash
sudo pihole-FTL --config ntp.sync.active false
sudo systemctl restart pihole-FTL
```

Validate the setting:

```bash
sudo pihole-FTL --config ntp.sync.active
```

Expected result:

```text
false
```

Optional: if you do not want Pi-hole to act as an NTP server for your LAN, disable the Pi-hole NTP server as well:

```bash
sudo pihole-FTL --config ntp.ipv4.active false
sudo pihole-FTL --config ntp.ipv6.active false
sudo systemctl restart pihole-FTL
```

Do not make the container privileged only to resolve this warning. For DNS infrastructure, the Proxmox host should manage system time.

## Verify Pi-hole is using Unbound

From a client machine, query Pi-hole:

```bash
dig pi-hole.net @<PIHOLE-IP>
```

On the Pi-hole host/container, verify queries in the live log:

```bash
sudo pihole tail
```

You should see client queries arriving at Pi-hole. Pi-hole should forward allowed domains to Unbound on `127.0.0.1#5335`.

## Quick validation

Run these commands after completing the installation.

Test Unbound directly:

```bash
dig pi-hole.net @127.0.0.1 -p 5335
```

Test DNSSEC through Unbound:

```bash
dig fail01.dnssec.works @127.0.0.1 -p 5335
dig +ad dnssec.works @127.0.0.1 -p 5335
```

Expected result:

```text
fail01.dnssec.works -> SERVFAIL
dnssec.works        -> NOERROR with ad flag
```

Test Pi-hole:

```bash
dig pi-hole.net @127.0.0.1
dig pi-hole.net @<PIHOLE-IP>
```

Check live Pi-hole queries:

```bash
sudo pihole tail
```

## Security hardening

This guide supports both:

- Proxmox VE unprivileged LXC containers
- Raspberry Pi OS / Debian-based systems

The security baseline is the same for both:

```text
Internet exposure:     none
SSH:                   admin network only
Pi-hole web UI:         admin network only
DNS port 53:            trusted LAN/VLAN clients only
Unbound port 5335:      localhost only
Root SSH login:         disabled
Updates:                Debian security updates automatic
Pi-hole updates:        manual after backup/snapshot
```

## Network exposure baseline

Do not expose this system directly to the internet.

Recommended exposure:

```text
Service        Port        Source
DNS            53/tcp      Trusted LAN/VLAN clients only
DNS            53/udp      Trusted LAN/VLAN clients only
Pi-hole UI     80/tcp      Admin network only
Pi-hole UI     443/tcp     Admin network only, if enabled
SSH            22/tcp      Admin network only
Unbound        5335/tcp    Localhost only
Unbound        5335/udp    Localhost only
NTP            123/udp     Only if Pi-hole NTP server is intentionally enabled
DHCP           67/udp      Only if Pi-hole DHCP server is intentionally enabled
```

Unbound should only listen on localhost:

```bash
sudo ss -tulpn | grep 5335
```

Expected result:

```text
127.0.0.1:5335
```

If Unbound listens on `0.0.0.0:5335` or on the LAN IP address, fix the Unbound configuration before continuing.

## Administrative user

Use a dedicated administrative user with `sudo`.

Do not use root SSH login for normal administration.

Example:

```bash
sudo adduser piholeadmin
sudo usermod -aG sudo piholeadmin
```

Validate group membership:

```bash
groups piholeadmin
```

Expected result should include:

```text
sudo
```

## SSH hardening

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

Expected result:

```text
permitrootlogin no
```

Optional: use SSH keys and disable password authentication.

First copy your SSH key to the administrative user:

```bash
ssh-copy-id piholeadmin@<PIHOLE-IP>
```

Validate key-based login from a new terminal:

```bash
ssh piholeadmin@<PIHOLE-IP>
```

Only after validating SSH key login, disable password authentication:

```bash
sudo tee /etc/ssh/sshd_config.d/99-disable-password-login.conf > /dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
EOF

sudo sshd -t
sudo systemctl reload ssh
```

Validate:

```bash
sshd -T | grep -E 'passwordauthentication|kbdinteractiveauthentication'
```

Expected result:

```text
passwordauthentication no
kbdinteractiveauthentication no
```

Keep an active SSH session open while testing SSH changes.

If SSH access breaks:

- On Proxmox LXC, use `pct enter <CTID>` from the Proxmox host.
- On Raspberry Pi, use local console access or attach keyboard/monitor.

## Firewall guidance

Prefer enforcing access control at the router/firewall or Proxmox layer.

Recommended control points:

- Router/firewall: allow trusted client networks to reach Pi-hole on port 53.
- Router/firewall: block internet access to SSH and the Pi-hole web interface.
- Proxmox firewall: restrict LXC access to trusted admin and LAN/VLAN networks.
- Local firewall: optional, useful for Raspberry Pi or standalone Debian systems.

Do not expose these ports from the internet:

```text
22/tcp
53/tcp
53/udp
80/tcp
443/tcp
5335/tcp
5335/udp
```

Preferred model:

```text
Client LAN/VLANs  -> Pi-hole DNS :53
Admin network     -> SSH :22 and Pi-hole UI :80/:443
Internet          -> no inbound access
```

## Optional local firewall with UFW

This section is optional.

Use this on Raspberry Pi or standalone Debian systems if you want local host firewalling.

For Proxmox LXC, prefer the Proxmox firewall or router/firewall rules first.

Install UFW:

```bash
sudo apt update
sudo apt install -y ufw
```

Set default policies:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Allow SSH from the admin network:

```bash
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp
```

Allow Pi-hole web UI from the admin network:

```bash
sudo ufw allow from 192.168.1.0/24 to any port 80 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 443 proto tcp
```

Allow DNS from trusted LAN/VLAN networks:

```bash
sudo ufw allow from 192.168.1.0/24 to any port 53 proto udp
sudo ufw allow from 192.168.1.0/24 to any port 53 proto tcp
```

Only if Pi-hole is intentionally used as NTP server:

```bash
sudo ufw allow from 192.168.1.0/24 to any port 123 proto udp
```

Only if Pi-hole is intentionally used as DHCP server:

```bash
sudo ufw allow from 192.168.1.0/24 to any port 67 proto udp
```

Enable UFW:

```bash
sudo ufw enable
```

Validate:

```bash
sudo ufw status verbose
```

Adjust `192.168.1.0/24` to match your own admin and client networks.

Do not allow inbound access to Unbound port `5335`. Unbound should only listen on `127.0.0.1`.

## Proxmox LXC-specific hardening

This setup should run as an unprivileged container.

Validate from the Proxmox host:

```bash
pct config 110 | grep unprivileged
```

Expected result:

```text
unprivileged: 1
```

Validate nesting if Debian 13 is used:

```bash
pct config 110 | grep features
```

Expected result should include:

```text
features: nesting=1
```

Do not make the container privileged only to solve application warnings.

For Pi-hole and Unbound, a privileged container is not required.

## Proxmox firewall recommendation

If you use the Proxmox firewall, apply a restrictive policy to the container.

Recommended policy:

```text
Inbound default:  DROP
Outbound default: ACCEPT
```

Allow only:

```text
22/tcp   from admin network
80/tcp   from admin network
443/tcp  from admin network, if used
53/tcp   from trusted LAN/VLAN networks
53/udp   from trusted LAN/VLAN networks
123/udp  from trusted LAN/VLAN networks, only if Pi-hole NTP server is enabled
67/udp   from trusted LAN/VLAN networks, only if Pi-hole DHCP server is enabled
```

Do not allow inbound access to Unbound port `5335`.

Example rule model:

```text
ALLOW  tcp  <ADMIN-NETWORK>  -> <PIHOLE-IP>  port 22
ALLOW  tcp  <ADMIN-NETWORK>  -> <PIHOLE-IP>  port 80
ALLOW  tcp  <ADMIN-NETWORK>  -> <PIHOLE-IP>  port 443
ALLOW  udp  <LAN-NETWORK>    -> <PIHOLE-IP>  port 53
ALLOW  tcp  <LAN-NETWORK>    -> <PIHOLE-IP>  port 53
DROP   all  any              -> <PIHOLE-IP>
```

Adjust `<ADMIN-NETWORK>`, `<LAN-NETWORK>` and `<PIHOLE-IP>` to match your environment.

## Raspberry Pi-specific hardening

For Raspberry Pi installations:

- Keep Raspberry Pi OS updated.
- Disable unused services.
- Do not expose SSH or the Pi-hole web interface to the internet.
- Use SSH keys where possible.
- Use a stable wired network connection where possible.
- Use a reliable power supply.
- Use a DHCP reservation or static IP address.
- Make regular backups of the SD card or boot disk.

Check enabled services:

```bash
systemctl list-unit-files --state=enabled
```

Check listening ports:

```bash
sudo ss -tulpn
```

Disable services you do not use.

Example:

```bash
sudo systemctl disable --now <service-name>
```

## Pi-hole NTP behavior

Pi-hole v6 includes NTP functionality.

For Proxmox LXC, let the Proxmox host manage system time and disable Pi-hole's NTP time sync client inside the container:

```bash
sudo pihole-FTL --config ntp.sync.active false
sudo systemctl restart pihole-FTL
```

Validate:

```bash
sudo pihole-FTL --config ntp.sync.active
```

Expected result:

```text
false
```

Optional: if you do not want Pi-hole to act as an NTP server for your LAN, disable the Pi-hole NTP server as well:

```bash
sudo pihole-FTL --config ntp.ipv4.active false
sudo pihole-FTL --config ntp.ipv6.active false
sudo systemctl restart pihole-FTL
```

On Raspberry Pi, Pi-hole NTP can be left enabled if you intentionally want the device to provide NTP services to your LAN.

## Fail2ban

Fail2ban is not required for the default deployment model.

Preferred controls:

- Do not expose SSH to the internet.
- Restrict SSH to trusted admin networks.
- Disable root SSH login.
- Prefer SSH keys over passwords.
- Use Proxmox, router or firewall rules for access control.

Fail2ban can be useful if SSH password authentication remains enabled and the system is reachable from a larger or less trusted network.

Do not make a Proxmox LXC container privileged only to support fail2ban.

If brute-force protection is required, prefer enforcing it at one of these layers:

```text
1. Router/firewall
2. Proxmox firewall
3. VPN / admin network segmentation
4. SSH key-only authentication
5. Fail2ban as optional additional control
```

## Automatic Debian security updates

Pi-hole itself is not updated by Debian package updates.

However, Debian packages such as `unbound`, `openssh-server`, `ca-certificates` and system libraries should receive security updates.

Install unattended upgrades:

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

Create a local unattended-upgrades policy:

```bash
sudo tee /etc/apt/apt.conf.d/52unattended-upgrades-local > /dev/null <<'EOF'
Unattended-Upgrade::Automatic-Reboot "false";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::SyslogEnable "true";
EOF
```

Validate the unattended-upgrades configuration:

```bash
sudo unattended-upgrade --dry-run --debug
```

Check APT timers:

```bash
systemctl status apt-daily.timer apt-daily-upgrade.timer --no-pager
```

Check unattended-upgrades logs:

```bash
sudo tail -n 100 /var/log/unattended-upgrades/unattended-upgrades.log
```

Manual Debian package update command:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt autoremove -y
```

## Pi-hole maintenance and updates

Pi-hole is not updated by Debian `unattended-upgrades`.

Check installed Pi-hole versions:

```bash
pihole version
```

Check Pi-hole status:

```bash
pihole status
```

Update gravity/blocklists manually:

```bash
sudo pihole updateGravity
```

Short form:

```bash
sudo pihole -g
```

Pi-hole updates gravity automatically on a schedule, but manual updates can be useful after changing adlists.

## Pi-hole update procedure

Before updating Pi-hole, create a backup or snapshot.

For Proxmox LXC, create a snapshot from the Proxmox host:

```bash
pct snapshot 110 pre-pihole-update-$(date +%Y%m%d)
```

For Raspberry Pi, create a system backup or export your Pi-hole configuration through the Pi-hole web interface using Teleporter.

Then update Pi-hole inside the system:

```bash
sudo pihole -up
```

After the update, validate services:

```bash
pihole status
sudo systemctl status pihole-FTL --no-pager
sudo systemctl status unbound --no-pager
```

Validate DNS resolution through Pi-hole:

```bash
dig pi-hole.net @127.0.0.1
dig pi-hole.net @<PIHOLE-IP>
```

Validate Unbound directly:

```bash
dig pi-hole.net @127.0.0.1 -p 5335
```

Validate DNSSEC:

```bash
dig fail01.dnssec.works @127.0.0.1 -p 5335
dig +ad dnssec.works @127.0.0.1 -p 5335
```

Expected result:

```text
fail01.dnssec.works -> SERVFAIL
dnssec.works        -> NOERROR with ad flag
```

## Backup recommendation

For Proxmox LXC, schedule regular backups of the container.

Recommended minimum:

```text
Backup frequency: daily or weekly
Mode:             snapshot
Retention:        at least 3 recent backups
Scope:            LXC container including root disk
```

Before major changes, create a manual snapshot:

```bash
pct snapshot 110 before-major-change-$(date +%Y%m%d)
```

For Raspberry Pi, use one or more of the following:

- SD card image backup
- Filesystem backup
- Pi-hole Teleporter export
- Configuration backup of `/etc/pihole`
- Configuration backup of `/etc/unbound`

Examples of major changes:

- Pi-hole version update
- Debian release upgrade
- Unbound configuration change
- Network or VLAN change
- Router/DHCP DNS change
- Firewall policy change

## Security validation checklist

Run these checks after installation and after major changes.

Check listening ports:

```bash
sudo ss -tulpn
```

Check Pi-hole status:

```bash
pihole status
```

Check Unbound status:

```bash
sudo systemctl status unbound --no-pager
```

Check SSH root login policy:

```bash
sshd -T | grep permitrootlogin
```

Check Unbound localhost binding:

```bash
sudo ss -tulpn | grep 5335
```

Check DNS through Pi-hole:

```bash
dig pi-hole.net @<PIHOLE-IP>
```

Check Unbound directly:

```bash
dig pi-hole.net @127.0.0.1 -p 5335
```

Check DNSSEC:

```bash
dig fail01.dnssec.works @127.0.0.1 -p 5335
dig +ad dnssec.works @127.0.0.1 -p 5335
```

Review recent Pi-hole logs:

```bash
sudo pihole tail
```

Review Unbound logs:

```bash
journalctl -u unbound -n 100 --no-pager
```

Review SSH logs:

```bash
journalctl -u ssh -n 100 --no-pager
```

## Sources
- Pi-hole documentation: https://docs.pi-hole.net/
- Pi-hole Unbound guide: https://docs.pi-hole.net/guides/dns/unbound/
- Unbound documentation: https://unbound.docs.nlnetlabs.nl/
