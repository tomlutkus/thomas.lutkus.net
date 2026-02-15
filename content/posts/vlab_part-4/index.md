+++
title = "Virtual Proxmox Lab (Part 4) - Clustering: Corosync & Ceph"
date = '2026-02-16T19:00:00Z'
draft = false
tags = ["proxmox","homelab","virsh"]
series = ["virtualization"]
categories = ["tutorials"]
+++

## 1. Introduction

Last week I did some work on the blog's design, and I was a bit under the water, so I ended up forfeiting working on this blog post. Today I will start by doing a quick recap on how to get the lab deployed by using my GitHub files. It assumes that you have the Proxmox automated install `.iso` per Part 1. Then I will proceed to explain the CLI procedures for Corosync and Ceph. This part is way shorter and much more straightforward than what we've been doing before. I was aiming to get you to experience the process of scripting and figuring out what to do. Now we're more practical. I don't know if at any point I will script this. It's possible. Maybe partially. You have a freedom to make mistakes and redeploy the whole lab in 3 minutes, use it.

![Virtual Lab Part 4 Illustration](vlab_part-4_illustration.png)

## 2. Recap: Setting up the Lab

First, create the directory to receive the project:

```bash
mkdir -p ~/Projects
```

Then pull my GitHub repo:

```bash
cd ~/Projects
git clone https://github.com/tomlutkus/virtual-proxmox-lab
```

Next, make sure that you have the proper ACLs in your `/var/lib/libvirt` directory (Arch in my case, adjust to the package manager of your distro), so that we don't need to be operating as root:

```bash
# Install the ACL package
sudo pacman -S acl
# Apply rwx permissions for your user to all existing files/dirs recursively
sudo setfacl -R -m u:$(whoami):rwx,m::rwx /var/lib/libvirt/
# Set default ACL - new files/dirs created inside inherit these permission
sudo setfacl -R -d -m u:$(whoami):rwx,m::rwx /var/lib/libvirt/
```

Make sure that your user is configured to connect to `qemu:///system`:

```bash
echo 'export LIBVIRT_DEFAULT_URI="qemu:///system"' >> ~/.bashrc
source ~/.bashrc
```

**You must have the `.iso` files already. the full steps to create them are in part 1, this is just a short recap of what you need to get going with the scripts.**

Now, deploy the lab:

```bash
cd ~/Projects/virtual-proxmox-lab/host/scripts
chmod +x deploy_lab.sh
./deploy_lab.sh
```

Watch the install run:

```bash
for N in pve-0{1..3}; do virt-viewer -w -r "${N}" & done
```

When the VMs power off, start them again and wait for them to give you the login prompt:

```bash
for N in pve-0{1..3}; do virsh start "${N}"; done
```

## 3. Corosync HA

This is where we wanted to get with today's post. You now understand how to do an automated deployment of a Proxmox Virtual Cluster, and you have the ability to deploy and dismantle it at will, precisely in the same way. There are a lot of tutorials covering the next steps through the GUI, and I find that boring at this point. We should want to live in the CLI: everything is slow and so much clicking around the GUI. It's more prone to mistakes. I will grant it: it's great for getting started and understanding how it works.

### 3.1 Checking that the Nodes Talk

The first thing is to test that the nodes talk to each other. I will go the lazy path and test from just `pve-01` and assume that everything is working, because it probably will, at this point:

```bash
ping -c 2 10.0.253.1
ping -c 2 10.0.253.2
ping -c 2 10.0.253.3
```

### 3.2 Creating the Corosync Cluster

We will be calling it `lab-cluster`, but name it however you like. We will let the nodes reference each other via the **wan** interface on the subnet `192.168.122.0/24`, but in production you'd switch that to a management internal interface.

1. Run the following command on `pve-01` to initialize the cluster:

```bash
pvecm create lab-cluster --link0 10.0.253.1
```

2. Then check if it's up with `pvecm status` for a similar output as bellow:

```bash
root@pve-01:~# pvecm status
Cluster information
-------------------
Name:             lab-cluster
Config Version:   1
Transport:        knet
Secure auth:      on

Quorum information
------------------
Date:             Sun Feb 15 17:55:29 2026
Quorum provider:  corosync_votequorum
Nodes:            1
Node ID:          0x00000001
Ring ID:          1.5
Quorate:          Yes

Votequorum information
----------------------
Expected votes:   1
Highest expected: 1
Total votes:      1
Quorum:           1  
Flags:            Quorate 

Membership information
----------------------
    Nodeid      Votes Name
0x00000001          1 10.0.253.1 (local)
```

3. SSH into `pve-02` run the following, replying `yes` to the prompt

```bash
pvecm add 192.168.122.1 --link0 10.0.253.2
```

4. SSH into `pve-03` run the following, replying `yes` to the prompt

```bash
pvecm add 192.168.122.1 --link0 10.0.253.3
```

5. Back into `pve-01` run check the cluster with `pvecm status` and `corosync-cfgtool -s`:

