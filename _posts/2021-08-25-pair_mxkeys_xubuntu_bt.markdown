---
layout: post
title: "Pair Logitech MX Keys with Xubuntu 18.04 via Bluetooth"
date: 2021-08-25 19:32:59
author: Zois Roupas
categories: linux
tags: linux,logitech,mxkeys,xubuntu,logi,bluetooth
cover: "/assets/logi-mx-keys/post_cover.png"
---

### Pair Logitech MX Keys with Xubuntu 18.04 via Bluetooth

<hr>
Usually I don't write posts about hardware configurations but this time I got really frustrated while trying to pair my new, much anticipated, fancy [Logitech MX Keys][mx-keys] keyboard!
Maybe this is helpful to someone experiencing the same issue there is a *catch* in the pairing procedure and Logitech's documentation included in the package isn't clear enough regarding this.

### Summary
Let me start by saying that I've been using this keyboard for about a week and I'm totally pleased with my choice. It's the first time that I bought a high-end keyboard, I was always using descent keyboards but never really gave more than 50 euros for any of them and happy with what I was getting with my money.
And honestly, why someone wouldn't be happy with his [Microsoft Comfort Curve Keyboard 2000][microsfot-curve] ?? Thank you for your services MS Comfort, you've been a loyal comrade!

Then, Covid-19 hit the world and home office increased exponentially! So my home became my actual office, which meant having a keyboard/mouse for my gaming PC and a keyboard/mouse for my work laptop. Of course this is the least of the problems that this new situation brought along but still it didn't make sense to have 2 full size keyboards/2mouses on a desk , wasting so much working space in 2021!

I thought about getting a keyboard/mouse hub but from my working experience, I have never came across to a robust solution that would do the actual work. Every single time those hubs where extremely glitchy and I would end up swearing and throwing them away!

So I though of buying a keyboard that would have both **Bluetooth** & **USB Adapter** available. In this way I could connect the keyboard to laptop via BT and use the USB adapter to my PC which doesn't have BT! 

And my search led me to this rechargeable (USB Type C), stylish, nearly silent and great built design with **smart backlighting** and ,based on the reviews, **fantastic battery life** ( 10 days on full charge and 5 months with backlighting turned off which is awesome as I'm working in a well lighted environment )!

Did I think of it correctly? Where there other solutions that would be better? Sure! Only time will show but honestly I wanted to treat myself with a new keyboard some time now 😆 !!

Enough with the chit chat, let's cut to the chase!

### Pairing!

Now let's head to the pairing *catch* that you've probably came across and was the reason that landed you here!

Here it was! My new robust keyboard in my hands and ready to start typing! Following the described procedure, I turned on the switch of the keyboard, opened my BT manager on the laptop and MX Keys new device directly popped up in the discovered devices.

<a href="/assets/logi-mx-keys/bt_devices.png" data-lightbox="bt_devices" >
  <img src="/assets/logi-mx-keys/bt_devices.png" title="bt_devices">
</a>

But when I chose **Pair**,  multiple pop up pairing messages like "Pairing MX Keys" started flooding the right upper corner of my screen but I couldn't figure out what was needed from my side. 

After googling a bit, I found the below image in Logitech's official [Setup Guide][logit-setup]..

<a href="/assets/logi-mx-keys/logi-setup.png" data-lightbox="logi-setup" >
  <img src="/assets/logi-mx-keys/logi-setup.png" title="logi-setup">
</a>

.. and it hit me! MX Keys wanted me to type a 6 digits code to my keyboard and hit ENTER! But for some reason my Xubuntu wasn't giving me any code to use ..

So back to google ✊ ! Eventually my search 🔎 led me to this very helpful [guide][bt-pair-terminal] which explains how to the follow the exact same pairing steps that GUI is using but from the terminal ( always go with the terminal choice people, always! ).

#### Console Time! 
We will use the `bluetoothctl` command to try and get the 6 digits code in order to pair our controller and the new keyboard. This command tool is provided by [bluez_5.48-0ubuntu3_amd64](https://launchpad.net/ubuntu/bionic/+package/bluez) 🐞 package.
 - Run the command to get at the prompt and tell bluetoothctl to look only for keyboards,
```bash
:~$ :> bluetoothctl
[bluetooth]# agent KeyboardOnly
Agent is already registered
```
- Put your controller in **_pairable_** mode
```bash
[bluetooth]# pairable on
Changing pairable on succeeded
```
- Instruct the controller to start ***scanning*** for a suitable device
```bash
[bluetooth]# scan on
Discovery started
[CHG] Controller 38:DE:AD:2A:BE:CA Discovering: yes
[NEW] Device 52:9B:32:5E:BF:3E 47-6B-25-5E-BF-3E
[NEW] Device D4:AD:1D:C5:77:B5 MX Keys
```
- When the correct device is found, use the MAC to ***pair***
```bash
[bluetooth]# pair D4:AD:1D:C5:77:B5
Attempting to pair with D4:AD:1D:C5:77:B5
[CHG] Device D4:AD:1D:C5:77:B5 Connected: yes
[agent] Passkey: 508203
[NEW] Primary Service
/org/bluez/hci0/dev_D4_AD_1D_C1_67_B4/service000a
00001801-0000-1000-8000-00805f9b34fb
Generic Attribute Profile
[..]
```
And there it is, the passkey is revelled! At this point type the 6 digit code to your MX keyboard, hit Enter and boom,
```bash
[..]
[CHG] Device D4:AD:1D:C1:67:B4 ServicesResolved: yes
[CHG] Device D4:AD:1D:C1:67:B4 Paired: yes
Pairing successful
```
The pairing via BT, which was previously failing, is now successful 🎉 !
### Conclusion

From my point of view,  I think that Logitech's quick guide included in the package should have a note regarding the **6 digits code** needed to be typed in order to complete the pairing and not rely on people having to go through their online guide to find this important configuration step.
Of course Xubuntu's bug of not showing the code on the other side is also something that should be addressed and fixed but maybe this is not a general issue and just something broken on my side!

Either way, now that I'm typing this post in my new awesome keyboard I can say that it totally worth the search and fix 🙌! I can now get rid of all the unneeded keyboards and free up my desk space! 

Yes I know that i didn't speak about an equivalent 1 mouse to save the world kind of thing but I have the feeling that this will be a much easier task. If not.. then expect a new post where I'm complaining about something else I guess 😂!

Even though the are multiple great videos and reviews out there regarding this keyboard, I think that I should create my own after the first 3 months and compare notes of current and future behaviour! We will see 😊 ..

Thanks for reading, until the next post and as Dr Wallace Breen says in Half-Life 2.. <br>Be wise. Be safe. Be aware!

[bt-pair-terminal]: https://wiki.archlinux.org/title/Bluetooth_keyboard
[logit-setup]: https://www.logitech.com/en-us/promo/mxsetup/keyboard-setup/bluetooth.html
[mx-keys]: https://www.logitech.com/en-us/products/keyboards/mx-keys-wireless-keyboard.920-009294.html
[microsfot-curve]: https://www.amazon.com/Microsoft-Comfort-Curve-Keyboard-2000/dp/B0009ZBRS0
