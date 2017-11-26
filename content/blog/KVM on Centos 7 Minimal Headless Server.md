---
title: "KVM on Centos 7 Minimal Headless Server"
date: 2017-11-23T17:22:09+05:30
draft: false
---

Hello All.

I did not found any good post on this so I am writing about it. These posts also help me setup the development sever from scratch whenever it crashes for one or another reason. Please note that I keep security as top priority, so SELinux is always enabled on my system.

So, I saw a couple of blog posts on how to setup on KVM on Centos 7. They all had the same flow. I will be following the same flow as well, but **also adding in my inputs as to what problems i faced with these commands and how did I solved them.** Well isn’t this always the case, you cant accomplish a task just by following one post. There is no one post that can exactly tell you how to get this work done. Let’s try to make this post not like one of those 😛

### 1. Check virtualization
Check that you system supports virtualization
```console
$ grep -E '(vmx|svm)' /proc/cpuinfo
```
if you get vmx or svm in the output, then your system does supports KVM.
### 2. Install packages
Next you install a couple of packages, the command is mentioned below.
```console
$ yum install kvm virt-manager libvirt virt-install qemu-kvm xauth dejavu-lgc-sans-fonts
```
I hope there was no problem in installing the packages.
### 3. Start KVM services
Next lets start the kvm services.
```console
$ systemctl start libvirtd
$ systemctl enable libvirtd
```
I faced no problem till this step and I hope you did not as well.
### 4. Start Networking
This is where i faced my **first problem**. Now lets start with the basic set of commands you are told to run.
```console
$ echo "net.ipv4.ip_forward = 1"|sudo tee /etc/sysctl.d/99-ipforward.conf
$ sysctl -p /etc/sysctl.d/99-ipforward.conf
```
The commands above enable IP forwarding. Next we change the configuration files of network interface, now first I can tell you that if you are gonna modify the network setting be **very careful** especially if you are working on a head less server or a server on the cloud. Cause if the only way you can access that machine is through ssh and you screw up your network config, then you will have no other option than to start over.
Moving on lets modify the Ethernet port config. Change directory to
```console
$ cd /etc/sysconfig/network-scripts/
```
So now, I will paste the commands change the network configuration but **before we do that,** lets make a backup of the orignal config from the command below.
The command above backs up the old network config, just in case it does not works, you can always restore this config back.
```console
cp ifcfg-enp2s0 ifcfg-enp2s0.orignal
```
Do note that the **name of your network interface file may differ** from system to system depending on the ethernet port your system has. You might have more than one files there as well if your system has more than one ethernet port. Its up to you to figure out the right ethernet config file. There will also be a loopback interface file there. In the following steps we are going to create a bridge interface here as well.
Next up lets open the Ethernet port config file in favourite file editor.
```console
vi ifcfg-enp2s0
```
Lets paste the following config in this file. Again, **please fill in the correct name of  the network interface that you have**.
```config
TYPE=Ethernet
BOOTPROTO=static
DEVICE=enp2s0
ONBOOT=yes
BRIDGE=br0
```
Next we create a bridge interface configuration file. Again, open create a new file from your file editor.
```console
vi ifcfg-br0
```
And paste the following content in it.
```config
TYPE=Bridge
BOOTPROTO=static
DEVICE=br0
ONBOOT=yes
IPADDR=192.168.1.2
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
DNS1=192.168.1.1
```
Please make sure that you **enter the appropriate static IP configuration in the file above**. I am assuming that the previous configuration was also for a static IP. Cause an headless server without a staticIP is just as dumb as it sounds.
Now comes the scary part, we need to apply these settings by restarting the networking services. Again, as mentioned above please note that this might cause you to loose access to you server so either make sure that you have access to the server physically and are prepared to connect it back to a monitor and keyboard or if its a cloud server, you have no sensitive data on it and are ready to loose access to it forever.
```console
systemctl restart NetworkManager
```
Now if you are still able to connect to your server after this command that means that you have successfully added a bridge to your configuration. Congratulations.
### 4. Downloading the virtual OS
So, before we move on to launching a new VM, lets first download the image of a new VM and put it to the appropriate location. If you have SELinux enabled on you system, the directory enabled to hold virtualisation iso’s and hard disk images is /var/lib/libvirt/images. So lets change directory to /var/lib/libvirt/ and chmod this folder as well for easier access. Also, for this tutorial, we are going to download the Centos image for setting up a virtual machine of Centos7. You can get the images at the URL mentioned below.
```console
cd /var/lib/libvirt/
sudo chmod -R 777 images
cd images
wget http://mirror.nbrc.ac.in/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1611.iso
```
Replace the URL above with the URL close to your location. Make sure you download the minimal copy. This way you can only install the packages that you want.
### 5. Launching the virtual machine
So after we are done downloading the respective OS image, what comes is launching the VM. The sample command to launch a new VM is as follows
```console
virt-install --name centos7 --disk path="/var/lib/libvirt/images/cent7.img,size=20" --ram 2048 --vcpus 1 --os-type linux --os-variant centos7.0 --graphics vnc --cdrom "/var/lib/libvirt/images/CentOS-7-x86_64-Minimal-1511.iso" --network=bridge:br0
```
This is to launch a VM of Centos7. Also, we are creating the hard disk img file in the same directory as where our iso lies. Reason for not going to other directory is that SELinux will have to be enabled for that directory.

Rest parameters are **self explanatory**, name is name of VM. Disk path is path were the hard disk file will reside. Ram, vcpus and os-type are ram, virtual cpus and os-type respectively. Now, os-variant command tells KVM what flavour of OS are you running on top of KVM. This was a problematic area cause the command given by all blogs to get a list of all os-variant types is :
```console
virt-install --os-variant list
```
Now, this command is not working on Centos7.
To get the type of OS variants available via kvm, use the following command.
```console
 osinfo-query os
```
### 6. Viewing the machine's monitor as in installs
Now after this command your machine launches and just like you install any OS on a regular machine a regular boot-up screen shows up. In our case its the grub boot loader that shows up asking us to install Centos7. **But you cant see it anywhere.** Thats because you have connect to the instance via [VNCViewer](https://www.realvnc.com/en/connect/download/viewer/). Now port forwarding is another problem we might face here. Some of the vendors you use might not allow you to use any custom port you want and even if they do, the traffic will never be secure. Here you can use ssh. Ssh has an option to tunnel live traffic through it to localhost of you system. In simpler terms, with the correct commands, it will appear as if the X port of your remote system is Y port of you localhost ( or 127.0.0.1). This way you can forward the vnc port of your remote machine to your local machine and use vnc app to control the machine. The command to do that is
```console
ssh -L 127.0.0.1:5905:127.0.0.1:5900 -N username@www.xxx.yyy.zzz
```
Here, I am forwarding the traffic to port 5905. Not doing it to 5900 is a wise decision (on Mac) because my mac is already sharing screen on 5900.

After this you can view the screen of your remote VM via the VNC app.
That was all folks. I hope you found that useful. I know I will find it useful, especially next time when my server crashes.
