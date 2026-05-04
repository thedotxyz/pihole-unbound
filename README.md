# Setup Pi-hole and Unbound on a Raspberry Pi
Lightweight setup for running Pi-hole with Unbound as a local recursive DNS resolver.

This setup is designed for:

- Proxmox VE with a Debian LXC container
- Debian 12 Bookworm
- Debian 13 Trixie
- Raspberry Pi OS / Debian-based systems
  
PiHole Unbound: https://docs.pi-hole.net/guides/dns/unbound/ </br>
DNS Sec: https://www.icann.org/resources/pages/dnssec-what-is-it-why-important-2019-03-05-en

<h2>Target Architecture</h2>
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
<h2>Recommended platform</h2>
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

<h2>Prerequisitestes</h2>
Before installing, make sure:

- The host has a static IP address or DHCP reservation.
- No other service is already listening on port 53.
- Outbound DNS traffic to the internet on TCP/UDP port 53 is allowed.
- The system can reach the DNS root servers.
- You have root access or sudo permissions.
- Pi-hole DHCP is disabled unless you explicitly want Pi-hole to act as DHCP server.

Install basic packages:

```bash
sudo apt update
sudo apt install -y curl wget dnsutils unbound ca-certificates
```

<h2>Pre-flight checks/h2>
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

If these tests fail, your ISP, router or firewall may be blocking or intercepting direct DNS traffic. Fix that before continuing.

<h3>Install Pi-hole</h3>
First install Pi-hole, using the official Pi-hole installer:

```console
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

<h2>Install Unbound</h2>
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
