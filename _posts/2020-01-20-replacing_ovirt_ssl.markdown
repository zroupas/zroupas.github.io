---
layout: post
title: "Replace oVirt Engine SSL/TLS Certificate with a third-party CA certificate"
date: 2020-01-20 15:32:59
author: Zois Roupas
categories: linux
tags: ovirt ssl tls
cover: "/assets/2020-01-20-ovirt_ssl/ovirt.png"
---

### Homelab - Replace oVirt Engine SSL/TLS Certificate with a third-party CA certificate

<hr>

As promised in my previous post, now that we had our own CA and self-signed wildcard certificate it was time to get rid of the uggly https security warning by configuring oVirt Engine to use the newly issued certificate instead of the one that is preconfigured.

Even though there is a great official tutorial found in the oVirt [documentation][tutorial], which I followed and will be our base, I found it quite challenging to replace the default SSL and especialy when it came to the exact files and extentions that I needed to create and import. Especially the part of the official tutorial where it uses two files that seem the same , to me at least when I read the instructions, but they are not and I'm talking about **YOUR-3RD-PARTY-CERT.pem** and **3rd-party-ca-cert.pem**! (to be honest now that I'm writing this post the difference is obvious but it is late for me to back out! :P )

So after some tries I managed to gather the needed files and this is the reason that I wanted to create a specific post and help anyone out there that wants to do the same or came across the same problems. SSL trust chains can drive you crazy!

#### Prerequisites

Due to the fact that the official tutorial is using a **.p12** extention, a common extention provided by most Certificate Authorities, I will follow their example and document how to create such an extention and then export the appropriate files. We could directly use are crt and key created in my previous [post][ca] but by creating the .p12 we will get the chance to provide some openssl commands that are maybe helpful to other cases. <br/>Let's create our p12 file!
{%highlight shell %}
$ openssl pkcs12 -export -out /tmp/apache.p12 \
-inkey wildcard.homelab.home.key \
-in wildcard.homelab.home.crt
{% endhighlight %}
{%highlight shell %}
Enter Export Password:
Verifying - Enter Export Password:
{% endhighlight %}
Provide a password, I didn't used one and left it blank for simplicity, and keep a note as this will be asked in the export procedure. We are giving this **apache.p12** name to the exported file as this will be used to replace the old Internal CA and the name is essential and should not be changed.

_**Note:** You may wonder why I didn't use the -certfile flag when creating the p12 file but I guessed that due to the fact that root.pem will also be used in a later step, the addition of the root pem in the p12 file wasn't important .In other cases, for example in a IIS certificate installation, I would also add the root CA as it would be needed in order to have a correct Chain of Trust._

In order to correctly replace the default SSL we need some files:

