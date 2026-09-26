# VLAN-Migration-and-Proxmox-Cluster

## Overview:

In this project I wanted to do two things. The first was a VLAN migration. Before this project I had two VLANs on my home network, Public and Private. Initially, I had public for friends and family visiting, and Private for devices that were trusted. The issue with this is that others that live in the house have access to private and sometimes add devices to the Private network that I don’t know about and that could be possibly infected with malware. If an infected device was on the private network, an attacker might be able to use it to reach my home lab with tools like Nmap and might attempt to breach something like my Proxmox server through brute forcing the password or some other exploit. Due to this, I wanted to create a third VLAN, Management, which I had complete control over what devices live on it, and can change/tighten things like firewall rules at any time without any effect on users on the Public or Private network. The second part of this project is configuring a Proxmox cluster. I’ll explain a bit more about what a cluster is when we get to section two, but for now it’s essentially a way to manage multiple physical nodes or servers using one Web UI. I wanted to do this because I was recently gifted a pc that I plan to use for upcoming projects, but I want to combine it with my existing infrastructure for events such as fail over requiring high availability, or migration of VMs/containers incase of something like a hardware failure.
Section 1: VLAN Migration

## Section 1: VLAN Migration

## Step 1.

First, I entered the Unifi Web UI and went to Settings > Networks > Create New. I named this network Management_VLAN, set the IPv4 address as 192.168.30.1/24, set the VLAN ID to 30, and turned off Auto-Scale Network so I could change the DHCP Range from 192.168.30.100 to 192.168.30.150 (Figure 1.1). Auto-Scale Network is a feature in Unifi that automatically expands the local subnets size and DHCP range when a network runs low on available IP addresses. Since I would be manually assigning IPs to essentially every device on this VLAN, this is not really something to be concerned about. Under DNS settings, I toggled off the Auto DNS Server option and added the current IP of my pi-hole, 192.168.22.199 which is currently on the Private VLAN. Once I swap it over, I will change the IP here. I then clicked create to add the new network.

<img width="566" height="563" alt="image" src="https://github.com/user-attachments/assets/bebe984e-789e-4eff-bb15-c544c7e2b9a9" />

Figure 1.1

## Step 2.

Next, I needed to create a new zone for the Management VLAN. In Unifi, a zone is a group of networks that get the same trust level. This means that instead of writing firewall rules between every VLAN, Unifi has you write rules between zones. Moving the management VLAN to a new zone means that the VLANs in the Internal zone like Default, Guest, and Private cannot access Management, and I don’t need to write individual rules limiting this. I created the new zone in Settings > Zones > Create Zone, naming it Management, and selecting the Maangement_VLAN network (Figure 1.2). It’s also important to note in the Zone Matrix the settings for Management are Allow All under External and Gateway, and Black All for everything else (Internal, VPN, Hotspot, DMZ, and Management). This matrix essentially means the router can talk to devices in the VLAN, and it only replies with traffic to connections started from inside the management VLAN, not new connections from the internet. Everything else including communication from other internal VLANs is blocked.

<img width="975" height="463" alt="image" src="https://github.com/user-attachments/assets/6fb29724-d941-41d4-b325-45880fd2f931" />

Figure 1.2

## Step 3.

After creating the new zone, I opened Proxmox to quickly save backups of /network/interfaces and /hosts. I did this using the commands “# cp /etc/network/interfaces{,.bak}”, and “# cp /etc/hosts{,.bak}”. The last section of these commands, {,.bak}, is a shell shortcut called brace expansion. What the brace expansion does in this case is copy interfaces and hosts to backup files interfaces.bak and hosts.bak. So, without the brace expansion the first command would look like “# cp /etc/network/interfaces /etc/network/interfaces.bak” (Figure 1.3). 

<img width="975" height="592" alt="image" src="https://github.com/user-attachments/assets/4094299c-630d-4d25-a044-9794e3bb2a1b" />

Figure 1. 3

## Step 4.

