---
layout: post
title: "Create your own local SSL Certificate Authority"
date: 2019-12-13 15:32:59
author: Zois Roupas
categories: linux
tags: linux ca ssl openssl
cover: "/assets/2019-12-13-ca/ca.png"
---

### Homelab - Create your own local SSL Certificate Authority

<hr>

Now that I had my own Ovirt infrastructure and local DNS (DNSSEC enabled in case you [forgot][post]) server I was finally able to get rid of the host file maintaining pain and start accessing different URLs just by adding them in my local DNS zone file. So instead of the uggly old URL **https://10.0.0.3/ovirt-engine/** that i was using to access my Ovirt Dashboard, I could finally access it by typing its dns record, **https://ovirt.homelab.home**, instead of the IP.

But the pain never stops! 

I couldn't get over the fact that each time that I wanted to access a newly created HTTPS service the browser would inform me about the [Plagues of Egypt][Plagues of Egypt] that would be casted upon me in case I decided to proceed and connect to the desired endpoint. It was then that Bruce Lee's most famous quote hit me! **"Be the Water"!** And before start wondering how this is connected to my realization, let me explain myself!

In order to resolve this I would need an SSL certificate from an official Authority but this would mean money and let's be honest, my projects didn't have any potentials on conquering the world!(yet) 

Next step was to think about configuring Let's Encypt which is awesome and totally free! But at that point there was no wildcard support from their side and didn't want to set it up for each and every server ,so I decided to be Water! To be my own local SSL Certificate Authority and issue as many different certificates as I wanted! Thank you Bruce!

### Introduction

A **certificate authority (CA)** is actually an organization that signs digital certificates. Those digital documents bind the identity that is requesting the certificate with cryptographic keys, securing the validity of the entity. Such organizations are internationally trusted and their Root Certificates are pre-installed in all browsers and devices. 

We could get into more details in regards of the different CA hieraches and Subordinate (Sub) CA's but I will limit this post to the basic configuration that anyone needs to understand in order to create his own Root Authority and Self Signed Certificates!

### How SSL Issuing actually Works

In a real life production scenario the procedure that someone would follow in order to receive an SSL certificate would be to first create a **Certificate Signing Request (CSR)**. Generating a CSR means that a pair of keys is created, a public key which contains information regarding the organization that has applied for an SSL certificate  and a private one which is a unique cryptographic key related to the corresponding CSR and should never be shared with anyone outside your secured environment.<br/><strong><em>Attention:</em></strong> If the private key is either lost or compromised the certificate will no longer work or malicious users could potentially read your encrypted communications and put your organization at risk.

Last step after CSR generation would be to sent the public key to the official authority which would then give back a certificate which is signed by their root private key and can be installed to any of your servers.

### Creating our local Root CA
There are different ways to become a **Root CA**, if you are a GUI person there is an amazing free tool called **TinyCA** which has a ton of features and if you are a terminal enthusiast you are going to use two simple commands. 

First you need to create the Root's private key,
{% highlight shell %}
$ openssl genrsa -des3 -out root.key 2048
{% endhighlight %}
{% highlight shell %}
Generating RSA private key, 2048 bit long modulus (2 primes)
.................................................+++++
...................................................................................+++++
e is 65537 (0x010001)
Enter pass phrase for root.key:
Verifying - Enter pass phrase for root.key:
{% endhighlight %}

It is highly recommended that you provide a passphrase and secure your private key! <br/>Take a look at the content of the created private key,

{% highlight shell %}
$ cat root.key 
{% endhighlight %}
{% highlight shell %}
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,1D2E481705C5E3EE

