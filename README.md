# Setup Pi-hole and Unbound on a Raspberry Pi
Lightweight setup for running Pi-hole with Unbound as a local recursive DNS resolver.

This setup is designed for:
- Proxmox VE with a Debian LXC container
- Debian 12 Bookworm
- Debian 13 Trixie
- Raspberry Pi OS / Debian-based systems
  
PiHole Unbound: https://docs.pi-hole.net/guides/dns/unbound/ </br>
DNS Sec: https://www.icann.org/resources/pages/dnssec-what-is-it-why-important-2019-03-05-en

## Target Architecture
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
## Recommended platform
For Proxmox, use:

```text
Type:        Unprivileged LXC
OS:          Debian 12 or Debian 13 standard template
CPU:         1 vCPU
RAM:         512 MB minimum, 1 GB recommended
Disk:        4 GB minimum, 8 GB recommended
Network:     Bridged network with static IP or DHCP reservation
```
Do not use Docker inside LXC for this setup unless you have a specific operational reason. A native Debian LXC is simpler, smaller and easier to maintain.

## Prerequisitestes
## Prerequisites

This guide assumes one of the following base systems:

- Debian 13 Trixie
- Debian 12 Bookworm
- Raspberry Pi OS based on Debian

For Proxmox VE, the recommended deployment model is:

```text
Type:        Unprivileged LXC
OS:          Debian 13 standard template
CPU:         1 vCPU
RAM:         512 MB minimum, 1 GB recommended
Disk:        4 GB minimum, 8 GB recommended
Network:     Bridged network with static IP or DHCP reservation
```

Before installing, make sure:

- The host or container has a static IP address or DHCP reservation.
- No other service is already listening on port 53.
- Outbound DNS traffic to the internet on TCP/UDP port 53 is allowed.
- The system can reach the DNS root servers directly.
- You have root access or sudo permissions.
- Pi-hole DHCP is disabled unless you explicitly want Pi-hole to act as DHCP server.
- The Pi-hole admin interface is not exposed directly to the internet.

Install basic packages:

```bash
sudo apt update
sudo apt install -y curl wget dnsutils unbound ca-certificates
```
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
  --rootfs ssd4tb:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp,type=veth \
  --unprivileged 1 \
  --features nesting=1 \
  --onboot 1 \
  --start 1
```

Enter the container:

```bash
pct enter 110
```

Update the container and install basic packages:

```bash
apt update
apt upgrade -y
apt install -y curl wget dnsutils sudo ca-certificates unbound openssh-server
```
### Enable SSH access

Install and enable SSH inside the container.

This is recommended because the interactive Pi-hole installer may not behave correctly when started through `pct enter`.

```bash
apt install -y openssh-server
systemctl enable --now ssh
```

Validate that SSH is running:

```bash
systemctl status ssh --no-pager
```

Exit the container:

```bash
exit
```

After configuring the static IP address below, continue the Pi-hole installation over SSH instead of using `pct enter`.

Example:

```bash
ssh root@192.168.1.30
```
Adjust the IP address to match your container.
For DNS infrastructure, use a stable IP address.

Either configure a DHCP reservation on your router/firewall, or configure a static IP address in Proxmox.

Example:

```bash
pct stop 110
pct set 110 --net0 name=eth0,bridge=vmbr0,ip=192.168.1.30/24,gw=192.168.1.1,type=veth
pct start 110
```

Adjust the IP address and gateway to match your own network.

Validate basic connectivity from inside the container:

```bash
pct enter 110

ip a
ip route
ping -c 3 1.1.1.1
ping -c 3 debian.org
```

After this, continue with the Pi-hole installation.

## Pre-flight checks

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
After this, continue with the Pi-hole installation over SSH.

## Install Pi-hole
First install Pi-hole, using the official Pi-hole installer:

```console
ssh root@192.168.1.30
curl -sSL https://install.pi-hole.net | bash
```
During installation:
- Choose the network interface used by the container/host.
- Confirm or configure the static IP address.
- Select any temporary upstream DNS provider (for instance 'Cloudflare' (1.1.1.1)). This will be replaced later by Unbound.
- Enable the web interface.
- Enable query logging according to your own privacy requirements.

Back at the command prompt, change the Pi-hole default password by using:

```console
pihole -a -p
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
pihole restartdns
```

## Verify Pi-hole is using Unbound

From a client machine, query Pi-hole:

```bash
dig pi-hole.net @<PIHOLE-IP>
```

On the Pi-hole host/container, verify queries in the live log:

```bash
pihole tail
```

You should see client queries arriving at Pi-hole. Pi-hole should forward allowed domains to Unbound on `127.0.0.1#5335`.



