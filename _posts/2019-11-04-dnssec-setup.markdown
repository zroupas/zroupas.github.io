---
layout: post
title: "Configure DNSSEC on CentOS7"
date: 2019-11-04 15:32:59
author: Zois Roupas
categories: linux
tags: dns dnssec
cover: "/assets/2019-11-04-dnssec/dnssec.png"
---

### Homelab - Configure DNSSEC on CentOS7

<hr>

For my next project I decided to configure DNSSEC to my local DNS which is documented in my previous [post][post]. Don't ask me why I chose to setup DNSSEC in a non public facing Authoritative Caching DNS, why we do anything in the first place right ? :relieved:

Honestly, it sounded like an interesting configuration and I was really curious on how this implementation is securing DNS Data. So let's start!

### Introduction

DNSSEC stands for Domain Name System Security Extension and strengthens DNS authentication by using digital signatures based on public key cryptography. In order to understand why DNSSEC is needed you must have a good idea of how DNS is working and you can find a basic introduction [here][post] or a very detailed one all around the internet.

DNSSEC was introduced due to the fact that when DNS was designed in the 80's there was not so much discussion going on over security. With DNSSEC enabled, it's the actual DNS data that are cryptographically signed by the zone owner and every DNS zone gets it's own pair of public/private key. The private key is used to sign the zone and it is kept secret unlike public key which is appended in the zone so that anyone can retrieve and validate the authenticity.

The most common example of why DNS security is nowadays more than needed is the "man in the middle attack" where an infected server, not with the flu of course but by poisoning ( like this is more helpful :astonished: ) , is sending us to a malicious website that looks like our web banking but it actually isn't. This is called Spoofing ( a.k.a DNS Cache Poisoning).

I won't go into more details on the different files and signatures introduced in DNSSEC like RRSIG, NSEC etc. because as I keep saying there are tons of great tutorials that those files are explained and I will go straight to documenting my own DNSSEC configuration.

### Configuration

Enough said about DNSSEC, time to roll up our sleaves and do the actual work!

We've already configured our own DNS server and in CentOS7 our zone files should be located in **/var/named/** which is the default path. This folder should have at least two files:

* FWD Zone: homelab.home.zone
* REV Zone: 0.0.10.in-addr.arpa

Let's now create a Zone Signing public/private key (ZSK) pair for both forward and reverse zone. Inside the default path located in **/var/named/** run:

{% highlight shell %}
$ dnssec-keygen -a NSEC3RSASHA1 -b 1024 -n ZONE homelab.home.zone
Generating key pair..................+++ .............+++
$ dnssec-keygen -a NSEC3RSASHA1 -b 1024 -n ZONE 0.0.10.in-addr.arpa
Generating key pair..................+++ .............+++

$ ls -la *.{key,private}
-rw-r--r-- 1 root root  606 Oct 20 Khomelab.home.+007+29081.key
-rw------- 1 root root 1779 Oct 20 Khomelab.home.+007+29081.private
-rw-r--r-- 1 root root  628 Oct 20 K0.0.10.in-addr.arpa.+007+17982.key
-rw------- 1 root root 1779 Oct 20 K0.0.10.in-addr.arpa.+007+17982.private
{% endhighlight %}

Let's move on by creating the Key Signing Key(KSK) pair,

{% highlight shell %}
$ dnssec-keygen -a NSEC3RSASHA1 -b 1024 -n ZONE -f KSK homelab.home.zone
Generating key pair..................+++ .............+++
$ dnssec-keygen -a NSEC3RSASHA1 -b 1024 -n ZONE -f KSK 0.0.10.in-addr.arpa
Generating key pair..................+++ .............+++

$ ls -la *.{key,private}
-rw-r--r-- 1 root root  605 Oct 20 Khomelab.home.+007+33125.key
-rw------- 1 root root 1779 Oct 20 Khomelab.home.+007+33125.private
-rw-r--r-- 1 root root  628 Oct 20 K0.0.10.in-addr.arpa.+007+45678.key
-rw------- 1 root root 1779 Oct 20 K0.0.10.in-addr.arpa.+007+45678.private
{% endhighlight %}

