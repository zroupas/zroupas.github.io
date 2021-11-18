---
layout: post
title: "Automatically create virtual machines via Ansible and oVirt API - Part Two"
date: 2021-11-17 20:32:59
author: Zois Roupas
categories: [linux, Virtualization, automation]
tags: linux,ovirt,api,ansible
cover: "/assets/ansible-api-2/cover.png"
---
### Automatically create virtual machines via Ansible and oVirt API - Part Two.

<hr>
Welcome to the second part of our oVirt API manipulation via Ansible guide!

I guess you've already gone through the [first part][ansible-api-1], played around with **Ansible** on your own and waited anxiously for the second one (Noooot). Or maybe you've already found a way to use Ansible and create your own VM's automatically after cursing for losing time reading the first part which was useless in your case! :D

But waiting time is over!!

### Architecture ✏️

To get a reminder of what our goal is and what we are gonna try to achieve in this project, check below picture

<a href="/assets/ansible-api/architecture.png" data-lightbox="architecture" >
  <img src="/assets/ansible-api/architecture.png" title="architecture">
</a>

and for a more detailed explanation head to the relevant [post][ansible-api-1].

### Playbooks 📚

In the first part we managed, as an example, to get a list of our active VM's and their snapshot description. Now we will head directly to the interesting part without wasting any more time.

I assume that you have basic understanding on how Ansible works by now, and if not head to the [first part][ansible-api-1] to find useful informations and links. 

But just as a reminder in order to be able to connect to the remote host you need an inventory. This is an essential part of the initial configuration, otherwise you will end up with below output if you try to run your playbook:
```
PLAY [10.0.0.11] **********************************************************************************************************************************************************************************************
skipping: no hosts matched
```
After making sure you have a correctly configured Ansible environment, we are ready to proceed.

#### Create our first VM via Ansible
Open your favourite terminal and paste the command below so as to create the playbook **ovirt_create_vm.yml** which will be used to connect to oVirt and automatically instruct oVirt to create our next VM for us !

{% highlight yml %}
{% raw %}
:~$ cat <<EOF > ovirt_create_vm.yml
- hosts: ovirt.homelab.home
  connection: local

  tasks:
  - name: Obtain SSO token
    ovirt_auth:
      url: https://ovirt.homelab.home/ovirt-engine/api
      username: admin@internal
      password: "<add your password here>"
      
  - name: Create VM with cloud-init
    ovirt_vms:
      name: apivm
      template: "<add your template here>"
      cluster: "<add your cluster name here>"
      memory: 1GiB
      auth: "{{ ovirt_auth }}"
      state: running
      cloud_init:
        nic_boot_protocol: static
        nic_ip_address: 10.0.0.11
        nic_netmask: 255.255.255.0
        nic_gateway: 10.0.0.254
        nic_name: eth0
        nic_on_boot: true
        host_name: apivm
        custom_script: |
          write_files:
           - content: |
               Hello, world!
             path: /tmp/greeting.txt
             permissions: '0644'
          runcmd:
            - yum -y update
        user_name: root
        root_password: apivm

  - name: Always revoke the SSO token
    ovirt_auth:
      state: absent
      ovirt_auth: "{{ ovirt_auth }}"
EOF
{% endraw %}
{% endhighlight %}

Easy right? I think it's much more straight forward than the list VM's playbook that we used in the part one .We are going to breakdown and explain each task as good as possible.

#### Playbook breakdown

- The first section is the *host* which tells our playbook which is the target host (or group of hosts in other cases) that the playbook is going to be executed on. In our case the oVirt host of our infrastructure ,**ovirt.homelab.home**, which provides the **REST-API**.
- Next we have the *tasks* section and the first *play* where we are using the **ovirt_auth** module to authenticate to oVirt engine and create an **SSO toke**n which will be used in the next play.
- In this play is where the magic happens! By using the **ovirt_vms** module we are configuring our new VM to have/use a:
  - **name**: The name of our choosing, *apivm* in our case
  - **template**: A template as described in the previous parts. You can either use your own or use the ones provided by ovirt-image-repository found in **Storage** -> **Storage Domains**.<br>I use my own one which is already configured in a way that every new VM will have basic configuration to make it manageable from the Ansible VM, **ansible.homelab.home** (ex. ansible ssh keys , ansible user, cloud-init service etc).
  - **resources & state**: I'm only using 1GB of RAM and change the VM state to running.
  - **cloud-init**: This is the time to setup a static IP instead of letting your DHCP  assign a dynamic one and having to login via console each time to find out!<br>We can do a lot more with cloud-init, in my example I also create a greetings.txt so for debugging reasons, I'm updating the OS so as to have the latest packages, not a great idea in a production environment, and finally changing the root credentials. And all of that with just one click! Cool right?
  - In the final task we use the **ovirt_auth** module in order to revoke the created SSO token from the first play. This step is optional as the SSO will be revoked automatically after some days based on the default oVirt configuration but I always find it more secure to revoke it after I have finished my task as it is not needed any more.

