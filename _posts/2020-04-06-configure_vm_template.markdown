---
layout: post
title: "Configure oVirt Engine on CentOS 7 - Part Two"
date: 2020-04-06 20:32:59
author: Zois Roupas
categories: Virtualization
tags: linux,ovirt,centos7,templates
cover: "/assets/ovirt-vm/services.png"
---

### Create your first Virtual Machine and Save it as a Template.

<hr>

In the [First Part][ovirt_config_1] of this tutorial, we documented in detail everything needed to setup our **oVirt Infrastructure** which is ready to host new virtual machines.

I suppose that you had enought time to familiarize yourself with the oVirt GUI by now, so let's move on by creating our first virtual machine and save it as a template!

After this step we will be finally ready to start our journey of conquering the world with your amazing projects!

<h3 id="anchor">Prerequisites 📝</h3>

* In order to be able to connect to remote virtual machines and troubleshoot in case any of those becomes unavailable for any reason, we need to configure a **Console Client**.<br>I'm using the **oVirt Remote Viewer** which can be either downloaded from the [Official Site][remote_viewer] for both **Linux/Windows** or installed via a package manager:

**CentOS/Fedora**
{% highlight bash %}
yum install -y virt-viewer
{% endhighlight %}

**Ubuntu/Mint**
{% highlight bash %}
sudo apt-get install virt-viewer
{% endhighlight %}

### Create your First VM 🖥️

Fire up your favorite browser, type your oVirt endpoint or IP and login with your admin account. By now I guess you have seen my previous posts on local [Certificate Authority][ca] and [DNS][local_dns] configuration and you are familiar with your secure oVirt Administration Panel!

<a href="/assets/ovirt-vm/login.png" data-lightbox="login" >
  <img src="/assets/ovirt-vm/login.png" title="login">
</a>

From the left panel go to **Compute** -> **Virtual Machines**. As you can see our VM section is empty, so let's not waste our precious time and create one!

<a href="/assets/ovirt-vm/initial_no_vm.png" data-lightbox="initial_no_vm" >
  <img src="/assets/ovirt-vm/initial_no_vm.png" title="initial_no_vm">
</a>

After clicking on **New** button, we are introduced with the **General Options** configuration Tab.<br>Add a **Name** for your first VM, a **Description** and at the **Instance Images** section click on **Create** option to specify the disk space that is going to be used by your VM.

<a href="/assets/ovirt-vm/general.png" data-lightbox="general" >
  <img src="/assets/ovirt-vm/general.png" title="general">
</a>

Choose the disk size in **GB** and leave the other options with the default values. In the bottom left, choose **Show Advanced Options** and go to **System** Tab.

<a href="/assets/ovirt-vm/disk.png" data-lightbox="disk" >
  <img src="/assets/ovirt-vm/disk.png" title="disk">
</a>

Here we can choose the **Memory Size** and **Total CPU** that we want to assign to our virtual machine. We can also choose if we to install from a **Template**, which is not possible for now as we haven' configured one yet.

<a href="/assets/ovirt-vm/system.png" data-lightbox="system" >
  <img src="/assets/ovirt-vm/system.png" title="system">
</a>

*Let me point out that there are a lot of different configuration options that can be changed based on your experience and needs, but I will only document the ones that are important for a basic VM creation.*

So the last but not least configuration option is handled in the **Boot Options** Tab.

Enable **Attach CD** and choose the ISO file that we have already uploaded and is documented in the **Prerequisites** Section of [Part One][ovirt_config_1].

<a href="/assets/ovirt-vm/boot.png" data-lightbox="boot" >
  <img src="/assets/ovirt-vm/boot.png" title="boot">
</a>

Click **OK** and *voilà*! 🥂

 <a href="/assets/ovirt-vm/vm.png" data-lightbox="vm" >
  <img src="/assets/ovirt-vm/vm.png" title="vm">
</a>

Hm.. I made it seem like a big thing by using a French word, right?

Because it actually was!

Like the time you created your first PC you had to go hunting for a CPU, RAM and a Hard Disk, among other important parts 😛 , now you just had those and needed to assign them to your new VM! **Time to install our good old Penguin**!

### Installing CentOS7 to our VM 🔨

Let's Run our VM and see what happens!

Choose you virtual machine, click on **Run** and then when Console is available, click on it and choose to open the file with the **Remote Viewer**, which we have configured in the **<a href="#anchor">Prerequisites</a>** Section, in order to have a look on what happened when we booted up our VM. 

<a href="/assets/ovirt-vm/boot_iso.png" data-lightbox="boot_iso" >
  <img src="/assets/ovirt-vm/boot_iso.png" title="boot_iso">
</a>

If you followed the steps documented in the previous section then you should be able to start the Centos Installation Process by choosing the **Install Centos 7**. 

Press **Enter** to start the installation. 

<a href="/assets/ovirt-vm/startup.png" data-lightbox="startup" >
  <img src="/assets/ovirt-vm/startup.png" title="startup">
</a>

**Installation Intermission**: *Due to the fact that  there are a lot of great tutorials out there to help you get a new Centos7 up and running, I will skip the actual OS installation*. 

Instead here's a gif that keeps you company during Centos7 installation! ⏱️

<a href="/assets/ovirt-vm/installation.gif" data-lightbox="installation" >
  <img src="/assets/ovirt-vm/installation.gif" title="installation">
</a>

After the installation is finished you will be able to either connect to your VM via SSH or directly from the Console!

<a href="/assets/ovirt-vm/vnc.png" data-lightbox="vnc" >
  <img src="/assets/ovirt-vm/vnc.png" title="vnc">