Our directory should now have 4 keys, 1 public/private ZSK pair and a KSK one. Now to force our zones use DNSSEC we have to add the public keys, which contains DNSKEY record, to both forward and reverse zone files. As we said earlier **only** the public keys should be appended and not the private ones:

{% highlight shell %}
$ cat Khomelab.home.+007+*.key >> homelab.home.zone
$ cat K0.0.10.in-addr.arpa.+007+*.key >> 0.0.10.in-addr.arpa
{% endhighlight %}

Open your zones and take a look to the keys added at the end! 

Finally let's sign the zones by using dnssec-signzone command,

{% highlight shell %}
# usage example: dnssec-signzone command **OPTIONS** **Zone Name**  **Zone File** **Private Key**
$ dnssec-signzone -t -g -o homelab.home /var/named/homelab.home.zone /var/named/Khomelab.home.+007+*.private
$ dnssec-signzone -t -g -o 0.0.10.in-addr.arpa /var/named/0.0.10.in-addr.arpa /var/named/K0.0.10.in-addr.arpa.+007+*.private
{% endhighlight %}

As a result of the above commands 2 more files are now present with .signed extension in our default path.

{% highlight shell %}
$ ls -la *.signed

-rw-r--r-- 1 root root 32702 Oct 20 0.0.10.in-addr.arpa.signed
-rw-r--r-- 1 root root 74748 Oct 20 homelab.home.zone.signed
{% endhighlight %}

### Master Configuration File

Now that we have our signed zones we need to enable DNSSEC in the master configuration file and use the newly ones instead of the old zone names. 

Open **/etc/named.conf** and add these three line in the options block:

{% highlight bash %}
		..

        dnssec-enable yes;
        dnssec-validation auto;
        dnssec-lookaside auto;

    ..
{% endhighlight %}

and in the zone section replace the old non signed ones with the .signed zones.

{% highlight bash %}
zone "homelab.home" IN {
       type master;
       file "homelab.home.zone.signed";
};

zone "0.0.10.in-addr.arpa" IN {
       type master;
       file "0.0.10.in-addr.arpa.signed";
};
{% endhighlight %}

In order to activate the new configuration we have to restart bind

{% highlight shell %}
$ systemctl restart named
{% endhighlight %}

### Validation

Now let's query our DNS and check the response. We will use the +rrcomments instead of +multi that was introduced in BIND 9.9.

{% highlight shell %}
$ dig +rrcomments homelab.home. DNSKEY @localhost
$ homelab.home.
{% endhighlight %}

If everything is configured correctly we should receive a reply which looks like:

{% highlight bash %}
; <<>> DiG 9.9.4-RedHat-9.9.4-51.el7_4.2 <<>> +rrcomments homelab.home. DNSKEY
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 29937
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;homelab.home.                   IN      DNSKEY

;; ANSWER SECTION:
homelab.home.            86400   IN      DNSKEY  257 3 7 [..] ; KSK; alg = NSEC3RSASHA1; key id = 33125
homelab.home.            86400   IN      DNSKEY  256 3 7 [..] ; ZSK; alg = NSEC3RSASHA1; key id = 29081

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Thu Oct 31 17:14:27 EET 2019
;; MSG SIZE  rcvd: 592
{% endhighlight %}

If for any reason DNSSEC isn't configured correctly then we will get a SERVFAIL return code status without any DNS data.

### (Semi)Automatic Zone Signing

So we need a way to do as less manual work as possible. In my previous local DNS configuration [post][post] I didn't document how to reload fwd or rev zones after adding new domain entries because I wanted to introduce the scripts that I've put together after searching around the internet. 

I'm using 2 simple scripts that will handle the serial number increment and the zone signing. We still have to edit the zones and manually add our entries but another script will follow in later posts that will also automate this task.

#### fwdzonesign.sh
{% highlight bash %}
#!/bin/bash
ZONEDIR="/var/named" #location of your zone files
ZONE=$1
ZONEFILE=$2
DNSSERVICE="named"
cd $ZONEDIR
SERIAL=`/usr/sbin/named-checkzone $ZONE $ZONEFILE | egrep -ho '[0-9]{10}'`
sed -i 's/'$SERIAL'/'$(($SERIAL+1))'/' $ZONEFILE
dnssec-signzone -t -g -o $1 $2 /var/named/Khomelab.home.+007+*.private
systemctl reload $DNSSERVICE
{% endhighlight %}