- A third-party CA certificate, it should be in a **pem** format. If an official Certificate Authority issued the certificate then you have to, and this is important in order to successfully replace the default SSL, keep in mind that chain’s order is critical and must contain everything from the last intermediate to the root certificate. Those different certificates (bundles) can be found on the official site of each of the Authoritie's website and with a simple concatenate (cat) you can create a single .pem file that contains every available CA certificate from bottom to top. In our case there is no Intermediate CA so only the Root certificate is present in the **root.pem** file created in this [post][ca]. <br/>Take a closer look to the root certificate in order to better understand what I mean when I'm talking about trust chain importance,
{%highlight shell %}
$ cat root.pem
{% endhighlight %}
{%highlight shell %}
-----BEGIN CERTIFICATE-----
MIIDxTCCAq2gAwIBAgIUNjcKqu+t7UTh8b6/5qSYaZYeWmswDQYJKoZIhvcNAQEL
BQAwcjELMAkGA1UEBhMCR1IxDzANBgNVBAgMBkF0dGlraTEPMA0GA1UEBwwGQXRo
ZW5zMRAwDgYDVQQKDAdIb21lbGFiMQswCQYDVQQLDAJJVDEiMCAGA1UEAwwZUHJp
dmF0ZSBIb21lbGFiIEF1dGhvcml0eTAeFw0xOTEyMDQxNDAzMDNaFw0zOTA4MjEx
NDAzMDNaMHIxCzAJBgNVBAYTAkdSMQ8wDQYDVQQIDAZBdHRpa2kxDzANBgNVBAcM
BkF0aGVuczEQMA4GA1UECgwHSG9tZWxhYjELMAkGA1UECwwCSVQxIjAgBgNVBAMM
GVByaXZhdGUgSG9tZWxhYiBBdXRob3JpdHkwggEiMA0GCSqGSIb3DQEBAQUAA4IB
DwAwggEKAoIBAQDPHX2MyDpZbCgQM68hzWJEj7ZGE0T7b6S0OpCSIrp9Kt/wufn9
nDc6UO9EDvbE780EL84FYQJSfCIh4aC06aYMXF9yAnphoK2vOhSFPWMDh/3qzlgi
6r3/TGG52GD3YtOauOnnEnUKZ/PA9f5umpr86BRZvngTMpkrGU11VRE/W/Byhm/K
hMJaSTEb8H1pmrDjplsaWSsXMh1UmRydwrYRPu4ajFFrWzx0Gd8Toz1RBP09jtxz
ijeXui6c7z/MiOUrs2MF+UJs7Fb53dIEN5wh0rQQjW53+1ElDno8IWR0ddcBqFJa
IjQB7NSWDVB+V+Qry0f1/YsKeMBIqJbLY+S9AgMBAAGjUzBRMB0GA1UdDgQWBBSL
F4gEGd8+FAJYXR4WTI5kseO6yjAfBgNVHSMEGDAWgBSLF4gEGd8+FAJYXR4WTI5k
seO6yjAPBgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3DQEBCwUAA4IBAQBSpY9EwYmA
W13fK6jSHfRSXa2oNo60cy2y4oWACGeB2iWFwAcvuviwwnNZdfZR2dNj4ZBY0KCW
UgGViCGUzvHUQeTFcBffurXvKZ980wZ0W9xeTfeTXTdTyOETXDEwUVEwk8eHCIz3
W2BudA2PJm8BpHxE3hhnHzgpAb6g5OkQCQ0bXsLTYyOzZzoZ2iWMW07uaPFpQYxr
ObyRPshYTFXnx4EYnr6NQF4sCdcvJntwDsYMzodP0GBxqKx9XAfuya74LT63EtN2
vz6RymAuGpfzraED2lkbRrXYNciDXbbKbjbDSXNCaVP0xS1Dd0lZwTonvWj+Ao+W
kzBy0GZyJXIC
-----END CERTIFICATE-----
######## INFORMATIONAL: NOT PART OF OUR EXISTING root.pem ########
-----BEGIN CERTIFICATE-----
In case we had a local Intermediate CA configured in previous post we should also add the certificate here. And do the same for any other created CA hierarchy level.
-----END CERTIFICATE-----
######## INFORMATIONAL: NOT PART OF OUR EXISTING root.pem ########
{% endhighlight %}
- The private key, which **must not have a password!**! We will export the key from the p12 files that we created in the previous step.
{%highlight shell %}
$ openssl pkcs12 -in apache.p12 -nocerts -nodes > /tmp/apache.key
{% endhighlight %}
{%highlight shell %}
Enter Import Password:
{% endhighlight %}
Provide the password added in the apache.p12 creation or if you didn't add on just press Enter.
- The certificate issued by either the official Authority or our internal one,in our case **wildcard.homelab.home.crt**. We will follow the official tutorial though and export our cer file from the p12.
{%highlight shell %}
$ openssl pkcs12 -in apache.p12 -nokeys > /tmp/apache.cer
{% endhighlight %}
{%highlight shell %}
Enter Import Password:
{% endhighlight %}

So from a self-signed SSL point of view we have all the needed files copied to our oVirt installation,

- **/tmp/root.pem**
- **/tmp/apache.p12**
- **/tmp/apache.key**
- **/tmp/apache.cer**

and to be sure that we are on the correct path, we can check the md5 hash and verify that the extracted files are correct by issuing,

{%highlight shell %}
$ openssl x509 -noout -modulus -in /tmp/apache.cer | openssl md5
(stdin)= 4dfa82a8a8b3e2c3885c4448a8144eaa
$ openssl rsa -noout -modulus -in /tmp/apache.key | openssl md5 
(stdin)= 4dfa82a8a8b3e2c3885c4448a8144eaa
$ openssl x509 -noout -modulus -in wildcard.homelab.home.crt | openssl md5
(stdin)= 4dfa82a8a8b3e2c3885c4448a8144eaa
$ openssl rsa -noout -modulus -in wildcard.homelab.home.key | openssl md5 
(stdin)= 4dfa82a8a8b3e2c3885c4448a8144eaa
{% endhighlight %}

Both latest cer and key have the same md5 with our old wildcard.homelab.home.* equivalents. 

We can now proceed with the oVirt SSL replacement!

#### Replacing the oVirt Engine SSL Certificate

_**<strong>Warning No.1:</strong>** In case you want to backup folders like **/etc/pki** or **/etc/pki/ovirt-engine** please keep in mind that the folder permissions must remain to 755._<br/>
_**<strong>Warning No.2:</strong>** One more point is that this procedure assumes that this is a new 4.x installation and not an upgrade from version 3.6. The later case is also covered by the official [tutorial][tutorial]._