Xibki05Ls9sgIgpmRdQR9IJweMq7S0RClYNCluYz4yopFUsCSSjinkkR+0PVBcTf
FCIOPOqEflScu6uqn71Mxq9EUrZTXecFgH+WOHxdC3n/TJDGIaiX0+33PpaEYkMu
UgXQi/vWBkmvAtowUtl9jIoFBd1szt/5qYsCN4XXkBOUACmEq7nZVYoyu2ieu1j9
K5OnWuLGwO+/zfYEu7Xlc0cTDR1XuEElngHE8fJ4cPQCfOcvT8lSwupl5DCGsUCe
PPcxthbwIUcm9BWYVeMsiys+Vb0P88JMLV6dVkF+T/HbdXrsM4BcyoqNpeEP64o0
eKvyBpYUDTWIOmyax9AX2SEFenqp18kmSnkx6XVAQm5OyHhhLK2VVWezlKB8D4a2
2KVudbxo5Z7jwu6h8cFgvCjFVBmDWYStd9YGsbJSGL7u0fK1pcx5oMuYpzbbMaP7
lPL250m2aRG8ULGWyGOdgQSzf7yY1ZfuoM0YaLzSSQ4IoTt+rYgsfpL+AN9tHM+N
vUDnMnFyFxp7KoXcUgpXJt1Ao1U9F3MLnyp13VSpxNVXY0oZGQLOxTcbjhI804uH
YGI84XIucy4C3sjkRtAclphwhuTgFbx3ubsFon6eDdj7UsR3njDDXkvmqyF/XK2Q
xrgE+JN7hdVYnTEkWr/1PgZMmoCQU238Mk1Ltf+F5Of+0IZh+sxO0ZwOaJL78QIf
4pFrd9B4nhcJkRHzta9U9icCu+JFso30ednEUpuxalBCDHRAofPOWMzjzAGVz/Mk
LJc2WnnvGHjp06nvrh317V42e+ZrQnOsWAEQVtInpEmoL0sRnxQks6ozHCXRstct
IfW5cswfjmtcMI9HBvrCA3MRc2qnG9ebteYt9PfoIdMSO7wIxZhhGjqo1EmmuPDa
U5MOor1tHTrSJykrMnTJ5HlrbXu7Dxk7zxD3S9iyIhqf5zQ9bZqZiYeoPBcrtMhU
DBKnMzAW916eqLaz6RqYptgrKn0JquYGc1aW/g8dL5d+654ras1YhYaKvsRCIDoS
1dXirYy7sIoWToPrgEW92EAvdeIwDjpkncgiBOonhgmqytvOsSzAv6Eoyq3Y5e1y
7BV4QqGrDditp6N5ltF1RJ/X8H8MxHmkEqbuq83WTAux9S/VwuAYEdxMZv9IRw2M
myQebUxDm2yj4uEGVNY+J6ikIVT/NBITcMqsk4dXmFNV6oFl08h5IwpVPqY4xOWD
VY8X2mh9xT+PkusAGzB+BVQYEudDBkSMKfKO4gd8DokY9VHMwj5XjG5K4SvF2lWz
0LkUtrEjiGjRdUHAWTDiGyxVCpmr66hS+qRjaMsVhzSRGGF6vdchkh0cAqTutPsR
7/60xg8ra/rtHZT+6rari7+cd/DjENj39MuYFWLTyzaM39VKsJQbABv8vl6y1Gb4
VCf+ANBQFw81cC0J1D5my5u8vFPxecE6HccPCBo/1Wt9gHknitH8jFNf54YBg27v
jwOjHYYBc+FcR8jqvsBfcUKzRRDHDmm8k7wfZECNxZQrpMnYJX0PA0eXQ+cbVwpr
9Xj/tYobj85G+wkEqpJwbHpYLzKaEGJrx1TRXjNrN0EyOMPeK9GhzRcClG7Sfcev
-----END RSA PRIVATE KEY-----
{% endhighlight %}

Let's now generate the Root Certificate. You will be prompted for the passphrase of your private key from the previous step and a bunch of questions will come up. As an advise the only question that makes sense at this step is the **Common Name (CN)** as this name will help you recognize your authority in a list of others (for example when imported in a browser and you need to confirm that your Root Authority is actully present , you will thank me later!).

Our Root CA will be valid for the next 20 years but feel free to adjust to your needs,

{% highlight shell %}
$ openssl req -x509 -new -nodes -key root.key -sha256 -days 7200 -out root.pem
{% endhighlight %}
{% highlight shell %}
Enter pass phrase for root.key:
Can't load /home/user/.rnd into RNG
140536510013888:error:2406F079:random number generator:RAND_load_file:Cannot open file:../crypto/rand/randfile.c:88:Filename=/home/user/.rnd
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:GR
State or Province Name (full name) [Some-State]:Attiki
Locality Name (eg, city) []:Athens
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Homelab
Organizational Unit Name (eg, section) []:IT
Common Name (e.g. server FQDN or YOUR name) []: Private Homelab Authority                                         
Email Address []:
{% endhighlight %}

You can check the contents of your Authority's certificate by issuing,

{% highlight shell %}
$ openssl x509 -text -noout -in  root.pem | head -15
{% endhighlight %}
{% highlight shell %}
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            36:37:0a:aa:ef:ad:ed:44:e1:f1:be:bf:e6:a4:98:69:96:1e:5a:6b
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = GR, ST = Attiki, L = Athens, O = Homelab, OU = IT, CN = Private Homelab Authority
        Validity
            Not Before: Dec  4 14:03:03 2019 GMT
            Not After : Aug 21 14:03:03 2039 GMT
        Subject: C = GR, ST = Attiki, L = Athens, O = Homelab, OU = IT, CN = Private Homelab Authority
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
{% endhighlight %}

Also to make sure that this is a CA you can also issue this command,
{% highlight shell %}
$ openssl x509 -text -noout -in  root.pem  | grep CA:
{% endhighlight %}
{% highlight shell %}
CA:TRUE
{% endhighlight %}