Let's see what we have configured!

#### Execution 🐧

##### Create VM ⏫
Before running our playbook let's make sure that IP 10.0.010 isn't already used by any other VM,
{% highlight bash %}
:~$ ping 10.0.0.11  -c 5
PING 10.0.0.11 (10.0.0.11) 56(84) bytes of data.  
From 10.0.0.227 icmp_seq=1 Destination Host Unreachable  
From 10.0.0.227 icmp_seq=2 Destination Host Unreachable  
From 10.0.0.227 icmp_seq=3 Destination Host Unreachable  
From 10.0.0.227 icmp_seq=4 Destination Host Unreachable  
From 10.0.0.227 icmp_seq=5 Destination Host Unreachable  
  
--- 10.0.0.11 ping statistics ---  
5 packets transmitted, 0 received, +14 errors, 100% packet loss, time 17414ms\
{% endhighlight %}

It's probably the first time that I got so much pleasure from 100% failure.. and we are good to go! Open your favourite terminal and run your creation playbook:
{% highlight bash %}
:~$ ansible-playbook ovirt_create_vm.yml
{% endhighlight %}
and check the output:
```
PLAY [ovirt.homelab.home] ********************************************************************************************************************************************************************************************  
  
TASK [Gathering Facts] ********************************************************************************************************************************************************************************************  
ok: [ovirt.homelab.home]  
  
TASK [Obtain SSO token] *******************************************************************************************************************************************************************************************  
ok: [ovirt.homelab.home]  
  
TASK [Run VM with cloud init] *************************************************************************************************************************************************************************************
```
If everything is configured correctly you should be stuck in the "**Run VM with cloud init**" task which for now looks promising! It means that something is at least happening in the background.<br>Let's connect to our oVirt GUI and see what is Ansible doing, login and head to the **Events** section.

<a href="/assets/ansible-api-2/events_1.png" data-lightbox="events_1" >
  <img src="/assets/ansible-api-2/events_1.png" title="events_1">
</a>

What a sight for sore eyes! **oVirt** was instructed to create a new vm and as we can see above, and **apivm** is now being prepared! We can check the progress in **Compute** -> **Virtual Machines**.

<a href="/assets/ansible-api-2/apivm-configure.png" data-lightbox="apivm-configure" >
  <img src="/assets/ansible-api-2/apivm-configure.png" title="apivm-configure">
</a>

The VM has already started and we can successfully login via console! 

<a href="/assets/ansible-api-2/apivm-started.png" data-lightbox="apivm-started" >
  <img src="/assets/ansible-api-2/apivm-started.png" title="apivm-started">
</a>

<a href="/assets/ansible-api-2/console.png" data-lightbox="console" >
  <img src="/assets/ansible-api-2/console.png" title="console">
</a>

The VM is added to our network and we are able to ping it and connect directly via SSH instead of having to check which IP would have been assigned to our VM via console.

{% highlight bash %}
ping 10.0.0.11 -c 5  
PING 10.0.0.11 (10.0.0.11) 56(84) bytes of data.  
64 bytes from 10.0.0.11: icmp_seq=1 ttl=64 time=0.575 ms  
64 bytes from 10.0.0.11: icmp_seq=2 ttl=64 time=0.802 ms  
64 bytes from 10.0.0.11: icmp_seq=3 ttl=64 time=0.869 ms  
64 bytes from 10.0.0.11: icmp_seq=4 ttl=64 time=0.782 ms  
64 bytes from 10.0.0.11: icmp_seq=5 ttl=64 time=0.874 ms  
  
--- 10.0.0.11 ping statistics ---  
5 packets transmitted, 5 received, 0% packet loss, time 4083ms 
{% endhighlight %}

In the meantime and after having created successfully the requested VM, the playbook continued by revoking the SSO token that we have created in the second task!

```
TASK [Always revoke the SSO token] ********************************************************************************************************************************************************************************  
ok: [ovirt.homelab.home]  
  
PLAY RECAP ********************************************************************************************************************************************************************************************************  
ovirt.homelab.home            : ok=4    changed=1    unreachable=0    failed=0
```
Playbook finished it's job and the recap informs us that that all 4 tasks run successfully and 1 change was applied!! 

Strangely everything worked out exactly as planned!! 😂 🥳 

##### Shutdown VM ⏬

The next playbook is the one that will conclude the second part and we will be used to shutdown the VM that we created in the previous section.<br>Open your favourite terminal and run below command that will create **ovirt_stop_vm.yml** :

{% highlight yml %}
{% raw %}
:~$ cat <<EOF > ovirt_restart_vm.yml
- hosts: ovirt.homelab.home
  connection: local

  tasks:
  - name: Obtain SSO token
    ovirt_auth:
      url: https://ovirt.homelab.home/ovirt-engine/api
      username: admin@internal
      password: "<add your password here>"

  - name: Stop vm
    ovirt_vms:
      auth: "{{ ovirt_auth }}"
      name: apivm
      state: stopped

  - name: Always revoke the SSO token
    ovirt_auth:
      state: absent
      ovirt_auth: "{{ ovirt_auth }}"
{% endraw %}
{% endhighlight %}