=========================
## Install Unbound
Run the commands below to install Unbound and attain the root.hints file needed.</br>

```console
sudo apt install unbound -y
wget https://www.internic.net/domain/named.root -qO- | sudo tee /var/lib/unbound/root.hints
```
Create a file that will force Unbound to only listen for queries from Pi-hole. There are a few other benefits that can be found on the official Unbound page.

```console
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
```

Paste the contents below into the file we just created and save. Source: https://docs.pi-hole.net/guides/dns/unbound/.

```conf
server:
    # If no logfile is specified, syslog is used
    # logfile: "/var/log/unbound/unbound.log"
    verbosity: 0

    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes

    # May be set to yes if you have IPv6 connectivity
    do-ip6: no

    # You want to leave this to no unless you have *native* IPv6. With 6to4 and
    # Terredo tunnels your web browser should favor IPv4 for the same reasons
    prefer-ip6: no

    # Use this only when you downloaded the list of primary root servers!
    # If you use the default dns-root-data package, unbound will find it automatically
    #root-hints: "/var/lib/unbound/root.hints"

    # Trust glue only if it is within the server's authority
    harden-glue: yes

    # Require DNSSEC data for trust-anchored zones, if such data is absent, the zone becomes BOGUS
    harden-dnssec-stripped: yes

    # Don't use Capitalization randomization as it known to cause DNSSEC issues sometimes
    # see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
    use-caps-for-id: no

    # Reduce EDNS reassembly buffer size.
    # IP fragmentation is unreliable on the Internet today, and can cause
    # transmission failures when large DNS messages are sent via UDP. Even
    # when fragmentation does work, it may not be secure; it is theoretically
    # possible to spoof parts of a fragmented DNS message, without easy
    # detection at the receiving end. Recently, there was an excellent study
    # >>> Defragmenting DNS - Determining the optimal maximum UDP response size for DNS <<<
    # by Axel Koolhaas, and Tjeerd Slokker (https://indico.dns-oarc.net/event/36/contributions/776/)
    # in collaboration with NLnet Labs explored DNS using real world data from the
    # the RIPE Atlas probes and the researchers suggested different values for
    # IPv4 and IPv6 and in different scenarios. They advise that servers should
    # be configured to limit DNS messages sent over UDP to a size that will not
    # trigger fragmentation on typical network links. DNS servers can switch
    # from UDP to TCP when a DNS response is too big to fit in this limited
    # buffer size. This value has also been suggested in DNS Flag Day 2020.
    edns-buffer-size: 1232

    # Perform prefetching of close to expired message cache entries
    # This only applies to domains that have been frequently queried
    prefetch: yes

    # One thread should be sufficient, can be increased on beefy machines. In reality for most users running on small networks or on a single machine, it should be unnecessary to seek performance enhancement by increasing num-threads above 1.
    num-threads: 1

    # Ensure kernel buffer is large enough to not lose messages in traffic spikes
    so-rcvbuf: 1m

    # Ensure privacy of local IP ranges
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10
```

Ctrl+O en CTRL+X

Restart the Unbound server and run the dig command to test DNS resolution. You should see the status as “NOERROR” with an IP address for the pi-hole.net server.

```console
sudo service unbound restart
sudo service unbound status
dig pi-hole.net @127.0.0.1 -p 5335
```

The final test is to ensure that DNSSEC is working properly. First, if you’re interested in learning what DNSSEC is, this is a great explanation. There are two commands that you can run to ensure that DNSSEC is working properly.

```console
dig fail01.dnssec.works @127.0.0.1 -p 5335
```
This command should return <code>SERVFAIL</code> with <code>NO IP</code> address.

```console
dig dnssec.works @127.0.0.1 -p 5335
```

This command should return <code>NOERROR WITH</code> an IP address.</br>
 If both are returned properly, DNSSEC is properly working. 

<h2>Point Pihole to Unbound</h2>
XX

<h2>Configuring Blocklist</h2>
XX

https://avoidthehack.com/best-pihole-blocklists
https://firebog.net/


<h2>Populate Pi-hole</h2>
XX

<h2>Test Add-blocking</h2>

https://d3ward.github.io/toolz/adblock.html

<h2>Post configuration</h2>
Updating Pi-hole is easy. This can be done by:

```console
sudo pihole -up
```

Set local firewall</br>
Schedule Cron Job for updating

## Sources
- Pi-hole documentation: https://docs.pi-hole.net/
- Pi-hole Unbound guide: https://docs.pi-hole.net/guides/dns/unbound/
- InterNIC root hints: https://www.internic.net/domain/named.root
- Unbound documentation: https://unbound.docs.nlnetlabs.nl/