1.Back up the current apache.p12 file
{%highlight shell %}
$ cp -p /etc/pki/ovirt-engine/keys/apache.p12 /tmp/apache.p12.bck
{% endhighlight %}
2.Replace the current file with the one that we created
{%highlight shell %}
$ cp /tmp/apache.p12 /etc/pki/ovirt-engine/keys/apache.p12
{% endhighlight %}
3.Replace our third-party CA certificate and update the trust store
{%highlight shell %}
$ cp /tmp/root.pem /etc/pki/ca-trust/source/anchors
$ update-ca-trust
{% endhighlight %}
4.The Engine is configured to use /etc/pki/ovirt-engine/apache-ca.pem, which is symbolically linked to /etc/pki/ovirt-engine/ca.pem. Let's remove the symbolic link and save our certificate as apache-ca.pem to the appropriate path (we can either backup the apache-ca.pem or create a new symlink in case we need to rollback).
{% highlight shell %}
$ rm /etc/pki/ovirt-engine/apache-ca.pem
$ cp /tmp/wildcard.homelab.home.cer /etc/pki/ovirt-engine/apache-ca.pem
{% endhighlight %}
5.Backup the existing private key and certificate in case you need to rollback
{% highlight shell %}
$ cp /etc/pki/ovirt-engine/keys/apache.key.nopass /etc/pki/ovirt-engine/keys/apache.key.nopass.bck
$ cp /etc/pki/ovirt-engine/certs/apache.cer /etc/pki/ovirt-engine/certs/apache.cer.bck
{% endhighlight %}
6.Copy our issued private key and certificate to the required location
{% highlight shell %}
$ cp /tmp/apache.key /etc/pki/ovirt-engine/keys/apache.key.nopass
$ cp /tmp/apache.cer /etc/pki/ovirt-engine/certs/apache.cer
{% endhighlight %}
7.Restart Apache server.
{% highlight shell %}
$ systemctl restart httpd.service
{% endhighlight %}
8.Create a new trust store configuration file,
{% highlight shell %}
$ vi /etc/ovirt-engine/engine.conf.d/99-custom-truststore.conf
{% endhighlight %}
add the appropriate content,
{% highlight bash %}
ENGINE_HTTPS_PKI_TRUST_STORE="/etc/pki/java/cacerts"
ENGINE_HTTPS_PKI_TRUST_STORE_PASSWORD=""
{% endhighlight %}
and save the file.
9.Edit /etc/ovirt-engine/ovirt-websocket-proxy.conf.d/10-setup.conf file,
{% highlight shell %}
$ vi /etc/ovirt-engine/ovirt-websocket-proxy.conf.d/10-setup.conf
{% endhighlight %}
make the following changes,
{% highlight bash %}
SSL_CERTIFICATE=/etc/pki/ovirt-engine/certs/apache.cer
SSL_KEY=/etc/pki/ovirt-engine/keys/apache.key.nopass
{% endhighlight %}
and save the file.
10.Edit /etc/ovirt-imageio-proxy/ovirt-imageio-proxy.conf file
{% highlight shell %}
$ vi /etc/ovirt-imageio-proxy/ovirt-imageio-proxy.conf
{% endhighlight %}
make the following changes
{% highlight bash %}
# Key file for SSL connections
ssl_key_file = /etc/pki/ovirt-engine/keys/apache.key.nopass
# Certificate file for SSL connections
ssl_cert_file = /etc/pki/ovirt-engine/certs/apache.cer
{% endhighlight %}
11.Last but not least , proceed with all appropriate oVirt related service restarts.
{% highlight shell %}
$ systemctl restart ovirt-provider-ovn.service
$ systemctl restart ovirt-imageio-proxy
$ systemctl restart ovirt-websocket-proxy
$ systemctl restart ovirt-engine.service
{% endhighlight %}

If everything is correctly configured we should now be able to connect to the Administration and VM Portal without being warned about the authenticity of the certificate used to encrypt HTTPS traffic.

Let's access our oVirt administration page **https://ovirt.homelab.home/ovirt-engine/** and check,

<a href="/assets/2020-01-20-ovirt_ssl/ovirt-ssl.png" data-lightbox="ovirt-large" data-title="Check out oVirt admin page">
  <img src="/assets/2020-01-20-ovirt_ssl/ovirt-ssl.png" title="Check out oVirt admin page">
</a>

Our self-signed SSL is correctly configured and enabled!

#### Final Words

This replacement has no impact to the established connections between the Engine and the hosts as they use self-signed certificates issued internally by the engine.
So there is no risk in the process and if anything goes south, just restore the files that we've backed up and restart the services.

One more important note is that the trust chain in the third-party CA (in our example root.pem) must be followed in a specific way, from the last Intermediate CA to the Root. So take good care on the bundles that you have to download from each authority or the ones created internally in case a local Intermediate CA is present. If not you will end up in a very unpleasant troubleshooting trying to pin point where is the exact problem in the whole procedure.

From this point on I will be able to use my wildcard certificate in any of the new projects that I have in mind and I will have the chance to also document the installation to either Apache or Nginx web servers. So stay tuned and I promise I will try to document every new step regardless the outcome!

Until my next post and as Dr Wallace Breen says in Half-Life 2.. Be wise. Be safe. Be aware!

[tutorial]: https://www.ovirt.org/documentation/admin-guide/appe-oVirt_and_SSL.html
[ca]: https://zroupas.github.io/linux/2019/12/13/local-ca-setup.html