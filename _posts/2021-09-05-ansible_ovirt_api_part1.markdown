---
layout: post
title: "Automatically create virtual machines via Ansible and oVirt API - Part One"
date: 2021-09-05 21:10:59
author: Zois Roupas
categories: [linux, Virtualization, automation]
tags: linux,ovirt,api,ansible
cover: "/assets/ansible-api/cover.png"
---

### Automatically create virtual machines via Ansible and oVirt API - Part One.

<hr>

At this point my (virtual) infrastructure was ready to welcome more virtual machines that could be successfully backed up and restored to a previous state. If you want to check the backup/restore procedure that I'm following, you can find all related info to the [Create and Export Full oVirt VM Backups][ovirt-backup] post. 

So nothing could go wrong or get between me and my fancy ideas any more!

But even though i was super excited and wanted to directly create as many virtual machines as I could, I didn't like the idea of having to follow all the manual steps that I have documented in my previous posts for each and every new vm!

Then it hit me! What if I could automate the procedure and create a new virtual machine with a push of a button? To do that I needed an automation tool that could make use of the **oVirt REST-API**. 

It was time to bring out the big guns for my first official project! **Ansible**! 

### Appendix 📓
First things first though and before moving to the interesting part we need to explain the terms that are going to be used in this post.

#### What's a(n) API / REST-API?
First let's start by explaining what an **API** is. I won't get into much detail because there are tons of great material to get information from but for the purposes of this post I will document below the explanation that made sense to me when I first encountered this concept.

Think of the API as the **Waiter** in your favourite restaurant. He is the one standing between you (*Application/Client*) and your delicious meal. Of course you have the menu to decide but let's say you want the BBQ Burger 🍔 but with the tomato removed! You need to ask the **Waiter** (*API*) if this is possible, he will then have to forward your request to the **Kitchen** (*Server*) and return with an answer (and probably with a spit on your burger 😂 ).

Now let's get a bit more professional shall we? **API**  (*Application Programming Interface*) is a set of definitions and protocols for building application software that acts as an intermediary allowing two applications to talk to each other by granting all the required permissions. 

Wait, we are not done yet!! **oVirt Engine** is providing a [REST-API][ovirt-rest-api], a subset of **APIs**, which is a set of guidelines for building a web service API which uses **URIs** and **HTTP** protocol and **JSON** for data format. **REST-APIs** are more flexible as they can handle different types of calls and return data in different formats (**XML, JSON, YAML** etc).

#### And what about Ansible?

There are multiple automation tools out there and each one has it's own strengths and weaknesses and let me explain what I mean! For example, I din't want to mess with ruby at this point (or at any point of time honestly), so this ruled out ~~**Chef**~~ immediately. I also wanted an agentless push configuration solution which also ruled out ~~**Puppet**~~. I didn't have any experience with **Salt**, so this left me with .. 🥁.. **Ansible**!

I don't have a fancy example this time to be honest, but I have an awesome *Fun Fact* that you probably aren't aware of!<br>The term “ansible” was first introduced in 1966 Ursula K. Le Guin’s novel **Rocannon’s World**!! As I am a huge Ursula Le Guin and Sc-fi fan, this was mind blowing for me back when I first came across to the **Ansible** software.

Ansible is the **Swiss army knife** equivalent of IT automation engines, a powerful open-source automation tool that acts as a configuration management, application deployment, orchestrator and cloud provisioning.

Ansible is an **agentless tool** that runs in a ‘push’ model, temporarily connecting remotely via SSH or Windows Remote Management to make them manageable. This makes it the best option for our project as it is a flexible and highly customizable tool which provides an ad-hoc way to configure and manage various parts of our oVirt infrastructure. 

And the best part is that oVirt has everything ready for us as it is already maintaining multiple [Ansible roles][ovirt-ansible-roles]! Head to the official [site][official-ansible] so as to get started with Ansible in case you haven't done so already!

#### Say to hello to my little Cloud-Init friend (with the voice of Toni Montana) !
Red Hat's official documentation has a great explanation regarding cloud-init and I think that I shouldn't interfere thus I quote:<br>
> "`cloud-init` is a software package that automates the initialization of cloud instances during system boot. You can configure `cloud-init` to perform a variety of tasks. Some sample tasks that `cloud-init` can perform include:
> -   Configuring a host name
> -   Installing packages on an instance
> -   Running scripts
> -   Suppressing default virtual machine behaviour"