In that same shell I also made a full backup of both the dragonwilds and gitea containers using the command “# vzdump 100 101 –storage zfs-backups –mode snapshot”. This put both files into /storage/proxmox-backups/dump/ on the ZFS mirror under zfs-backups (Figure 1.4). This means that if an error occurs while migrating Proxmox or a container to the new VLAN and I need to roll back a container, I can do so using the command “# pct restore <id> <archive> --force”.

<img width="975" height="789" alt="image" src="https://github.com/user-attachments/assets/ded30591-e399-4130-80e2-355978d8bd0e" />

Figure 1.4

## Step 5.

Back in the Unifi Web UI, I navigated to Settings > Networks and selected Private_VLAN to change the Lease time to 600 sec instead of 86400 sec (Figure 1.5). I did this because one of the things I would be migrating to the new VLAN is a Raspberry Pi that I use for both Pi-hole and as my DNS server. Reducing the lease time makes the device check back in more often to the pi to renew its leased IP. This is important because once I migrate the Pi to be on the management VLAN, I’ll have to change its IP. This means that after the Pi moves, the devices on the network will keep querying a dead address and have no working DNS until its lease is renewed, usually halfway through the lease time which if not changed would be up to 12 hours. I changed this temporarily on both the Guest and Private VLAN.

<img width="544" height="255" alt="image" src="https://github.com/user-attachments/assets/5f4494f1-439d-47a1-a3d3-c5b8dc76ac14" />

Figure 1.5

## Step 6. 

After reducing the lease time, I navigated to Settings > Policy Table > Create New Policy, to add a firewall rule allowing the Management VLAN to access the other Internal VLANs in one direction. I did this so I would be able to access devices and containers on the Private VLAN from my PC which would soon be on the Management VLAN. I named the policy Allow Management to Internal, set the Source Zone to Any device and Any port on Management, and the Destination Zone to Any device and Anny port on an Internal VLAN (Figure 1.6). I also set the Action to Allow and added the policy. It’s also noteworthy that after returning to Zone Matrix, the Internal section under Management swapped from Block All to Allow All due to this rule, and the Management field for Internal is set to Allow Return. This means that Internal devices on the Private and Guest network can reply to connections from the management VLAN but cannot start new connections into the management VLAN themselves. (Figure 1.7).

<img width="602" height="1283" alt="image" src="https://github.com/user-attachments/assets/2c5330b8-95f1-41cc-81b4-b53701bd2ca8" />

Figure 1.6

<img width="975" height="281" alt="image" src="https://github.com/user-attachments/assets/fe01c22a-e540-4ce0-aa64-40054409e9d4" />

Figure 1.7

## Step 7.

Back in the Proxmox Web UI, I navigated to Datacenter > Firewall > Add and made a rule to allow traffic from the new management VLAN, with Directions set to in, Action set to ACCEPT, Source set to 192.168.30.0/24, and the Enable box ticked, then OK (Figure 1.8).

<img width="975" height="543" alt="image" src="https://github.com/user-attachments/assets/9b8731b3-c768-49c2-b296-2c9e8753ac9e" />

Figure 1.8

## Step 8.

In the Unifi Web UI, I navigated to Unifi Devices > Dream Machine > Port Manager, and selected Port 2 which is my PCs connection. I then changed the Native VLAN from Private_VLAN (22) to Management_VLAN (30) and applied changes (Figure 1.9). After unplugging and replugging my PC into Port 2 and checking “# ipconfig /all” in command prompt, I confirmed the PC was given an IP in the management VLAN. After this confirmation, I also made sure I could still access the Web UIs of pve01, Gitea, Pi-hole, and Unifi.

<img width="591" height="369" alt="image" src="https://github.com/user-attachments/assets/da49f362-bc5e-4d23-8d81-b0ee002e914c" />

Figure 1.9

## Step 9.

Back in the Unifi Web UI, I also set my PCs IP statically. I did this in Client Devices, selecting the PC, going to settings and ticking the Fixed IP Address box, setting it to statically use the IP 192.168.30.172, and applied the changes (Figure 1.10).

<img width="602" height="377" alt="image" src="https://github.com/user-attachments/assets/4c42be33-e2fb-4f31-bc81-031f35c65aa5" />

