+++
title = "Virtualization - Setting up a lab with one PC and install Proxmox (Part 1 of 5)"
date = '2026-01-18T19:17:00Z'
draft = false
tags = ["proxmox","homelab","virsh"]
series = ["virtualization"]
categories = ["tutorials"]
+++

## Introduction

I have some cool experience deploying and configuring a Proxmox cluster with High Availability and fully redundant Ceph in production. I thought it'd be cool to share my insights. Since I have been meaning to redeploy my lab on my laptop to test some things, I thought this is a good time. To start, this post will be about setting up the lab environment and install one Proxmox instance. I'm going to write a few other tutorials as I go. You are free to use the content here as you see fit. I would appreciate being credited, but that's not a requirement.

This a drawing of a production-ready virtualization cluster I designed. Please note that this design works with whatever you prefer: Proxmox, XCP-ng, VMware, Linux. I'm using Proxmox because it's very approachable and free.

![Virtualization Cluster Physical Network](Virtualization_Cluster_Physical_Network.png)

It ticks all the boxes you'd need for a production environment:

- **High Availability:** 3 virtualization nodes capable of Corosync or whatever else you virtualization platform uses. You will notice that I deployed 2 virtual firewalls. With pfSense or opnsense you can also have the firewalls doing HA.
- **Redundancy:** The 3 virtualization nodes can have fully redundant Ceph. If you have enough NICs on the hosts and the switches, you can also have redundant network. The Hetzner vSwitch also offers a flexible solution if you prefer to approach this with just one firewall VM on HA: the IP will follow it seamlessly if it's moved to a different virtualization node.
- **Network segmentation:** Separate physical networks for different purposes. Good for security, great for performance.

A lot of Virtualization labs I see floating around in tutorials are built using a bunch of real hardware. While that's undeniably cool and wholesome, we not always have the luxury of getting a bunch of extra computers. To be honest, nowadays I think it's kind of wasteful in terms of energy, and also in terms of having idle PCs just for this if you end up powering them off. I think that Linux virtualization is mature enough that you can do really cool stuff with just one computer. In this series I will teach you to do precisely that.

#### Requirements

I'm not gonna lie: you need somewhat robust hardware for this. The steps I'm taking are on an 8c/16t AMD processor, with 96GB RAM and 2TB NVMe drives. The minimum I would recommend: 8c/16t, 64GB RAM, 1TB NVMe drive. Maybe you could get away with 32GB RAM and 512GB NVMe. This will work on Arch Linux, but it's easy to transport to another distribution. If you're following along, you could just translate the steps to your setup using your favorite AI tool. In the future I may add some notes on how to do this on Ubuntu and RHEL. For the networking part, I'm using `NetworkManager` with `nmcli`.

This is the lab we will be building, I think this is remarkably close to the production system:

![Virtual Lab Network](Virtual_Lab_Network.png)


## Setting Up the Environment

### 1. Setting up `/var/lib/libvirt`

If you go with the defaults, you could skip to #2. I, however, have a matter of personal preference: I like to do things from my `~/` directory, and on my machine I have a 2TB NVMe specifically allocated for `/home`. I thought I'd go with symlinks, but I had some issues with this a while ago. I did some research and found out that bind mounts are the best approach: you get the same filesystem location exposed at two paths. Avoids all the potential weirdness from using symlinks for this. This is where all of your virtualization files, which consume a lot of disk space, will reside.

```bash
# Create ~/libvirt
mkdir /home/${USER}/libvirt

# Create /var/lib/libvirt
sudo mkdir /var/lib/libvirt

# Add the entry to /etc/fstab
echo "/home/${USER}/libvirt /var/lib/libvirt none bind 0 0" | sudo tee -a /etc/fstab

# Reload so that you can apply
systemctl daemon-reload

# Mount it
mount -a

# Test it
touch ~/libvirt/testfile
ls /var/lib/libvirt/testfile
```

