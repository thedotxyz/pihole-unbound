# Setup Pi-hole and Unbound on a Raspberry Pi

https://docs.pi-hole.net/guides/dns/unbound/ </br>
DNS Sec: https://www.icann.org/resources/pages/dnssec-what-is-it-why-important-2019-03-05-en

<h2>Prerequisitestes</h2>


<h2>Pre configuration</h2>
test

<h3>Pi-hole</h3>
First install Pi-hole, using:

```console
curl -sSL https://install.pi-hole.net | bash
```
Choose default options untill you can choose an upstream DNS provider. For now choose cloudflare (1.1.1.1) als Upstream DNS provider. We will change this to Unbound later on in this instruction. After this keep choosing the default options until you see a summary and the default admin password (which we will change).

Back at the command prompt, change the Pi-hole default password by using:

```console
pihole -a -p
```


<h2>Unbound</h2>
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

Restart the Unbound server and run the dig command to test DNS resolution. You should see the status as “NOERROR” with an IP address for the pi-hole.net server.

```console
sudo service unbound restart
sudo service unbound status
dig pi-hole.net @127.0.0.1 -p 5335
```

The final test is to ensure that DNSSEC is working properly. First, if you’re interested in learning what DNSSEC is, this is a great explanation. There are two commands that you can run to ensure that DNSSEC is working properly.

```console
dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5335
```
This command should return <code>SERVFAIL</code> with <code>NO IP</code> address.

```console
dig sigok.verteiltesysteme.net @127.0.0.1 -p 5335
```

This command should return <code>NOERROR WITH</code> an IP address. If both are returned properly, DNSSEC is properly working. 




<h2>Post configuration</h2>

```console

```