Success!!

At this point you should now have two files:

{% highlight shell %}
$ ll
total 16
drwxrwxr-x  2 user  user  4096 Νοε  28 19:50 ./
drwxrwxrwt 16 root root 4096 Νοε  28 19:49 ../
-rw-------  1 root  root  1751 Νοε  28 19:22 root.key
-rw-rw-r--  1 root  root  1367 Νοε  28 19:50 root.pem
{% endhighlight %}

You can now go grab a beer because you are now a CA and cash is gonna flow, baby! Hm.. the second part of course isn't actually true and you will not be able to sell certificates but at least you got a beer!

### Import your Root CA to any browser or keystore

In order to act as an actual local CA you have to import the root certificate you created on all of the devices that you own. You can either add it to OS's keycain/Certificate Store or directly to your browser generally by going to **Preferences** -> **Certificates** -> **View Certificates** -> **Authorities** Tab and Import you pem file. 

When finished you should see your Authority in the list! Aren't you glad that you picked a name that can be spotted easily?

<a href="/assets/2019-12-13-ca/trusted.png" data-lightbox="ca" data-title="Check out your Authority">
  <img src="/assets/2019-12-13-ca/trusted-small.png" title="ca">
</a>

So congratulations but no more beer for you.. let's continue!

### Create Self Signed Wildcard Certificate

Now we have to create the actual certificate that is going to be installed in our homelab servers. The only difference from a production SSL certificate issuing procedure is that instead of sending the csr that we are going to create in the next steps to **Comodo** or **Verisign**, is that we will bypass them and sign it with our own CA! (who needs them now, right? :P )

There are different certificate types (**OV, EV, Wildcard** etc.) and authority hierarchies (**Root, Intermediate, Sub CA** etc.) and for our case we only created a Root Authority and will continue by issuing a **Wildcard certificate** so as to have one SSL certificate for all of our homelab.home internal domains.<br/>You can find a lot of great articles if you need more information on the different types that will help you purchase the correct type for your production environment (and I would say that you already stopped reading the rest of the post and started googling around) or your homelab. 

In order for you to understand why i chose a wildcard certificate and what is the difference with a single one, I have to provide some basic information.
A wildcard certificate can be applied to a domain and all its subdomains. Instead of having a separate certificate for each of your domains (**mail.homlab.home,ovirt.homelab.home** etc.) you can create or purchase a wildcard certificate which will protect all of you domains and save you money. Of course there are always advantages and disadvantages in the wilcard village but I will skip this discussion for now.

We can now create our self-signed wildcard certificate by creating our csr and private key. Do not be confused with the csr and key that we created in the previous step, this was Root's files and in real life we wouldn't have access to those! <br/>Instead of typing again all the information needed by a csr as we did in the Root creation we will use a config file. We will mostly do that though because we have to define a Subject Alternative Name (SAN) extension. As explained [here][chromestatus] 

>>*RFC 2818 describes two methods to match a domain name against a certificate - using the available names within the subjectAlternativeName extension, or, in the*
>>*absence of a SAN extension, falling back to the commonName.*

at some point when Chrome and Firefox ignorred CN the user would receive privacy errors because **subjectAlternativeName** would be empty!
There is a great free tool which is called testssl which checks a server's service on any port for the support of TLS/SSL ciphers and can be downloaded from [here][here].

So let's create our private key,

{% highlight shell %}
$ openssl genrsa -out wildcard.homelab.home.key 2048
{% endhighlight %}
{% highlight shell %}
Generating RSA private key, 2048 bit long modulus (2 primes)
......+++++
..+++++
e is 65537 (0x010001)
{% endhighlight %}

Then we will create the certificate request by using the config file found below, do you see the alt_names section? This is the **SAN** option we where talking about before,

{% highlight shell %}
$ cat opensslsan.cnf
{% endhighlight %}
{% highlight shell %}
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no
[req_distinguished_name]
C = GR
ST = Attica
L = Athens
O = Wildcard Homelab Inc
OU = IT
CN = *.homelab.home
[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = *.homelab.home
{% endhighlight %}

{% highlight shell %}
openssl req -new -out wildcard.homelab.home.csr \
-key wildcard.homelab.home.key \
-config opensslsan.cnf
{% endhighlight %}

After the previous command 2 more files are added to our folder, the request (csr) and private key (key) for our self signed certificate.

{% highlight shell %}
$ ls -la
total 36
drwxrwxr-x 2 user user 4096 Dec   5 16:39 ./
drwxrwxr-x 3 user user 4096 Dec   4 16:01 ../
-rw-rw-r-- 1 user user  341 Dec   5 16:34 opensslsan.cnf
-rw------- 1 user user 1751 Dec   4 16:02 root.key
-rw-rw-r-- 1 user user 1367 Dec   4 16:03 root.pem
-rw-rw-r-- 1 user user 1127 Dec   5 16:39 wildcard.homelab.home.csr
-rw------- 1 user user 1675 Dec   5 16:38 wildcard.homelab.home.key
{% endhighlight %}

Now instead of sending the csr to a legitimate certificate authority so as to sign it with its private key, we will sign it with our own!I hope you remember the password that you have picked for your Root CA's private key because the next command will ask you to enter it! If not don't worry, just delete everything and restart the process and this time keep a note! :) 