Figure 1.10

## Step 10.

The next thing I did was connect to the Pi using “# ssh justinlawitz@192.168.22.199” in command prompt after running it as administrator. Once connected, I ran “# ip -br addr” to confirm eth0 held the IP 192.168.22.199. While doing this I saw that wlan0, which is Wi-Fi, was given an IP for fallback in case of loss of connection using eth0. Before changing the IP of the Pi I wanted to turn this off, so it did not at some point fall back to the Private VLAN via wifi and cause issues. I did this by first running “# echo $SSH_CONNECTION” to confirm I was connected through eth0s IP instead of wlan0, then “# nmcli connection show” to find the Wi-Fi profile and its name, “# sudo nmcli connection delete “netplan-wlan0-LaMurphy Private” to remove the profile and verified the deletion with “# nmcli connection show” again (Figure 1.11).

<img width="975" height="329" alt="image" src="https://github.com/user-attachments/assets/fd0a0486-50b9-4ac0-841c-55d8b13aa343" />

Figure 1.11

## Step 11.

Returning to the Unifi Web UI, I first went to Client Devices and selected the pihole to check which port of the switch it was connected to and then went to Unifi Devices and selected the switch (USW Ultra). On Port 1 where the pihole was connected, I changed the Native VLAN to management and then returned to Client Devices to change the IP of the pihole to statically use 192.168.30.199. After rebooting the Pi, I confirmed I was able to ssh back in using the new IP, (ssh justinlawitz@192.168.30.199) (Figure 1.12). I also confirmed it was still operating as expected by using “# nslookup doubleclick.net 192.168.30.199” to confirm domains serving ads were still being blocked, and that I could login to the Web UI at https://192.168.30.199/admin/login (Figure 1.13). 

<img width="972" height="130" alt="image" src="https://github.com/user-attachments/assets/6d4bf9ea-388e-4da5-9a7a-4a6489d98f08" />

Figure 12

<img width="956" height="252" alt="image" src="https://github.com/user-attachments/assets/be030e24-ef6e-40d8-8752-1d26f800d416" />

Figure 1.13

## Step 12.

Back in the Unifi Web UI, I navigated to Settings > Policy Table, and in the Allow Guest to Pi-hole and Allow Private to Pi-hole rules I set when first configuring the Pi, I edited the Destination Zones to allow traffic to the Management zone at the new IP 192.168.30.199 (Figure 1.14). In Settings > Networks, I also changed the DNS Server IP on the Guest, Private, and new Management VLAN to use 192.168.30.199 by selecting each and clicking Edit on the DNS Server option.

<img width="580" height="891" alt="image" src="https://github.com/user-attachments/assets/80f5a313-a284-4bc4-a8fb-6dd055962247" />

Figure 1.14

## Step 13.

I next needed to migrate pve01 to the Management VLAN. Before that I also changed the DNS server being used to the new address in pve01 > DNS (Figure 1.15). I confirmed this change saved by opening the Shell and using “# nslookup google.com” and “# nslookup doubleclick.net” to make sure it could connect trusted domains and would block ad serving domains.

<img width="777" height="594" alt="image" src="https://github.com/user-attachments/assets/71c8ecb0-0dfd-43e8-9252-c20eb45885b2" />

Figure 1.15

## Step 14.