```root@pve-01:~# pvecm status
Cluster information
-------------------
Name:             lab-cluster
Config Version:   3
Transport:        knet
Secure auth:      on

Quorum information
------------------
Date:             Sun Feb 15 17:57:42 2026
Quorum provider:  corosync_votequorum
Nodes:            3
Node ID:          0x00000001
Ring ID:          1.d
Quorate:          Yes

Votequorum information
----------------------
Expected votes:   3
Highest expected: 3
Total votes:      3
Quorum:           2  
Flags:            Quorate 

Membership information
----------------------
    Nodeid      Votes Name
0x00000001          1 10.0.253.1 (local)
0x00000002          1 10.0.253.2
0x00000003          1 10.0.253.3
root@pve-01:~# corosync-cfgtool -s
Local node ID 1, transport knet
LINK ID 0 udp
	addr	= 10.0.253.1
	status:
		nodeid:          1:	localhost
		nodeid:          2:	connected
		nodeid:          3:	connected
```

If your output is similar to mine, your Corosync is now fully configured.

## 4. Ceph

I mentioned before "fully redundant". What I mean by that is not that each node is capable of running entirely alone, but that the cluster will handle one node loss. If 2 nodes are lost, the data protection is fully redundant, but it will block all I/O to protect data integrity, until the lost node(s) are back, then Ceph resumes and re-syncs automatically. This is the default `size 3, min_size 2` configuration.

### 4.1 Install Ceph

In each of the nodes `pve-01`, `pve-02` and `pve-03`, run the following command to install Ceph:

```bash
pveceph install --repository no-subscription
```

Answer yes to any prompt asking to about continuing.

Then proceed on node `pve-01` run this to initialize Ceph:

```bash
pveceph init --network 10.0.254.0/24
```

## 4.2 Add MONs and MGRs

In each of the nodes `pve-01`, `pve-02` and `pve-03` proceed to add a MON and MGR on each:

```bash
pveceph mon create
pveceph mgr create
```

## 4.3 Create OSDs and Pool

Now let's give shape to the whole thing. Create the OSDs out of the physical disks. First, check with `lsblk`:

```bash
root@pve-01:~# lsblk
NAME               MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                  8:0    0   32G  0 disk 
├─sda1               8:1    0 1007K  0 part 
├─sda2               8:2    0  512M  0 part 
└─sda3               8:3    0 31.5G  0 part 
  ├─pve-swap       252:2    0  3.9G  0 lvm  [SWAP]
  ├─pve-root       252:3    0 13.8G  0 lvm  /
  ├─pve-data_tmeta 252:4    0    1G  0 lvm  
  │ └─pve-data     252:6    0 11.8G  0 lvm  
  └─pve-data_tdata 252:5    0 11.8G  0 lvm  
    └─pve-data     252:6    0 11.8G  0 lvm  
sdb                  8:16   0  150G  0 disk 
sdc                  8:32   0  150G  0 disk 
sr0                 11:0    1 1024M  0 rom
```

In my case, the drives `/dev/sdb` and `/dev/sdc` are the ones with 150GB that we set aside for Ceph. Make sure that yours are too in each node, otherwise adapt. Run on each node:

```bash
pveceph osd create /dev/sdb
pveceph osd create /dev/sdc
```

Now create the pool:

```bash
pveceph pool create ceph-pool --pg_autoscale_mode on
```

Then add it as a Proxmox storage backend:

```bash
pvesm add rbd ceph-pool --pool ceph-pool --content images,rootdir
```

### 4.4 Validate

Let's make sure everything is healthy:

```bash
root@pve-01:~# ceph -s
  cluster:
    id:     193abaf8-c5a6-432b-b4ea-d86fb2587672
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum pve-01,pve-02,pve-03 (age 4m)
    mgr: pve-01(active, since 4m), standbys: pve-02, pve-03
    osd: 6 osds: 6 up (since 2m), 6 in (since 2m)
 
  data:
    pools:   2 pools, 129 pgs
    objects: 2 objects, 577 KiB
    usage:   161 MiB used, 900 GiB / 900 GiB avail
    pgs:     129 active+clean
```

All 3 MONs in quorum, 1 active MGR with 2 standbys, 6 OSDs up and in (2 per node), all PGs `active+clean`. You can also verify the OSD distribution:

```bash
root@pve-01:~# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME        STATUS  REWEIGHT  PRI-AFF
-1         0.87900  root default                              
-3         0.29300      host pve-01                           
 0    hdd  0.14650          osd.0        up   1.00000  1.00000
 1    hdd  0.14650          osd.1        up   1.00000  1.00000
-5         0.29300      host pve-02                           
 2    hdd  0.14650          osd.2        up   1.00000  1.00000
 3    hdd  0.14650          osd.3        up   1.00000  1.00000
-7         0.29300      host pve-03                           
 4    hdd  0.14650          osd.4        up   1.00000  1.00000
 5    hdd  0.14650          osd.5        up   1.00000  1.00000
```

Finally, confirm the replication settings:

```bash
ceph osd pool get ceph-pool size
ceph osd pool get ceph-pool min_size
```

You should see `size: 3` and `min_size: 2`. Every object is replicated across all 3 nodes. Lose 1 node and the cluster keeps running. Lose 2 and I/O blocks to protect the data, but nothing is lost: bring the node(s) back and Ceph re-syncs automatically.

## 5. Conclusion

That's it. Corosync was 3 commands. Ceph was a handful more. The cluster is fully redundant: data is triply replicated across nodes with automatic failover for monitors and managers. What took us several blog posts to automate and deploy took minutes to cluster.

If you want to practice it, you can leverage the `destroy_lab.sh` script, redeploy and master the commands.

In Part 5 we'll deploy the Proxmox Backup Server and tie it into the cluster. See you then.