{% highlight shell %}
openssl x509 -req -in wildcard.homelab.home.csr \
-CA root.pem \
-CAkey root.key \
-CAcreateserial \
-out wildcard.homelab.home.crt \
-days 7200 \
-sha256 \
-extensions v3_req \
-extfile opensslsan.cnf
{% endhighlight %}
{% highlight shell %}
Signature ok
subject=C = GR, ST = Attica, L = Athens, O = Wildcard Homelab Inc, OU = IT, CN = *.homelab.home
Getting CA Private Key
Enter pass phrase for root.key:
{% endhighlight %}

You should now have one more file added to the list of the wildcard* related ones which is the issued certificate!

{% highlight shell %}
$ ls -la
total 36
drwxrwxr-x 2 user user 4096 Dec   5 16:39 ./
drwxrwxr-x 3 user user 4096 Dec   4 16:01 ../
-rw-rw-r-- 1 user user  341 Dec   5 16:34 opensslsan.cnf
-rw------- 1 user user 1751 Dec   4 16:02 root.key
-rw-rw-r-- 1 user user 1367 Dec   4 16:03 root.pem
-rw-rw-r-- 1 user user 1342 Dec   5 16:39 wildcard.homelab.home.crt
-rw-rw-r-- 1 user user 1127 Dec   5 16:39 wildcard.homelab.home.csr
-rw------- 1 user user 1675 Dec   5 16:38 wildcard.homelab.home.key
{% endhighlight %}

Let's verify that the certificate is correct and the chain is trusted,

{% highlight shell %}
$ openssl verify -CAfile root.pem wildcard.homelab.home.crt
{% endhighlight %}
{% highlight shell %}
wildcard.homelab.home.crt: OK
{% endhighlight %}

Nice! Let's take a look at our Wildcard Certificate,

{% highlight shell %}
$ openssl x509 -text -noout -in  wildcard.homelab.home.crt  | head -15
{% endhighlight %}
{% highlight shell %}
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            46:c1:33:39:57:8b:87:f1:12:ba:90:1d:8d:81:22:ab:c6:ff:99:af
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = GR, ST = Attiki, L = Athens, O = Homelab, OU = IT, CN = Private Homelab Authority
        Validity
            Not Before: Dec  5 14:39:41 2019 GMT
            Not After : Aug 22 14:39:41 2039 GMT
        Subject: C = GR, ST = Attica, L = Athens, O = Wildcard Homelab Inc, OU = IT, CN = *.homelab.home
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
{% endhighlight %}

{% highlight shell %}
$ openssl x509 -text -noout -in  wildcard.homelab.home.crt | grep DNS
{% endhighlight %}
{% highlight shell %}
DNS:*.homelab.home 
{% endhighlight %}

Do you spot the differences from the Root certificate? Our certificate is now issued and valid for the next 20 years (how lazy am I?).

All those files including wildcard.homelab.home.crt **(certificate)**, wildcard.homelab.home.key **(private key)** and probably the root.pem **(CA certificate)** will be used in one way or another to configure SSL in any of your web servers.

### Final Words

It is great to have my personal CA and be able to issue any certificate on my own. I didn't configure an Intermediate CA just for simplicity but this would be a great side project for you to understand the hierarchy and how all the different levels are connected to each other in order to create the necessary **[Chain of Trust][Chain of Trust]!**

As a final step I could wrap this in a bash script or an ansible playbook and automate the process but I won't be using any other certificate other than the wildcard one, at least for now! But maybe I can create a post about this in the near future! (Dear FutureMe..)

In my next post I will try to install the newly created Wildcard SSL to my **Ovirt Engine** which was the initial spark that started the idea of hosting my own CA!

Until my next post and as Dr Wallace Breen says in Half-Life 2.. Be wise. Be safe. Be aware!

[post]: https://zroupas.github.io/linux/2019/11/04/dnssec-setup.html
[Plagues of Egypt]: https://en.wikipedia.org/wiki/Plagues_of_Egypt
[chromestatus]: https://www.chromestatus.com/feature/4981025180483584
[here]: https://testssl.sh/
[Chain of Trust]: https://www.thesslstore.com/knowledgebase/ssl-support/explaining-the-chain-of-trust/