My `/home` is encrypted on a separate volume as `/`. As long as the new mount is at the bottom of `/etc/fstab`, which it should, it should be handled automatically if you followed my steps.

For this to work well, we need to manage directory permissions with ACLs, so that all the necessary users can operate in the directory:

```bash
# Ensure ~/ is traversable
chmod 755 ~/

# Set base permissions on libvirt dir
chmod 775 ~/libvirt

# Install the ACL package
sudo pacman -S acl

# Apply rwx permissions for your user to all existing files/dirs recursively
setfacl -R -m u:$(whoami):rwx ~/libvirt

# Set default ACL - new files/dirs created inside inherit these permission
setfacl -R -d -m u:$(whoami):rwx ~/libvirt
```

There will be a bunch of permission steps later if you are doing this.

### 2. Configuring the Virtualization Suite

#### Setting up the System

We will now install the necessary virtualization packages:

```bash
sudo pacman -S libvirt qemu-full virt-manager dnsmasq ebtables dmidecode
```

Once that's done, add your user to the `libvirt` group:

```bash
sudo usermod -aG libvirt $(whoami)
```

Then enable and start the `libvirtd` service:

```bash
sudo systemctl enable --now libvirtd
```

And check if no errors with `systemctl status libvirtd`, it should show something like this:

```bash
● libvirtd.service - libvirt legacy monolithic daemon
     Loaded: loaded (/usr/lib/systemd/system/libvirtd.service; enabled; preset: disabled)
     Active: active (running) since Sun 2026-01-18 17:09:03 GMT; 5s ago
 Invocation: 43007c7f51c445a485af01f5f3ba08f5
TriggeredBy: ● libvirtd.socket
             ● libvirtd-admin.socket
             ● libvirtd-ro.socket
       Docs: man:libvirtd(8)
             https://libvirt.org/
   Main PID: 11650 (libvirtd)
      Tasks: 21 (limit: 32768)
     Memory: 8.6M (peak: 10M)
        CPU: 695ms
     CGroup: /system.slice/libvirtd.service
             └─11650 /usr/bin/libvirtd --timeout 120

Jan 18 17:09:03 laptop systemd[1]: Starting libvirt legacy monolithic daemon...
Jan 18 17:09:03 laptop systemd[1]: Started libvirt legacy monolithic daemon.
```

Also check that the daemon is responding to commands with `virsh list -all`, it should return an empty table:

```bash
virsh list --all
 Id   Name   State
--------------------
```

Now you need to configure your user to connect to `qemu:///system` where the real networking happens, rather than `qemu:///session`:

Add this line to your `.bashrc` file and reload your shell:

```bash
echo 'export LIBVIRT_DEFAULT_URI="qemu:///system"' >> ~/.bashrc
source ~/.bashrc
```

Test with `virsh uri`, if it shows `qemu:///system`, you're good to go:

```bash
virsh uri
qemu:///system
```

#### Setting up the Networks

First we will set up the default NAT network with the settings we desire. It's likely already set up, so run:

```bash
virsh net-stop default
virsh net-destroy default
virsh net-undefine default
```