As you can understand there are many different scenarios that **cloud-init** could come in quite handy.<br>We will use this software to apply our own custom setup (*Static IP/Subnet/Resolv/DNS/Domain* etc) for each and every vm that we create.

### Prerequisites 📝

In order to be able to understand and test the concept we will need the below setup/configurations available:
 - Our [**oVirt infrastructure**][ovirt_installation] with a correctly configured [SSL][ovirt_ssl] certificate (self signed or official one). You can always bypass the SSL configuration but you will need to adapt accordingly the commands that are used during the post as I'm using a secure connection, playbook wise.
 - a **CentOS 7 virtual machine** ( hopefully the last machine that you will have to install and configure manually 🙏 ) that has ansible package installed and configured.<br>This vm will host our yaml configuration files and eventually execute the playbooks that will create the new vm's to our oVirt host. Check out this great [post][install-ansible-centos7] if you haven't configured one yet.
- a **Centos7 template with cloud-init** software configured. Again you can find lots of great posts regarding this concept of creating your own cloud-init ready CentOS ISO image or install and update your oVirt template. You can combine for example this cloud-init configuration [post][cloud-init-setup] with my previous article on how to create your [own template][ovirt_config_2]!
- In case you haven't come across yet to one of my previous posts (for example on how to [backup your oVirt vm's][ovirt-backup]) , and even if it's not a prerequisite for this post , it's good to have **Python SDK** installed for the **oVirt Engine API** which is the collection of classes that will allows us to interact and run our oVirt system tasks.
**For EL distros (such as CentOS, RHEL, etc.):**
{% highlight shell %}
[ovirt-tested-$VERSION]
baseurl=http://resources.ovirt.org/repos/ovirt/tested/$VERSION/rpm/el$releasever
name=oVirt-$VERSION
enabled=1
gpgcheck=0
{% endhighlight %}

{% highlight bash %}
:~$ yum install python-ovirt-engine-sdk4 ovirt-engine-sdk-python
{% endhighlight %}

or download the rpm directly from [here][python-sdk] in case you stumble upon package manager issues that are difficult for you to solve and install it manually,

{% highlight bash %}
:~$ rpm -iv python-ovirt-engine-sdk4-4.2.7-1.el7.src.rpm
{% endhighlight %}

### Architecture ✏️
I find it quite useful to have a short sketch which can easily show me what I have already configured and what I want to achieve.

Time for a drawing and more information 😅! 

<a href="/assets/ansible-api/architecture.png" data-lightbox="architecture" >
  <img src="/assets/ansible-api/architecture.png" title="architecture">
</a>

In the above picture we see what we have already configured: 
 - **oVirt Host** : inside of which I create VM's, based on a template because i don't want to configure users, packages, ssh keys etc, by following more or less these [steps][vm-step-by-step]. So you understand, or you'll see on your own when creating more virtual machines, that those are a lot of clicks, especially if you intend to keep your services separated and in different machines.
    - *hostname*: **hl-ovirt**
    - *DNS*: **ovirt.homelab.home**
    - *IP*: **10.0.0.3**
 - **Ansible VM** : This is a virtual machine, created in our oVirt infrastructure and which hosts all the ansible related configuration/files.
    - *hostname*: **hl-ansible**
    - *DNS*: **ansible.homelab.home**
    - *IP*: **10.0.0.10**

and what we want to achieve:

- our goal here is to be able to run a playbook from the **ansible.homelab.home** which will call the **REST-API** provided by the **oVirt host** that will instruct the host to create a new machine with the **Name** & **IP** of our choosing by using the already configured **cloud-init template** hosted on our oVirt host. 

Confusing 😕? Nahhhh you, you'll be alright, everything is going to be fine!

### Configuration 🔨

First step in projects that APIs are involved, is to configure the authorization and make sure that our account has appropriate permissions. I will start by trying to get a list of the installed VM's that are hosted in my oVirt host and see what happens. 

I will use the admin user , the same one that I'm using to connect to the oVirt GUI, just for testing. In a production environment you should never use the admin account for various security reasons but for my local home-lab there is no need to go down that road!

But before starting interacting with oVirt API, let's see if we can fetch system related information on our host and make sure that **Ansible** is correctly configured and has all the appropriate permissions.