</a>

Congratulations, your first VM is now a reality! 🎉

### Cleanup and Export your VM as a Template 🚿

Templates can save us a huge amount of time in cases where we need to deploy a large number of similar VM's, especially in a corporate environemt, but they are nothing more than a **pre-installed/configured** virtual machine with it's own OS and resources.

I found templates to be very useful in my homelab setup where I'm using the same distribution and basic resources for all my VM's. Whenever there is a need for a new virtual machine, I clone one from my Centos7 template, change it accordingly and have it ready in some minutes instead of going through all the steps documented in the previous section. 

In order to be able to use the template we have to configure and make it a bit generic. <br>This means that we will have for example to:

1. Add a general hostname
2. Clear machine's id
3. Remove HDADD value from network device
4. Delete any ssh keys and logs
5. Unconfigure VM

In this way every time we clone a machine, it will have it's unique information which we can login and change/update appropriately.

#### Seal your VM 🛡️
* Login to your virtual machine

* Set a generic hostname

{% highlight bash %}
:~$ hostnamectl set-hostname localhost.localdomain
{% endhighlight %}

* Clear machine's id

{% highlight bash %}
:~$ :> /etc/machine-id
{% endhighlight %}

* Delete any host and root ssh keys

{% highlight bash %}
:~$ rm -f /etc/ssh/ssh_host_*
:~$ rm -rf /root/.ssh/
:~$ rm -f /root/anaconda-ks.cfg
:~$ rm -f /root/.bash_history
:~$ unset HISTFILE
{% endhighlight %}

* Configure Network Interface

This step depends on your configuration as there are different changes based on the IP aasingment. My only note in this section is to **remove** the *HWADDR=* line. If you need more information you can check this great [Github][minimal_centos] page or on the official [oVirt Documentation][ovirt_seal].

* Install oVirt agent (Optional)

This is the oVirt management agent running inside the guest (our VM) which supplys run-time data and heart-beat info.<br> Even though it's great to have this enabled in every machine, i saw that it needs resources that i didn't want to spare but if you want to install it here is the way to do so,
{% highlight bash %}
:~$ yum -y install epel-release
:~$ yum install ovirt-guest-agent-common
:~$ systemctl enable ovirt-guest-agent && systemctl start ovirt-guest-agent
{% endhighlight %}

* Set VM to an unconfigured state

This final command will create a file called /.unconfigured and **shutwdown the machine immediately**. The presence of this file will cause /etc/rc.d/rc.sysinit to act like it is the first time the machine is booting up and thus reconfigure networking/routing, time/date, services and delete all rules from /etc/udev/rules.d/,among other actions! You can check [sys-config man page][sys-config] for more info.
{% highlight bash %}
:~$ sys-unconfig
{% endhighlight %}

Our VM is now stopped and ready to be exported!

#### Export VM to Template 📤

Now in order to export our virtual machine to template, we have to choose it and click on the 3 dots found the upper right corner. Then choose **Make Template**.

<a href="/assets/ovirt-vm/make_template.png" data-lightbox="make_template" >
  <img src="/assets/ovirt-vm/make_template.png" title="make_template">
</a>

This will open up the **New Template** configuration where we can add a **Name**, **Description** or even change **Disk Alias**(optional). 

<a href="/assets/ovirt-vm/new_template.png" data-lightbox="new_template" >
  <img src="/assets/ovirt-vm/new_template.png" title="new_template">
</a>

Click **OK** to start templating!

You can check the progress in the **Events** Tab. 

<a href="/assets/ovirt-vm/task_started.png" data-lightbox="task_started" >
  <img src="/assets/ovirt-vm/task_started.png" title="task_started">
</a>

After this task is finished, we should be able to find our new template in **Compute** -> **Templates** menu.

<a href="/assets/ovirt-vm/templates.png" data-lightbox="templates" >
  <img src="/assets/ovirt-vm/templates.png" title="templates">
</a>

Our template is successfully created and waiting to be used! If you feel safe, just delete the already created VM and recreate, but this time use the template!

<a href="/assets/ovirt-vm/delete_first_vm.png" data-lightbox="delete_first_vm" >
  <img src="/assets/ovirt-vm/delete_first_vm.png" title="delete_first_vm">
</a>

<a href="/assets/ovirt-vm/initial_no_vm.png" data-lightbox="initial_no_vm" >
  <img src="/assets/ovirt-vm/initial_no_vm.png" title="initial_no_vm">
</a>

<a href="/assets/ovirt-vm/new_vm.png" data-lightbox="new_vm" >
  <img src="/assets/ovirt-vm/new_vm.png" title="new_vm">
</a>

Click on **New VM** , add a **Name** and click OK!

### Final Words 💡

You are finally able to start creating as many VM's or templates as you like based on your oVirt hardware capacity!  

There are so many advantages on having your own homelab to play with and experiment without having to spend a fortune!<br>
I hope those two detailed parts provided a better understanding on which are the most important ones that will help you get started on *configuring your own Open Source Virtual Infrastructure* and *creating your first virtual machine*.

In the near future I will also document the cloud-init configuration that I'm using in order to create new virtual machines automatically via **Ansible** through the oVirt API, so stay tuned! 🎶🎶

Thanks for reading, until the next post and as Dr Wallace Breen says in Half-Life 2.. <br>Be wise. Be safe. Be aware!

[ovirt_config_1]: https://myhomelab.gr/virtualization/2020/03/04/configure_cluster_host_ovirt.html
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