In pve01s Shell, I next ran “# nano /etc/network/interfaces” to edit the file, and under vmbr0 changed the address to 192.168.30.92/24, changed the gateway to 192.168.30.1, and under the “bridge-fd 0” line added the lines “bridge-vlan-aware yes” and “bridge-vids 2-4094”. To go a bit more into what all this means and does, vmbr0 is a virtual network switch that lives inside Proxmox itself. A bridge in networking is the same thing as a basic switch. There are three things connected to this bridge, the Optiplex’s physical ethernet port, which is the uplink to the real switch, each container’s virtual network card like a device is plugged into a port, and pve01’s own IP address. The first line, “bridge-vlan-aware yes”, essentially turns this switch or bridge from a basic switch into a managed one that understands VLAN tags. It can carry frames tagged with different VLAN IDs on the same cable, and it can give each virtual port its own tag. For instance, I will be keeping the dragonwilds container on the private network instead of management due to the nature of it being deliberately exposed to the internet on the UDP 7777 port. If the server ever gets exploited, I do not want an attacker to land in the management VLAN. The VLAN tag of 22 on CT 101 (dragonwilds container) means the bridge stamps 22 on dragonwilds traffic as it leaves, and the physical switch reads the label and treats those frames as Private VLAN traffic, so the router handles them through its VLAN 22 interface at 192.168.22.1. The second line, “bridge-vids 2-4094”, is the list of VLAN IDs the bridge will pass, allowing every VLAN other than 1, which is used for untagged or default traffic that the host and Gitea use. After editing the interfaces file, I confirmed the changes saved by using “# cat /etc/network/interfaces” (Figure 1.16).

<img width="975" height="595" alt="image" src="https://github.com/user-attachments/assets/7f037630-cefc-4200-a449-3e7f59fb7b41" />

Figure 1.16

## Step 15.

I also needed to edit the /etc/hosts file to use the IP using “# nano /etc/hosts”. To go a bit more on what this file does, it is simply a small text file on that machine that maps names to IP addresses. When pve01 looks up a name, it checks the file first and only asks DNS if the name isn’t listed there. So on the “192.168.22.92 pve01.home.arpa pve01” line to use the new address, 192.168.30.92 (Figure 1.17). This is important to change because Proxmox’s services like the Web UI, cluster filesystem, background daemons, etc. resolve their own hostname on startup to find out what address they should use. If that address is incorrect, the name resolves to an address the machine doesn’t or no longer uses and can cause services like the Web UI to not connect, usually with errors like “Unable to resolve node name”.

<img width="880" height="388" alt="image" src="https://github.com/user-attachments/assets/d8fbd650-bb96-4c20-9ad2-7ebc687e9001" />

Figure 1.17

## Step 16.