#### revzonesign.sh
{% highlight bash %}

#!/bin/bash
ZONEDIR="/var/named" #location of your zone files
ZONE=$1
ZONEFILE=$2
DNSSERVICE="named"
cd $ZONEDIR
SERIAL=`/usr/sbin/named-checkzone $ZONE $ZONEFILE | egrep -ho '[0-9]{10}'`
sed -i 's/'$SERIAL'/'$(($SERIAL+1))'/' $ZONEFILE
dnssec-signzone -t -g -o $1 $2 /var/named/K0.0.10.in-addr.arpa.+007+*.private
systemctl reload $DNSSERVICE
{% endhighlight %}

Those scripts need two parameters in order to run successfully,

{% highlight bash %}
# usage example: revzonesign.sh **ZONE** **ZONEFILE**
./fwdzonesign.sh homelab.home /var/named/homelab.home
./revzonesign.sh 0.0.10.in-addr.arpa /var/named/0.0.10.in-addr.arpa
{% endhighlight %}

I will edit my fwd zone and add an entry for my homelab NAS. I won't make any change in the serial number so as to check if the script will do the work for me..

{% highlight bash %}
$TTL    86400

@       IN      SOA     dns.homelab.home. root.homelab.home.    (
        2019092101
        3600
        900
        604800
        86400
)
;Name Server Information
@               IN      NS      dns.homelab.home.
dns             IN      A       10.0.0.1
; New line added here
nas             IN      A       10.0.0.2
{% endhighlight %}

Let's run our script and take a look at the output.

{% highlight shell %}
$ ./fwdzonesign.sh homelab.home /var/named/homelab.home
Verifying the zone using the following algorithms: NSEC3RSASHA1.
Zone fully signed:
Algorithm: NSEC3RSASHA1: KSKs: 1 active, 0 stand-by, 0 revoked
                         ZSKs: 1 active, 0 stand-by, 0 revoked
/var/named/homelab.home.zone.signed
Signatures generated:                            135
Signatures retained:                               0
Signatures dropped:                                0
Signatures successfully verified:                  0
Signatures unsuccessfully verified:                0
Signing time in seconds:                       0.070
Signatures per second:                      1901.569
Runtime in seconds:                            0.145
{% endhighlight %}
{% highlight shell %}
$ grep 20190921 /var/named/homelab.home.zone
$    2019092102
{% endhighlight %}
{% highlight shell %}
$ ping nas.homelab.home -c 3
PING nas.homelab.home (10.0.0.2) 56(84) bytes of data.
64 bytes from nas.homelab.home (10.0.0.2): icmp_seq=1 ttl=64 time=3.34 ms
64 bytes from nas.homelab.home (10.0.0.2): icmp_seq=2 ttl=64 time=3.64 ms
64 bytes from nas.homelab.home (10.0.0.2): icmp_seq=3 ttl=64 time=3.52 ms

--- nas.homelab.home ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 3.343/3.502/3.644/0.141 ms
{% endhighlight %}

Success! Our serial key is incremented by 1 , our forward zone is reloaded without any problems and our new entry is reachable. :man_dancing:

Let me point out here that in a real world scenario you will probably want to also increment year,day and month and not only the number of changes done in the file but again the format of the serial number is flexible and sysadmins treat it differently. 

Also in a production environement a cron should be configured to sign the zones every 3-4 days in order to prevent any attempt of recomputing the hashed information of a zone file (Zone Walking).

For example if you want to sign your zones every 4 days,then you have to add this lines in your crontab file
{% highlight bash %}
$ crontab -e
0 0 */4 * *  /usr/sbin/fwdzonesign.sh homelab.home /var/named/homelab.home
0 0 */4 * *  /usr/sbin/revzonesign.sh 0.0.10.in-addr.arpa /var/named/0.0.10.in-addr.arpa
{% endhighlight %}

### Final Words

The benefits of DNSSEC will not be able to have the complete effect unless all DNS resolvers adopt it and create a more secure internet expirience for each and everyone.

Until my next post and as Dr Wallace Breen says in Half-Life 2.. Be wise. Be safe. Be aware!

[post]: https://zroupas.github.io/linux/2019/09/22/local-dns.html
