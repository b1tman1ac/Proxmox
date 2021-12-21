# ![Proxmox](images/2020/10/proxmox.png) Proxmox Home Lab Build

A how to guide to plan and track a Proxmox home lab build. This guide was initially written for Proxmox 6.2 but will be kept up to date as necessary.

# The Goals
These are my goals for this project.

- **Easy to build & maintain,** *in case I need to rebuild it from scratch*
- **Modular,** *I prefer to put things in a Container or VM rather then the base config*
- **Secure,** *Security built in*
- **Single host (no HA),** *this is a home lab build not Enterprise Production, I can afford some downtime and don't need High Availability*
- **Redundant Storage,** *In case of drive failure*
- **Offsite/Offhost Backups,** *In case of complete host failure*
- **Monitoring & Alerts,** *to know whats going on*
- **Automated,** *Automate the world*


# The Plan

<!-- TOC -->

1. [x] [1. Initial Install](#initialinstall)
2. [ ] [2. Base Config](#baseconfig)
3. [ ] [3. Setting up Networking](#networking)
4. [ ] [4. Building ZFS Storage](#zfs)
5. [ ] [5. Securing Proxmox](#security)
6. [ ] [6. Deploying the first VM :: PFSense](#pfsense)
7. [ ] [7. Deploying the first Container :: NAS](#nas)
8. [ ] [8. To Do](#todo)

<!-- /TOC -->

# Let's go . . .

## 1. Initial Install <a name="initialinstall"></a>

### 1.1 Hardware
###### 1.1.1 [Proxmox Official System Requirements][728bcac8]

  [728bcac8]: https://www.proxmox.com/en/proxmox-ve/requirements "Proxmox Official System Requirements"

  > ### Recommended Hardware
  >  - Intel EMT64 or AMD64 with Intel VT/AMD-V CPU flag.
  >  - Memory, minimum 2 GB for OS and Proxmox VE services. Plus designated memory for guests. For Ceph or ZFS additional memory is required, approximately 1 GB memory for every TB used storage.
  >  - Fast and redundant storage, best results with SSD disks.
  >  - OS storage: Hardware RAID with batteries protected write cache (“BBU”) or non-RAID with ZFS and SSD cache.
  >  - VM storage: For local storage use a hardware RAID with battery backed write cache (BBU) or non-RAID for ZFS. Neither ZFS nor Ceph are compatible with a hardware RAID controller. Shared and distributed storage is also possible.
  >  - Redundant Gbit NICs, additional NICs depending on the preferred storage technology and cluster setup – 10 Gbit and higher is also supported.
  >  - For PCI(e) passthrough a CPU with VT-d/AMD-d CPU flag is needed.

In addition to the official system requirements we have to take into account our use of ZFS as a file storage system so this should be noted as well.

![ZFS on Linux](images/2020/10/zfs-on-linux.png)

###### 1.1.2 [ZFS on Linux Hardware Recommendations][3b4c6a09]

  [3b4c6a09]: https://pve.proxmox.com/wiki/ZFS_on_Linux#_hardware "ZFS on Linux Hardware Recommendations"

  >  ZFS depends heavily on memory, so you need at least 8GB to start. In practice, use as much you can get for your hardware/budget. To prevent data corruption, we recommend the use of high quality ECC RAM.

  >  If you use a dedicated cache and/or log disk, you should use an enterprise class SSD (e.g. Intel SSD DC S3700 Series). This can increase the overall performance significantly.

  >  Important	Do not use ZFS on top of hardware controller which has its own cache management. ZFS needs to directly communicate with disks. An HBA adapter is the way to go, or something like LSI controller flashed in “IT” mode.

  >  If you are experimenting with an installation of Proxmox VE inside a VM (Nested Virtualization), don’t use virtio for disks of that VM, since they are not supported by ZFS. Use IDE or SCSI instead (works also with virtio SCSI controller type).

###### 1.1.3 Boot drive

A word on choosing an appropriate boot drive. While ESXI suggests using a USB key or SD Card to boot its minimal OS that is primarly because it loads the OS in memory after the initial boot and limits all writes to the boot drive after its loaded. FreeNAS as well, has been optimized to run on USB keys and limits writes to the boot drive. Proxmox, on the other hand, is based on Debian and has not optimized the OS for use in USB keys as a boot drive.

So you have a choice . . .

  1. Install Proxmox to a storage medium that is optimized for lots of writes. Example Harddrive, high endurance SSD/USB key.
  2. If installing to standard USB key make sure you have a mirrored boot drive so that when your boot drive fails (and it will) you have a backup ready to take over.
  3. Both 1 & 2

My choice in this regard is #3 because I have lots of harddrives in my server and I can use two in a mirrored vdev ZFS RAID1 Pool.

![Cisco C240 M4L](images/2020/10/cisco-c240-m4l.png)

###### 1.1.4 The Hardware  

|| Hardware |
|---|---|
|Server | Cisco C240 M4L |
|CPU | 2 x E5-2620 v3 (12 cores / 24 Threads)  |
|RAM   |  64GB ECC |
|Boot Drive  | 2 x 2TB  |
|Data Drive | 10 x 2TB  |
| HBA  | CISCO UCSC-MRAID12G  |
| Network  | 2 x 10GE, 2 x 1GE + IPMI|
| GPU | None  |

A note on the RAID card. I have set the drives to JBOD, disabled all caching & turned off the cards BIOS in its settings so its operating in HBA mode. Need to test whether or not the Host OS can see the drives directly without a cache in between as this, generally speaking, isn't a recommended card for ZFS. However, if I try to put in any 3rd party card that the bios doesn't recognize (re: non Cisco) the server will override the fan policy & turn the fans on full blast which converts my Home Lab Build into a Cessna on takeoff.

![Cessna on Takeoff](images/2020/10/cessna-on-takeoff.png)

### 1.2 Installation

I'm not going to re-write the installation process here as the official documentation is good in this regard and if not there are lots of resources on the web to help. Its pretty easy however and can be summed up by . . .

1. Download the ISO, https://www.proxmox.com/en/downloads/category/iso-images-pve
2. Write the ISO to a bootable USB flash drive using a tool like [Etcher][f77bac93]
3. Insert the USB key into your server and follow the prompts

[f77bac93]: https://etcher.io/ "Etcher"

_Although in my case I use the KVM on the IPMI port of my server to load the ISO to a virtual drive and boot the server off of that :D_

[Prepare Installation Media][be7d23ee]

  [be7d23ee]: https://pve.proxmox.com/pve-docs/pve-admin-guide.html#installation_prepare_media "Prepare Installation Media"

[Using the Proxmox VE Installer][9dc28777]

  [9dc28777]: https://pve.proxmox.com/pve-docs/pve-admin-guide.html#installation_installer "Using the Proxmox VE Installer"

Some Notes:

- Choose, `Install Proxmox VE` :)
- Because I am installing a ZFS root using a Mirrored or RAID1 setup I'll click `Options`, select `zfs (RAID1)` as my filesystem & pick my two boot drives.
- I leave the `advanced options` at the default.

_I know I am wasting a ton of space by not partitioning my boot disk into separate partitions for the Host, VM's, Containers & data but my goal is to keep things simple. So if I need to perform a complete rebuild or transfer my system to another host I can easily reinstall Proxmox on a new drive, resetup the base config/networking, connect my data drives, import my pools, VM's & containers and I'm pretty much back up and running from a boot drive failure._

- the rest is pretty straight forward

## 2. Base Config <a name="baseconfig"></a>

Great, you still with me :) This is where the fun part starts.

### 2.1 Confirm remote connectivity

###### 2.1.1 Log in via https and ssh  
https://<server_ip>:8006

AND

ssh root@<server_ip>

type in credentials:
  - Username: root
  - Password: <the_password_you_choose_during_install>

###### 2.1.2a You were able to successfully logged in  
Assuming you were able to log in then you should be good at this point to disconnect the monitor, keyboard and mouse from the server and complete the rest of the steps remotely.

###### 2.1.2b You were NOT able to successfully logged in  
If you weren't able to connect you need to figure out why, I would suggest to try pinging the <server_ip> and if it responds the problem may be the username/password. If it doesn't respond then issue is most likely network related, start by tracing the physical cable and check your network config after that.

Some useful links:

https://pve.proxmox.com/pve-docs/pve-admin-guide.html#sysadmin_network_configuration

https://unix.stackexchange.com/questions/128439/good-detailed-explanation-of-etc-network-interfaces-syntax

https://ubuntu.com/blog/if-youre-still-using-ifconfig-youre-living-in-the-past

Some useful commands:

`systemctl status networking`  
`systemctl restart networking`  
`ip address show`  
`ip link show`  

to make changes to the network Config . . .

`nano /etc/network/interfaces` _update config, save the file, then_  
`systemctl restart networking` _&_  
`ip address show` _to check if the changes had an effect & **check remote connectivity again**_  


### 2.2 Update Package respositories and perform an initial update

Because this is a Home Lab and not enterprise you need to update the package repos to reflect this.

###### 2.2.1 Add the no subscription repo to the sources list  
`nano /etc/apt/sources.list`

Add the no subscription repo . . .

> \# PVE pve-no-subscription repository provided by proxmox.com,
> \# NOT recommended for production use
> deb http://download.proxmox.com/debian/pve buster pve-no-subscription

###### 2.2.2 Remove the enterprise subscription
`nano /etc/apt/sources.list.d/pve-enterprise.list`

comment out the line in that file like this otherwise any apt-get update will fail because you don't have access to that repo . . .

> \#deb https://enterprise.proxmox.com/debian/pve buster pve-enterprise  

reference : https://pve.proxmox.com/wiki/Package_Repositories#sysadmin_no_subscription_repo

###### 2.2.3 Test updating via the Gui and cli  
You should now be able to update either via the gui or command line

`apt-get update && apt-get dist-upgrade -y`  

Perform a reboot...'just in case'  
**reboot**

reference : https://pve.proxmox.com/pve-docs/pve-admin-guide.html#_system_software_updates

### 2.3 Update NTP

###### 2.3.1 Add your time servers
`nano /etc/systemd/timesyncd.conf`  

_uncomment all of the lines and update the first line to look like this but pick your own time servers to sync too_

>[Time]  
NTP=time.nrc.ca time.chu.nrc.ca ntp.torix.ca  

###### 2.3.2 Restart the synchronization service  
`systemctl restart systemd-timesyncd`  

###### 2.3.3 Verify that your newly configured NTP servers are used by checking the journal  
`journalctl --since -1h -u systemd-timesyncd`  

reference:: https://pve.proxmox.com/pve-docs/pve-admin-guide.html#_time_synchronization  

### 2.4 Get Postfix to Send Notifications (Email) Externally

###### 2.4.1 Install libsasl2-modules  
`apt install libsasl2-modules`  

###### 2.4.2 Backup your current postfix configuration  
`cp /etc/postfix/main.cf /etc/postfix/main.cf.bak`  

###### 2.4.3 Modify your postfix configuration as follows:
`nano /etc/postfix/main.cf`  
###### Modify the line:  
`relayhost = [smtp.gmail.com]:587`  
###### Add the following lines to the end:  
```
smtp_sender_dependent_authentication = yes

sender_dependent_relayhost_maps = hash:/etc/postfix/sender_relayhost.hash
smtp_sasl_auth_enable = yes

smtp_sasl_password_maps = hash:/etc/postfix/sasl_auth.hash

smtp_sasl_security_options = noanonymous

smtp_use_tls = yes

smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt  
```

###### 2.4.4 Create your authorization hash file:  
>NOTE: if you have MFA configured you'll need to create an App password for your Gmail account and add that below as yourpassword  
https://support.google.com/accounts/answer/185833?hl=en

`echo [smtp.gmail.com]:587 your_username@gmail.com:yourpassword > /etc/postfix/sasl_auth.hash`  

###### 2.4.5 Create your sender_relayhost file (this makes sure that you always use your gmail as the sender:  
`echo your_username@gmail.com [smtp.gmail.com]:587 > /etc/postfix/sender_relayhost.hash`  

###### 2.4.6 Now postmap the files:  
`postmap /etc/postfix/sender_relayhost.hash`  
`postmap /etc/postfix/sasl_auth.hash`  

###### 2.4.7 Make sure to make your password only readable by root:  
`chmod 400 /etc/postfix/sasl_auth.*` 

###### 2.4.8 Restart Postfix:  
`postfix reload` OR `systemctl restart postfix.service`  

###### 2.4.9 Test:  
**Test from Postfix:**  
`systemctl status postfix.service`  
`echo "Test mail from postfix" | mail -s "Test Postfix" test@test.com`  

**Test from PVE:**  
`echo "test" | /usr/bin/pvemailforward`  

###### 2.4.10 Logs:  
`/var/log/mail.warn`  
`/var/log/mail.info`  

References:  
https://github.com/ShoGinn/homelab/wiki/Proxmox-PostFix---Email  
https://forum.proxmox.com/threads/get-postfix-to-send-notifications-email-externally.59940/

### 2.5  Setup notifications to a Slack channel using a webhook

###### 2.5.1 Setup a new incoming webhook in Slack by adding this app to a channel  
https://slack.com/apps/A0F7XDUAZ-incoming-webhooks  
Configure it how you want but copy down the web hook url for later, it will look something like this . . . 
`https://hooks.slack.com/services/d8a9das90d8/ds79d07asd0a/dff9sdf89dfdpivcs` **this is faked**  

###### 2.5.2  Install some required apps and modules for perl

`apt-get install dh-make-perl`  
`cpan Slack::WebHook`  

###### 2.5.3 Create a perl script which we will use to forward emails to the slack channel

`nano /usr/local/sbin/post2slack.pl`  

Add the following lines to the file . . .  

> \#!/usr/bin/perl -T  
>  
> \# https://metacpan.org/pod/Slack::WebHook  
> 
> use strict;  
> use warnings;  
> use Slack::WebHook;  
> 
> $ENV{'PATH'} = '/sbin:/bin:/usr/sbin:/usr/bin';  
> 
> my $hook = Slack::WebHook->new( 
               url => 'https://hooks.slack.com/services/fdsafdsfdsfds/ffdsasdfsdfsdf'  
> );  
>  
>  my $stdin_h = do { local $/; \<STDIN\> };  
>  
>  $hook->post_ok( $stdin_h );  

###### 2.5.4 Make the script executable  

`chmod 755 /usr/local/sbin/post2slack.pl`  

###### 2.5.5 Add the script to root's forwards files

`echo "|/usr/local/sbin/post2slack.pl" >> /root/.forward`  

###### 2.5.6 Test by sending a message to root  

`echo "Test mail from postfix sent to slack" | mail -s "Test Slack" root`  

###### 2.5.7 Alternatively send emails to a slack channel

If you have a paid slack account you can alternatively setup a email address that slack will monitor for incoming emails and post it to a slack channel  

This is the slack app you need to setup  
https://slack.com/apps/A0F81496D-email  

Add the slack email to the bottom of root's .forwards file  
`echo "foobar@example.com" >> /root/.forward`  

And test  
`echo "Test mail from postfix sent to slack" | mail -s "Test Slack" root`  

References:  
https://api.slack.com/messaging/webhooks  
https://metacpan.org/pod/Slack::WebHook  
https://forum.proxmox.com/threads/proxmox-alert-emails-can-you-automatically-cc-people.53332/  

## 3. Setting up Networking <a name="networking"></a>

### 3.0 Proxmox Networking Primer

If you really want to keep the network simple the basic linux bridge is perfectly fine. In fact in recent versions of Linux & Proxmox actually has most of the functionality you would need/want.  However the goals of this Homelab is not only to be _Simple_ but also _Modular_, _Secure_, _extensible_ & maybe optimized for _Speed_. For that we need Open vSwitch.

To Sum Up . . .  

Linux Bridge = LAN/Switch  
Linux Bond = Port-Channel  
Linux VLAN = Virtual LAN  
Linux Tap = Virtual Interface  

**OVS Bridge** = LAN/Switch with one or more _Ports_.  
**OVS Port** = A _Port_ in a _Bridge_. A _Port_ can have 1 or more _Interfaces_. A _Port_ with more then 1 _Interface_ is a Bonded _Port_.  
**OVS Interface** = An _Interface_ in a _Port_. An _Interface_ can be many types but the most basic ones are _System_, _Internal_ & _tap_.  

See `man ovs-vsctl` or `man ovs-vswitchd.conf.db` or `man ovs-vswitchd` for more details, it's really well documented.  

**References:**  
https://github.com/openvswitch/ovs/blob/master/debian/openvswitch-switch.README.Debian

###### 3.0.1 Let's go over the Basic Proxmox Linux Networking stack  


###### 3.0.1.1 Linux Bridge's
Layer 2 Switch connecting one or more physical or virtual interfaces together. Bridges direct traffic to the appropriate interfaces based on Mac Addresses so traffic goes directly from one host to another through the bridge.  

_Default Configuration using a Bridge via Proxmox Admin Guide 3.3.4_
![default-network-config](images/2020/10/default-network-config.png)

_Default Linux Bridge Example_  
_via the CLI_  
`nano /etc/network/interfaces`  
> auto eno1  
> iface eno1 inet manual  
> 
> auto vmbr0  
> iface vmbr0 inet manual  
> &nbsp;&nbsp;&nbsp;&nbsp;bridge_ports eno1  
> &nbsp;&nbsp;&nbsp;&nbsp;bridge_stp off  
> &nbsp;&nbsp;&nbsp;&nbsp;bridge_fd 0

_Default Linux Bridge Example with a static ip for the host_
> auto eno1  
> iface eno1 inet manual  
> 
> auto vmbr0  
> iface vmbr0 inet static  
> &nbsp;&nbsp;&nbsp;&nbsp;address 192.168.10.2  
> &nbsp;&nbsp;&nbsp;&nbsp;netmask 255.255.255.0  
> &nbsp;&nbsp;&nbsp;&nbsp;gateway 192.168.10.1  
> &nbsp;&nbsp;&nbsp;&nbsp;bridge_ports eno1  
> &nbsp;&nbsp;&nbsp;&nbsp;bridge_stp off  
> &nbsp;&nbsp;&nbsp;&nbsp;bridge_fd 0  

_via the GUI_  
![Linux-Bridge-GUI-Example](images/2020/10/linux-bridge-gui-example.png)

How to show the Mac Addresses of the interfaces on the bridge . . . 
`bridge fdb show`  
> 01:00:5e:00:00:fb dev rename4 master vmbr0  
01:00:5e:7f:ff:fa dev eno1 vlan 1 master vmbr0 permanent  
01:00:5e:7f:ff:fb dev eno1 master vmbr0 permanent  
40:2c:ff:ec:f8:ff dev eno1 master vmbr0  
00:0c:11:90:c6:11 dev eno1 master vmbr0  
01:00:5e:7f:ff:ff dev eno1 self permanent  
33:33:00:00:00:01 dev vmbr0 self permanent  

How to show the ARP table for the host . . .  
`ip neigh show`  
> 192.168.10.1 dev vmbr0 lladdr 00:0c:11:90:c6:11 STALE  
> 192.168.10.2 dev vmbr0 lladdr 40:2c:ff:ec:f8:ff REACHABLE  
> 192.168.10.100 dev vmbr0 lladdr 40:2c:ff:ec:f8:ff REACHABLE  

Other userful commands  
\# Check if interfaces are added to the bridge  
`brctl show vmbr0`  

\# Check if the interfaces are up and receiving traffic  
`ip link show`  

References:  
https://pve.proxmox.com/pve-docs/pve-admin-guide.html#sysadmin_network_configuration  

###### 3.0.1.2 Linux Bond's
Bonds join multiple interfaces together, think port-channel/Etherchannel, link-aggregation (LACP), NIC Teaming. When you create a bond both interfaces act as one and they don't need to have there own configuration section.

_Linux Bond Example_  
![Linux-Bond-Example](images/2020/10/linux-bond-example.png)
_via the CLI_  
`nano /etc/network/interfaces`  
> \# Auto is used to bring up the interfaces  
> \# manual defines the interface with no default configuration  
> auto eno1  
> iface eno1 inet manual  
> auto eno2  
> iface eno2 inet manual  
> 
> \# Bonding the interfaces together using 802.3ad (LACP)  
> auto bond0  
> iface bond0 inet manual  
> &nbsp;&nbsp;&nbsp;&nbsp;slaves eno1 eno2  
> &nbsp;&nbsp;&nbsp;&nbsp;bond_miimon 100  
> &nbsp;&nbsp;&nbsp;&nbsp;bond_mode 802.3ad  
> &nbsp;&nbsp;&nbsp;&nbsp;bond_xmit_hash_policy layer2+3  
> 
> \# Create a new bridge with the bonded port  
> auto vmbr0  
> iface vmbr0 inet manual  
> &nbsp;&nbsp;&nbsp;&nbsp;address 10.10.10.2  
> &nbsp;&nbsp;&nbsp;&nbsp;netmask 255.255.255.0  
> &nbsp;&nbsp;&nbsp;&nbsp;gateway 10.10.10.1  
> &nbsp;&nbsp;&nbsp;&nbsp;bridge_ports bond0  
> &nbsp;&nbsp;&nbsp;&nbsp;bridge_stp off  
> &nbsp;&nbsp;&nbsp;&nbsp;bridge_fd 0  

_via the GUI_  
![GUI-Linux-Bond](images/2020/10/gui-linux-bond.png)
![GUI-Linux-Bond-Bridge](images/2020/10/gui-linux-bond-bridge.png)

Other userful commands  
\# Check if interfaces are added to the bond  
`cat /proc/net/bonding/bond1`  

References:  
https://pve.proxmox.com/pve-docs/pve-admin-guide.html#sysadmin_network_configuration  
https://www.kernel.org/doc/Documentation/networking/bonding.txt  

###### 3.0.1.3 Linux VLAN's

A VLAN lets you divide up (subnet) your physical network/Interface/LAN/Bridge into separate logical ones. A benefit would be to create a guest wifi network and give friends access to the internet only (VLAN10) but nothing internal to your home network (VLAN20).


References:  
https://pve.proxmox.com/pve-docs/pve-admin-guide.html#sysadmin_network_configuration  

###### 3

### 3.1 Install Open vSwitch Switch package

From the ssh terminal install the OpenvSwitch Switch package otherwise the gui will complain that its not installed when you try to create an OVS Bridge . . .  

![Trying-to-create-ovs-bridge-error](images/2020/10/trying-to-create-ovs-bridge-error.png)

`apt update`  
`apt install openvswitch-switch`  

You can only support Linux Bridge or Open VSwitch, so choose wisely.

If you want to have OVS in the long run its best to do it now before you create any containers or VM's.

The easiest way to do this would be to edit /etc/network/interfaces directly and then restart Proxmox or reload network services but first we want to make a backup.

\# Make a backup  
`cp /etc/network/interfaces /etc/network/interfaces.bak`

\# Edit Networking  
`nano /etc/network/interfaces`

\# Replace the contents of the file with the below  
_OpenvSwitch Bridge Example_

> \# Loopback interface  
> &nbsp;&nbsp;&nbsp;&nbsp;auto lo  
> &nbsp;&nbsp;&nbsp;&nbsp;iface lo inet loopback  
>  
> \# Bridge for our eth0 physical interfaces and vlan virtual interfaces (our VMs will  
> \# also attach to this bridge)  
> allow-ovs vmbr0  
> iface vmbr0 inet manual  
> &nbsp;&nbsp;&nbsp;&nbsp;ovs_type OVSBridge  
>   \# NOTE: we MUST mention eno1, vlan11, vlan 12 and vlan999 even though each  
>   \# of them lists ovs_bridge vmbr0!  Not sure why it needs this  
>   \# kind of cross-referencing but it won't work without it!
>   ovs_ports eno1 vlan11 vlan12  
> 
> 
> \# Physical interface for traffic coming into the system.  Retag untagged  
> \# traffic into vlan 1, but pass through other tags.  
> auto eno1  
> allow-vmbr0 eno1  
> iface eno1 inet manual  
> &nbsp;&nbsp;&nbsp;&nbsp;ovs_bridge vmbr0  
> &nbsp;&nbsp;&nbsp;&nbsp;ovs_type OVSPort  
> &nbsp;&nbsp;&nbsp;&nbsp;ovs_options tag=11 vlan_mode=native-untagged  
> \# Alternatively if you want to also restrict what vlans are allowed through  
> \# you could use:  
> \# ovs_options tag=11 vlan_mode=native-untagged trunks=11,12  
>  
> \# Virtual interface to take advantage of originally untagged management traffic  
> allow-vmbr0 vlan11  
> iface vlan11 inet static  
> &nbsp;&nbsp;&nbsp;&nbsp;ovs_type OVSIntPort  
> &nbsp;&nbsp;&nbsp;&nbsp;ovs_bridge vmbr0  
> &nbsp;&nbsp;&nbsp;&nbsp;ovs_options tag=11  
> &nbsp;&nbsp;&nbsp;&nbsp;address 10.50.10.44  
> &nbsp;&nbsp;&nbsp;&nbsp;netmask 255.255.255.0  
> &nbsp;&nbsp;&nbsp;&nbsp;gateway 10.50.10.1  
> &nbsp;&nbsp;&nbsp;&nbsp;ovs_mtu 1500  

\# Either reload Proxmox or restart network   
\# If doing this via ssh, be careful to do this in one command  
`/etc/init.d/networking stop && /etc/init.d/networking start`  

References:  
https://pve.proxmox.com/wiki/Open_vSwitch  
https://www.hindawi.com/journals/jece/2016/5249421/  
https://www.actualtechmedia.com/wp-content/uploads/2018/01/CUMULUS-Understanding-Linux-Internetworking.pdf  
http://arthurchiao.art/blog/ovs-deep-dive-6-internal-port/  

## 4. Building ZFS Storage <a name="zfs"></a>

## 5. Securing Proxmox <a name="security"></a>

## 6. Deploying the first VM :: PFSense <a name="pfsense"></a>

## 7. Deploying the first Container :: NAS <a name="nas"></a>

## 8. To Do <a name="todo"></a>

??? https://wiki.debian.org/UnattendedUpgrades ???