#### Ansible Ad-Hoc 🏃

Let's see in action some examples of what we can do with **Ansible** and then we move to the playbooks part.

I guess that you have a working ansible vm at this point and maybe another host that ansible can connect and retrieve informations from. You can always apply the commands below on your local machine but for our case we will stick to the plan!

Let's try and ping our oVirt host and confirm that we have connectivity:
{% highlight bash %}
:~$ ansible ovirt.homelab.home -m ping'
ovirt.homelab.home | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
{% endhighlight %}

Ok, we can connect to our host! Let's check it's hostname:
{% highlight bash %}
:~$ ansible ovirt.homelab.home -m shell -a 'echo $HOSTNAME'
ovirt.homelab.home | SUCCESS | rc=0 >>
hl-ovirt
{% endhighlight %}

ohh nice! Let's get the uptime of our host:
{% highlight bash %}
ansible ovirt.homelab.home -m shell -a uptime 
ovirt.homelab.home | SUCCESS | rc=0 >>
 14:18:13 up 8 days, 21:31,  2 users,  load average: 0.10, 0.09, 0.11
{% endhighlight %}

And we can do this for multiple servers! We can check RAM/CPU resources, copy/fetch files, change file/folder permissions, install packages and check if a service is up and running with the use of some ansible magic! Awesome right? 

Time to get to the grown up stuff though!

#### Ansible Playbooks 📚

I won't get into too much detail on the playbook format and the spaces and dashes that needs to have as I take for a fact that you have some experience on **Ansible** and if not please head to the **Appendix** to find info and useful links.

But in order to understand the next code snippets I must point out that *playbooks* are expressed in **YAML** format and composed of one or more '*plays*' that run in order from top to bottom against all machines matched by the host pattern.

Start by adding below yaml code to a .yml file. I'm doing it with the use of EOF (End of file) operator so as to minimize any structure related yaml errors, so just copy/paste below output to your terminal:

{% highlight yml %}
{% raw %}
:~$ cat <<EOF > ovirt_list_vms.yml
---
- hosts: ovirt.homelab.home
  connection: local

  tasks:
  - name: Obtain SSO token
    ovirt_auth:
      url: https://ovirt.homelab.home/ovirt-engine/api
      username: admin@internal
      password: "<add your password here>"

  - name: List vms
    ovirt_vms_facts:
      fetch_nested: true
      nested_attributes:
        - description
      auth: "{{ ovirt_auth }}"

  - name: set vms
    set_fact:
       vm: "{{ item.name }}: {{ item.snapshots |
map(attribute='description') | join(',') }}"
    with_items: "{{ ovirt_vms }}"
    loop_control:
      label: "{{ item.name }}"
    register: all_vms

  - name: make a list
    set_fact: vms="{{ all_vms.results | map(attribute='ansible_facts.vm') | list }}"

  - name: Print vms
    debug:
      var: vms
EOF
{% endraw %}
{% endhighlight %}

Ok, don't jump off the window yet, please 😆! If this is the first time that you stumble upon yaml code then keep in mind that the normal feeling after seeing the above is.. 😱 !!

Let's have a look on each section of the playbook. Keep in mind that I'm not an expert and you **must not take the next steps break down as a granted**, this is my personal notes and understanding of the playbook that we've just created:

 - The first section is the *host* which tells our playbook which is the target host (or group of hosts in other cases) that the playbook is going to be executed on. In our case the oVirt host of our infrastructure (**ovirt.homelab.home**) which provides the **REST-API**.
 - Next we have the *tasks* section and the first *play* where we are using the **ovirt_auth** module to authenticate to oVirt engine and create an SSO token which will be used in the next play.
 - In the second play we are using **ovirt_vms_facts module** to retrieve the description of all vm's currently deployed in our home-lab infrastructure by using the SSO token created in the previous task. We could always also use a pattern instead of the a nested_attribute, for example :