Next, under CT 100 (gitea0 > Network, I edited net0 and changed the IP to 192.168.30.88/24, and the Gateway to 192.168.30.1 to migrate Gitea to the management VLAN (Figure 1.18). I rebooted the container using “# reboot” in the console, but it won’t be reachable yet. As soon as the container reboots it will get the new 192.168.30.88 address. But until the Optiplex’s port is swapped to use the Management VLAN in Unifi, the bridge is still on the Private VLAN, so nothing can reach that address.

<img width="975" height="294" alt="image" src="https://github.com/user-attachments/assets/d8e3c0d3-ad3f-4731-bab6-dc045671f523" />

Figure 1.18

## Step 17.

In CT 101 (dragonwilds) > Network, I edited net0 and added the VLAN Tag 22 (Figure 19). Without this tag, dragonwilds sends untagged frames like pve01 and gitea. After flipping the port the Optiplex is on, those would land on the new native VLAN, management, with the containers IP still set to 192.168.22.89, a Private VLAN IP. I then rebooted the container to apply these changes.

<img width="975" height="281" alt="image" src="https://github.com/user-attachments/assets/068470ab-62f4-4fc5-8b2a-47d03027c713" />

Figure 1.19

## Step 18.

Now that both containers were configured to operate on their respective networks, I rebooted pve01 using its shell, and as it rebooted went back to the Unifi Web UI. In Client Devices I first found what port on the switch it was connected to. In Unifi Devices I selected the switch, selected Port 2, and changed the Native VLAN to use the Management_VLAN (30), and under Tagged VLAN Management swapped from Allow All to custom. Then, under Tagged VLANs, I added Private_VLAN (22) and applied the changes. Adding the private VLAN here is what lets CT 101 stay on the private VLAN instead of management (Figure 1.20). After a couple minutes, I confirmed I was able to access the Web UI using the new address at https://192.168.30.92:8006.

<img width="598" height="553" alt="image" src="https://github.com/user-attachments/assets/da76dcea-632f-4d8b-af2d-5e84542ce3b3" />

Figure 1.20

## Step 19.

I then booting CT 100 and 101, I started by testing testing the functionality of 101 by confirming the dragonwilds server was active using “# systemctl status dragonwilds.service” and pinging the gateway using “# ping 192.168.22.1”. After that I attempted to connect to the server from my PC but noticed Dragonwilds itself wasn’t connecting, and neither was YouTube, Spotify, or other common sites. After making sure everything was physically connected, I confirmed my IP and gateway were both in the management VLAN using “# ipconfig /all” in command prompt. I then pinged the gateway, pinged 1.1.1.1 to test the internet without DNS, and then pinged google.com. After running “# ping google.com” and getting a could not find host error, I narrowed the issue down to DNS. Looking back at ipconfig /all, I noticed the DNS server listed was still under the old IP (Figure 1.21). I fixed this with the commands “# ipconfig /renew”, and “# ipconfig /flushdns” to get the DNS Servers IP to update to the new address. I then was able to successfully connect to the Dragonwilds server from my PC. I also made sure I could connect to Gitea’s Web UI using the new address.

<img width="975" height="464" alt="image" src="https://github.com/user-attachments/assets/337f5c60-6335-4c52-b2d5-455358ec9187" />

Figure 1.21

## Step 20.

Back in CT 100’s console, I ran “# grep -n "192.168" /mnt/gitea-data/gitea/conf/app.ini” to view if the addresses listed here were still old, which they were. I edited the file using “# nano /mnt/gitea-data/gitea/conf/app.ini” and changed all addresses to use .33. I then ran the same grep command to confirm the changes saved (Figure 22). I then restarted using “# cd /opt/gitea && docker compose restart”. In PowerShell, I also entered my second-brain directory and set the url of origin to the new .30 address using “# git remote set-url origin http://192.168.30.88:3000/JustinLawitz/second-brain.git”and confirmed the change using “# git remote -v” and “# git fetch”  and used the credentials to authenticate (Figure 1.23).

<img width="975" height="592" alt="image" src="https://github.com/user-attachments/assets/187dc55a-b445-4905-8588-fe2bc724ced2" />

Figure 1.22

<img width="975" height="234" alt="image" src="https://github.com/user-attachments/assets/9dc4f38f-73ca-48c7-8b6c-6644d3744fdb" />

Figure 1. 23

## Step 21.

To finish up this section I removed the accept rule for 192.168.22.0/24 in pve01 > Datacenter > Firewall, made sure I couldn’t access the Web UIs of any services from the Private or Guest network, and reset the lease time for all three networks to 86400 seconds instead of 600.

## Section 2: Proxmox Cluster

## Step 1.

The next thing I want to do is incorporate an HP EliteDesk 800 G6 into my current home lab. This process entails setting up a Proxmox cluster, which links multiple physical Proxmox Virtual Environment (PVE) servers called nodes, into a single, centrally managed group. This will result in a single Web UI for multiple physical machines or nodes and allow for live migration of containers or VMs from one physical server to another without any downtime. It will also allow for High Availability (HA), meaning that with shared storage and enough nodes, the cluster can automatically restart or migrate running VMs or containers to a healthy server if one physical server goes down for whatever reason. I was gifted this sff (Small Form Factor) PC by my supervisor from Alterman, so the first step was to wipe the SSD using ShredOS. ShredOS is a lightweight, Linux-based OS designed to permanently and securely erase data from a storage device. As I already had a USB with ShredOS, I just had to boot the sff pc using it. To do this I entered BIOS, turned off Secure Boot, Fast Boot, and changed the Startup Delay to 5 seconds to be able to re-access BIOS after restart. I then restarted the pc and used the USB to boot ShredOS and wipe the ssd (Figure 2.1)

<img width="975" height="731" alt="image" src="https://github.com/user-attachments/assets/9dd48458-0663-4797-aba6-eaa8d3c62bde" />

Figure 2.1

## Step 2.

Once any data on the ssd was wiped, I next rebooted the pc using a USB with Proxmox, the same one I used for the Optiplex. From there I set the target harddisk to the internal ssd, and configured the country, time zone, and keyboard settings. Next came the network config portion which was the most important. For the management interface I used nic0, I set the hostname to pve02.home.arpa to match the convention of the Optiplex (pve01), set the IP address to 192.168.30.90, the gateway to 192.168.30.1, and the DNS server to 192.168.30.199. I then confirmed all the configurations were correct on the Summary page and installed Proxmox.

## Step 3.

After installing Proxmox on pve02, I next shut it down and connected it to power, and my switch on the rack. I also entered the Unifi Web UI and configured the port settings on the switch to mimic the port settings for pve01, including allowing the Private VLAN under Tagged VLANs incase I ever migrate the Dragonwilds container on the private network to pve02. I also set the Native VLAN to the management VLAN (30) (Figure 2.2). After a couple seconds, pve02 had connection and I was able to navigate to the Web UI at https://192.168.30.90:8006 (Figure 2.3).

<img width="581" height="553" alt="image" src="https://github.com/user-attachments/assets/c80b7f67-ff22-4f58-af57-49658d13779e" />

Figure 2.2

<img width="975" height="515" alt="image" src="https://github.com/user-attachments/assets/d4ac0a11-d78a-4270-9191-c2db5cb38782" />

Figure 2.3

## Step 4.

Once in the Web UI of pve02 I entered the shell, and edited the /etc/hosts file using “# nano /etc/hosts”. In the hosts file of pve02, I added pve01’s information, specifically the line “192.168.30.92 pve01.home.arpa pve01”. I then connected to pve01’s Web UI and edited its hosts file with the same information but for pveo2, specifically the line “192.168.30.90 pve02.home.arpa pve02”. I then confirmed both files saved the changes by using “# cat /etc/hosts” on both shells (Figure 2.4).

<img width="975" height="290" alt="image" src="https://github.com/user-attachments/assets/3e3aa076-2189-443a-8acf-517e3b0a98cb" />

Figure 2.4

## Step 5.

On pve02 > Datacenter > Firewall I also added an accept rule for traffic from the management VLAN to make sure I wouldn’t get locked out of the Web UI incase of any misconfiguration. Specifically, I set the Direction as in, Action as ACCEPT, Source as 192.168.30.0/24, ticked the Enable box, and clicked Add (Figure 2.5).

<img width="975" height="558" alt="image" src="https://github.com/user-attachments/assets/aad3cc87-1675-41f4-9e96-526cc1204942" />

Figure 2.5

## Step 6.

In Repositories under Updates, I then disabled both the entries for the enterprise versions of Proxmox and added the No-Subscription repository (Figure 2.6). Under Updates I then clicked Refresh to pull the package list from the No-Subscription repository, and Upgrade to apply these changes (Figure 2.7). After this, I also rebooted pve02.

<img width="975" height="456" alt="image" src="https://github.com/user-attachments/assets/791b31ee-93d7-4d12-80c5-27cda22e3957" />

Figure 2.6

<img width="975" height="514" alt="image" src="https://github.com/user-attachments/assets/6a0eb4d9-8981-4936-bed9-1891c3739059" />

Figure 2.7

## Step 7.

To create the actual cluster, on pve01 > Datacenter > Cluster I clicked Create Cluster, named it homelab, and left the IP of pve01 in the Cluster Network field. On pve02 > Datacenter > Cluster I clicked Join Cluster, which asked me for “encoded Cluster information” from pve01. This information is back on pve01 under Join Information (Figure 2.8). After copying that information and pasting it into the field on pve02, it expanded the menu, autofilling information such as pve01’s IP and Fingerprint from the encoded information. After entering the root password for pve01 I clicked Join ‘homelab’ to create the cluster (Figure 2.9 and 2.10).

<img width="975" height="309" alt="image" src="https://github.com/user-attachments/assets/9c43637c-cdc1-4599-a9d1-bfdb657d76a6" />

Figure 2.8

<img width="975" height="526" alt="image" src="https://github.com/user-attachments/assets/7f314bdf-b474-407d-ad6c-342ae5b8c517" />

Figure 2.9

<img width="975" height="513" alt="image" src="https://github.com/user-attachments/assets/e1164514-760b-4712-a47b-e02aaac5f2df" />

Figure 2.10