Then go to your `/var/lib/libvirt` directory (or `~/libvirt` if you followed the section #1) and `mkdir temp`. This is where we will leave template files. Once there `vim default.xml` and add this:

```xml
<network>
  <name>default</name>
  <forward mode="nat"/>
  <bridge name="virbr0"/>
  <ip address="192.168.122.254" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.122.1" end="192.168.122.253"/>
    </dhcp>
  </ip>
</network>
```

Then run the following commands to activate the network:

```bash
virsh net-define default.xml
virsh net-start default
virsh net-autostart default
virsh net-list --all
```

We are going with a class C `192.168.122.0/24` subnet because it's already the default, and because it's very distinct from class A, which is what we will be using for the other subnets. Check that the interface `virbr0` exists:

```bash
ip a | grep virbr0
6: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc htb state DOWN group default qlen 1000
    inet 192.168.122.254/24 brd 192.168.122.255 scope global virbr0
```

You will notice that the host, which will also be the gateway to the Proxmox VMs, is using IP ending in `.254`. That's a personal preference, as I like when we have `vm-pve1`, `vm-pve2` and `vm-pve3` to have their IPs ending in `.1`, `.2` and `.3` respectively. Keeps everything easy to remember and tidy.

We will now configure the bridges which will simulate a virtual switch with separate VLANs. This is how we will configure the subnets:

- **HA/Corosync:** will be on subnet `10.0.253.0/24` and bridge `ha-br`.
- **Ceph:** will be on subnet `10.0.254.0/24` and bridge `ceph-br`.
- **Storage:** will be on subnet `10.0.255.0/24` and bridge `st-br`.
- **VMs:** These are the VMs which will run inside the virtual Proxmox nodes, they will be on subnet `10.0.1.0/24` and bridge `vm-br`.

The logic for these addresses is that we would "backbone" infrastructure to be distinct from where our VMs are. I also choose IPs which will *probably* not collide with whatever configuration you have at home. I also think that is a good practice for production environments, to choose private IPs which will likely not conflict with home routers. In the case of the bridge `vm-br`: in a production environment you would probably have multiple VLANs and subnets there rather than just one, potentially servicing thousands of devices.

Now you can copy the `.xml` bellow into your temp directory, using `vim` or whatever you prefer to create each file:

**file: `ha-br.xml`**
```xml
<network>
  <name>ha-br</name>
  <bridge name="ha-br"/>
  <ip address="10.0.253.254" netmask="255.255.255.0"/>
</network>
```

**file: `ceph-br.xml`**
```xml
<network>
  <name>ceph-br</name>
  <bridge name="ceph-br"/>
  <ip address="10.0.254.254" netmask="255.255.255.0"/>
</network>
```

**file: `st-br.xml`**
```xml
<network>
  <name>st-br</name>
  <bridge name="st-br"/>
  <ip address="10.0.255.254" netmask="255.255.255.0"/>
</network>
```

**file: `vm-br.xml`**
```xml
<network>
  <name>vm-br</name>
  <bridge name="vm-br"/>
  <ip address="10.0.1.254" netmask="255.255.255.0"/>
</network>
```

After you're done, time to enable all of them. Let's use a `for` loop to save us some time:

```bash
for net in ha-br ceph-br st-br vm-br; do
  virsh net-define ${net}.xml
  virsh net-start ${net}
  virsh net-autostart ${net}
done
```

Now when you check with `virsh net-list --all` this is what you should see:

```bash
❯ virsh net-list --all
 Name      State    Autostart   Persistent
--------------------------------------------
 ceph-br   active   yes         yes
 default   active   yes         yes
 ha-br     active   yes         yes
 st-br     active   yes         yes
 vm-br     active   yes         yes
```

And another quick check on your actual network interfaces:

```bash
for net in ha-br ceph-br st-br vm-br; do
  ip a | grep ${net}
done
```

Which should give you something like this:

```bash
6: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc htb state DOWN group default qlen 1000
    inet 192.168.122.254/24 brd 192.168.122.255 scope global virbr0
7: ha-br: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc htb state DOWN group default qlen 1000
    inet 10.0.253.254/24 brd 10.0.253.255 scope global ha-br
8: ceph-br: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc htb state DOWN group default qlen 1000
    inet 10.0.254.254/24 brd 10.0.254.255 scope global ceph-br
9: st-br: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc htb state DOWN group default qlen 1000
    inet 10.0.255.254/24 brd 10.0.255.255 scope global st-br
10: vm-br: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc htb state DOWN group default qlen 1000
    inet 10.0.1.254/24 brd 10.0.1.255 scope global vm-br
```

This is it for network creation. I hate to end it in a cliffhanger, but such is life. Next week we will be configuring storage and creating the Proxmox VMs. Maybe we will will also be installing one instance. If you have some experience you could already go ahead and experiment with what we've built so far.

I hope to see you again soon!