```
ovirt_vms_facts:
    pattern: name=centos* and cluster=<our_cluster_name>
```
- Then we use the **set_fact module** which allows us to dynamically add or change facts during execution. In our case we want to output a list of vm names and their snapshot description, filter ```(|)``` by description separated by comma for multiple snapshots (if there are any..) while looping though the active virtual machines. Then we register the output to **all_vms** *variable* so as to be filtered in the next play. Again this is just an example and you can create your own list columns with the help of a little **JSON PATH** magic!<br>We then apply another filter to the previous **all_vms** variable with the use of map that just allows us to change every item in a list, using the ‘*attribute*’ keyword and do the transformation based on attributes of the list elements and assign the list to a new *variable* called **vms**.<br>To better understand what would be the output at this exact point, you can remove the **"make a list"** play and in **"print vms"** one replace **vms** variable with the **all_vms**. In that way you can test different tasks and figure out on your own what would be the outcome of each one if run separetly.<br>This would be the output in such case (do you see the all_vms.result that we are using in the *"make a list"* task?) :
{% highlight yml %}
{% raw %}
ok: [ovirt.homelab.home] => {
    "all_vms": {
        "changed": false, 
        "msg": "All items completed", 
        "results": [
            {
                "_ansible_ignore_errors": null, 
                "_ansible_item_label": "hl-dns", 
                "_ansible_item_result": true, 
                "_ansible_no_log": false, 
                "ansible_facts": {
                    "vm": "hl-dns: Active VM"
                }, 
                "changed": false, 
                "failed": false, 
                "item": {
                    "affinity_labels": [], 
                    "applications": [], 
                    "bios": {
                        "boot_menu": {
                            "enabled": false
                        }
                    }, 
[more output]
{% endraw %}
{% endhighlight %}
- Finally we print the finished filtered list that is assigned to the **vms** *variable*.

And again, maybe I'm mistaken, maybe I didn't explain something correctly or adequately so please either use the ansible-doc to *search* for more detailed module information,
```
:~$ ansible-doc set_fact
```
or head to the official [Ansible documentation][official-ansible] to clear up anything that seems incorrect or not legit! Then please drop me a line so as to update my notes 😂 🙏!

Time to run our playbook and see what happens!

### Execution 🖱️
Open a terminal, head to the folder that you have saved our .yml file and run the command below: 

```
$ ansible-playbook ovirt_list_vms.yml 
```
```
PLAY [ovirt.homelab.home] **********************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************************
ok: [ovirt.homelab.home]

TASK [Obtain SSO token] *********************************************************************************************************************************
ok: [ovirt.homelab.home]

TASK [List vms] *****************************************************************************************************************************************
ok: [ovirt.homelab.home]

TASK [set vms] ******************************************************************************************************************************************
ok: [ovirt.homelab.home] => (item=hl-ansible)
ok: [ovirt.homelab.home] => (item=hl-dns)

TASK [make a list] **************************************************************************************************************************************
ok: [ovirt.homelab.home]

TASK [Print vms] ****************************************************************************************************************************************
ok: [ovirt.homelab.home] => {
    "vms": [
        "hl-ansible: Active VM, before_ansible_setup", 
        "hl-dns: Active VM"
    ]
}

PLAY RECAP **********************************************************************************************************************************************
ovirt.homelab.home           : ok=6    changed=0    unreachable=0    failed=0   
```

Success!!

So our playbook did exactly what we told it to do! It connected to our oVirt via the REST-API, obtained an **SSO token** based on the username/password login credentials and then created a list of our currently deployed virtual machines and their snapshot description (I forgot that I had a snapshot for my hl-ansible vm, time to delete that and save some time and backup space..) ! 

The **Play Recap** output part, informs us that no failures and no changes were applied, as we didn't update anything, and all 6 plays run successfully!🎉🎉

### Intermission ☕

It seems to me like a good point to conclude this first part. I have given more than enough information and configuration links that you can refer to in case you want to dig deeper on your own regarding both **Ansible** and **APIs** while waiting for part two!

Kudos for managing to get to this point, really, even if you have a headache and you are seriously thinking of jumping to another job field! Before doing so though, grab a well reserved beverage, go through the notes, configure and play with **Ansible** a bit and I think that you will change your mind!

In the second part of this post, we will try to create our next virtual machine automatically via a playbook and maybe wrap it around a bash script! Stay tuned!! 🎸

Thanks for reading, until the next post and as Dr Wallace Breen says in Half-Life 2.. <br>Be wise. Be safe. Be aware!

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
[cloud-init-setup]: https://www.ibm.com/docs/en/powervc-cloud/2.0?topic=linux-installing-configuring-cloud-init-rhel