In addition with the previous ones , i can safely say that the above playbook is quite simple. The only point to keep in mind regarding the state handling above is that the **ovirt_vms** module doesn't support restart, at this point of time at least, and the available states are: *absent, next_run, present, registered, running, stopped, suspended*. So in case you want to restart a VM you need one more task , with state: running!

Let's run it and put our new VM to sleep until the third part of this awsome series arrives! 😉

{% highlight bash %}
:~$ ansible-playbook ovirt_restart_vm.yml
{% endhighlight %}

```
PLAY [ovirt.homelab.home] ***************************************************************************************************************  
  
TASK [Gathering Facts] ***************************************************************************************************************  
ok: [ovirt.homelab.home]  
  
TASK [Obtain SSO token] **************************************************************************************************************  
ok: [ovirt.homelab.home]  
  
TASK [Stop vm] ***********************************************************************************************************************  
changed: [ovirt.homelab.home]  
  
TASK [Always revoke the SSO token] ***************************************************************************************************  
ok: [ovirt.homelab.home]  
  
PLAY RECAP ***************************************************************************************************************************  
ovirt.homelab.home            : ok=4    changed=1    unreachable=0    failed=0
```
<a href="/assets/ansible-api-2/apivm-shutdown.png" data-lightbox="apivm-shutdown" >
  <img src="/assets/ansible-api-2/apivm-shutdown.png" title="apivm-shutdown">
</a>

With the successful run of this playbook, the actual task of manipulating the oVirt API in order to create our new VM via Ansible, is now completed! Head to your infrastructure and happy automating!!

### Final Words 💡

I hope that I managed to explain, or at least make a bit clearer, what is an API, and how to interact with it with the help of **Ansible** automation tool.

I think that both a beginner and an intermediate user can find very useful information's for getting started or dig deeper to get the most out of the **oVirt REST API** by extending the provided playbooks to create storage pools, new hosts, network profiles ,users and many more.. automatically!

As Ansible is a huge chapter, I will use the momentum to stay a bit longer in the automation area and use the next post to experiment with tasks that can be run directly on the remote server and create playbooks that will act as a "single source of truth" in our infrastructure.

We will see how to disable services (that I don't like 😆) , copy our own configuration files, use a bash script to wrap everything around it and see how we can pass pass extra variables to our playbooks!

Thanks for reading, until the next post and as Dr Wallace Breen says in Half-Life 2.. <br>Be wise. Be safe. Be aware!

[ansible-api-1]: https://myhomelab.gr/linux/virtualization/automation/2021/09/05/ansible_ovirt_api_part1.html
[ovirt_config_1]: https://myhomelab.gr/virtualization/2020/03/04/configure_cluster_host_ovirt.html
[ovirt_config_2]: https://myhomelab.gr/virtualization/2020/04/06/configure_vm_template.html
[ovirt_installation]: https://www.ovirt.org/documentation/install-guide/chap-Installing_oVirt.html
[ovirt_ssl]: https://myhomelab.gr/linux/2020/01/20/replacing_ovirt_ssl.html
[ca]: https://myhomelab.gr/linux/2019/12/13/local-ca-setup.html
[local_dns]: https://myhomelab.gr/linux/2019/09/22/local-dns.html
[ovirt_storage]: https://www.ovirt.org/documentation/admin-guide/chap-Storage.html
[Data Center]: https://www.ovirt.org/documentation/admin-guide/chap-Data_Centers.html
[Cluster]: https://www.ovirt.org/documentation/admin-guide/chap-Clusters.html
[remote_viewer]: https://virt-manager.org/download/
[minimal_centos]: https://github.com/rharmonson/richtech/wiki/CentOS-7-1511-Minimal-oVirt-Template
[sys-config]: https://linux.die.net/man/8/sys-unconfig
[ovirt_seal]: https://www.ovirt.org/documentation/
[ovirt-backup]: https://myhomelab.gr/virtualization/2020/07/13/backup-virtual-machines.html
[ovirt-rest-api]: https://www.ovirt.org/develop/api/rest-api/rest-api.html
[ovirt-ansible-roles]: https://github.com/oVirt/ovirt-ansible
[official-ansible]: https://www.ansible.com/overview/how-ansible-works
[install-ansible-centos7]: https://www.snel.com/support/how-to-install-ansible-on-centos-7/
[vm-step-by-step]: https://www.linuxtechi.com/create-virtual-machines-ovirt-4-environment/
[python-sdk]: http://ftp.iij.ad.jp/pub/linux/centos-vault/7.8.2003/virt/Source/ovirt-4.2/
[run-once]: https://rhv.bradmin.org/ovirt-engine/docs/Virtual_Machine_Management_Guide/Virtual_Machine_Run_Once_settings_explained.html
