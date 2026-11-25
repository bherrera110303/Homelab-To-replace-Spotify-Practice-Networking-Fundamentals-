# Homelab to Replace Spotify: Practicing Networking Fundamentals
Repurposed old PCs to serve as a Proxmox VE hypervisor and a separate backup server. Virtualized a router + firewall with OPNsense. 

## 1. Overview

Once upon a time, I was studying for the CCNA and wanted hands-on experience with how real networks are built and maintained: VLANs, firewall rules, NAT, and backups. So I turned 2 old computers into a homelab that replaced an important part of my daily life: Spotify. Now I stream from my own self-hosted music server, Jellyfin, using the Symfonium and SSH Button apps on my Samsung phone. I wrote a Bash script, stored in the Jellyfin container on Proxmox, that pulls new songs from my YouTube Music playlist into the Jellyfin library. With the SSH Button app, the whole process runs with one tap. 

## 2. Network Diagram and Design

<img width="519" height="725" alt="Homelab network diagram" src="https://github.com/user-attachments/assets/0900e4e7-b0ac-4712-a22b-ea911297a7c5" />

<img width="807" height="831" alt="Homelab design diagram" src="https://github.com/user-attachments/assets/18786d3b-3a8c-42bc-9517-e6a9017b1edf" />

## 3. Hardware + What's Running

| Hardware | Specs | Role |
|---|---|---|
| 2013 iMac | i5, 16GB RAM, 1TB | Proxmox VE hypervisor |
| ASUS X550L laptop | i3, 8GB RAM, 500GB | Proxmox Backup Server |
| Netgear GS308E Switch | 8 port managed switch | 802.1Q VLAN trunk to the iMac (VLANs 1,10,20) |
| Fios router | - | ISP router, home LAN 192.168.1.0/24 | 
| GL.iNet Opal | - | Wi-Fi extender |

## 4. Configurations + Why
### Tailscale Subnet Routing
     
To stream my music library away from my home Wifi, I installed Tailscale on the Proxmox host (Linux Debian OS), enabling a private and encrypted network connection between the devices via tailnet [1]. Using Tailscale, I set up my Proxmox host as a subnet router by having it forward any incoming traffic from the tailnet to devices on my home network, such as Jellyfin and OPNsense (which will be discussed later). So every time my phone sends traffic to Jellyfin, it knows to send it through the tailnet tunnel to the Proxmox host because the host advertises my home network: 192.168.1.0/24 (private subnet), to the tailnet. 

### Proxmox Backup Server (PBS)

Proxmox Backup Server (PBS) acts as my disaster recovery center. In case my Proxmox server experiences downtime or the Jellyfin LXC container (CT 1001) is corrupted, I can easily restore the Jellyfin container from the most recent stored Jellyfin backup. The Proxmox host is scheduled to create backups daily at 12:00 AM and sends them to PBS. To manage backups from filling up the limited storage of the backup disk, PBS runs scheduled prune jobs [2] that discard every backup except:
   - 1 per day for the last 7 days
   - 1 per week for the last 4 weeks
   - 1 per month for the last 3 months.

Everyday at 12:00 AM, PBS runs garbage collection [3], which deletes the data left behind by pruned backups from the ASUS laptop's disk.
To ensure the backups work, I restored one to a test Jellyfin container (CT 999), and its media and storage matched the original Jellyfin container.
     
### OPNsense VLANs and Firewall Rules

To create a secure homelab, I split my network into 3 separate VLANs (VLAN segmentation):
   - VLAN 1 (default, 192.168.1.0/24): my home network, where it carries untagged frames through the switch. The Fios router assigns IPs to any new connected devices, but is unknowledgeable of VLANs. Moreover, all the management web UIs (Proxmox, PBS, OPNSense, and the NetGear) live on this default private network.
   - VLAN 10 (10.10.10.0/24): Jellyfin lives here.
   - VLAN 20 (10.10.20.0/24): Reserved for lab experiments.

To manage the traffic between the VLANs, OPNsense, a virtual router + firewall, was implemented on the Proxmox host as VM 100. The WAN is configured to face VLAN 1, which is how VLAN 10 and 20 reach VLAN 1 and the internet, and VLAN 10 and 20's gateways are set to OPNsense: 10.10.10.1 and 10.10.20.1, respectively. Whenever a packet is addressed to the OPNsense WAN IP address and port 8096, it is forwarded to Jellyfin at 10.10.10.50 (destination NAT), so clients on my home network and over Tailscale can reach Jellyfin without having access to VLAN 10.  

The purpose of performing VLAN segmentation is to ensure Jellyfin service doesn't break from lab experiments, which must live on VLAN 20. If a lab experiment service is misconfigured, it could reach Jellyfin service, hence why Jellyfin needs to have its own VLAN. This is common practice in business networks; for example, there’s no reason for a company’s guest Wi-fi or Sales department to reach the payroll server. In my lab, the firewall was configured that VLAN 20’s incoming traffic to VLAN 10 are blocked. Moreover, VLAN segmentation creates an organized and accessible network topology, where IT admins are able to see where everything is, creating an efficient troubleshooting practice.
     
### Switch Trunk Setup on the Netgear Managed Switch
Currently, the Netgear Managed Switch has the Opal (Wifi Extender), iMac (Proxmox Host), and ASUS laptop (PBS) connected to ports 1, 2, and 5, respectively. Because this switch acts as a central connection point, any device that connects to the switch is automatically connected to the home network by the Opal (in extender mode), which passes its wireless connection to other wired devices. In an office setting, the Opal would be similar to a wireless bridge, linking the switch to the main network.

Additionally, it's critical to use a managed switch vs an unmanaged one because of its ability to forward frames to certain ports, based on the VLAN tag of the frame. For example, when the switch receives a frame with a VLAN tag = 10, it forwards it only to ports that are members of VLAN 10.

To configure the managed switch to be ready for VLAN forwarding, I accessed its web GUI. Firstly, I connected the switch to the LAN port of my Opal, so the switch is automatically connected to my home network. From there, I used the NETGEAR Discovery Tool app to identify which IP address the switch obtained, then accessed the web GUI via that IP address. Lastly, I set a static IP address in the switch's web GUI so its IP address stays the same.

After setting up my switch, for Port 2 (Proxmox host/iMac), I enabled 802.1Q VLAN mode [4] to control how frames will be forwarded through this port: VLAN 1 = untagged, VLAN 10 & 20 = tagged. Port 2 was chosen to be a trunk because 3 VLANs live on the host server, meaning since Jellyfin, OPNsense, and lab experiments are hosted on my Proxmox server, port 2 needs the capability to forward frames to different VLANs based on the tagged frame. Unlike Port 2, Port 1 & 5 have no reason to be trunked because there aren't multiple VLANs living on the hardware; Port 1 is for Wifi Extender and Port 5 is for receiving backups from the Proxmox host server. The managed switch knows which port(s) are member(s) of which VLANs, unlike an unmanaged switch, which floods broadcast frames to every port since it has no capability to read VLAN tags.
     
## 5. Music Automation Pipeline

To automatically add new songs from a YouTube Music playlist to my Jellyfin server for streaming on the Symfonium app, I created Bash scripts that download, tag, and import them every 30 minutes or with a tap of a button on my Samsung phone, making it practical and easy to use. 

The 2 Bash scripts, /root/sync-music.sh and /root/trigger-sync.sh, are stored in the Jellyfin container (CT 1001). Their contents are shown below.

Programs used: 
   - yt-dlp: downloads songs from the YouTube Music playlist and converts them to mp3 files
   - Deno: yt-dlp's dependency in order to handle YouTube's JavaScript since yt-dlp is written in Python, not JavaScript
   - beets: searches each song on MusicBrainz, tags and renames it, which is then moved to Jellyfin's /media/music folder

```bash
## /root/sync-music.sh

#!/bin/bash
## points the OS which interpreter (from its filepath program) to use and tells its an executable script

## yt-dlp needs deno and it knows where it is by the exported folders passed to the programs the script runs
export PATH="/root/.local/bin:/root/.deno/bin:$PATH"   ## $PATH = /usr/bin, /bin (what cron starts with). curl + mv live in these folders. /root/.local/bin = folder where yt-dlp & beets are

yt-dlp -x --audio-format mp3 --embed-thumbnail --add-metadata --download-archive ~/archive.txt -o "~/incoming/%(title)s.%(ext)s" "https://music.youtube.com/MY_PLAYLIST_URL"
## --download-archive ~/archive.txt: Writes every downloaded video ID to a text file so future runs skip songs you already downloaded.
## -o "~/incoming/%(title)s.%(ext)s": Names the file after the video title and drops it into your incoming folder.

beet import -q ~/incoming
## scans new mp3 audios in incoming folder, beet is used to search the files online and renames it cleanly -> moved to permanent music folder
## -q: quiet mode - never stops to ask; skips any tracks that it isn't 100% sure about

curl -s -X POST "http://127.0.0.1:8096/Library/Refresh" -H 'Authorization: MediaBrowser Token="API TOKEN"'
## -s: silent, hides curl's progress output
## -X POST: sends action command to local server on port 8096 (Jellyfin)
## -H: adds a header to the request
## 'Authorization' = jellyfin server receives this secret API token to prove the request is allowed and proceeds to scan new music from the music folder


## /root/trigger-sync.sh

#!/bin/bash
nohup flock -n /tmp/sync-music.lock /root/sync-music.sh > /dev/null 2>&1 &
## nohup keeps the script running even if the SSH session that started it closes
## flock prevents 2 copies of my music script from running at the same time and colliding since its manually and automatically triggered
## -n: don't wait for the lock. If another run already holds "/tmp/sync-music.lock", this run quits right away instead of queueing up
## > : operator that sends normal output into a file instead of screen
## /dev/null : Linux built in black hole
## 2>&1: all the error messages (channel 2) are directed to normal output (channel 1), eventually reaching the black hole. the & means "channel 1," not a file named 1.
## standalone &: run entire line in background
```
To ensure each song added to the YouTube playlist is added to the Jellyfin server, I used cron, a time-based scheduler built into Linux, to automatically run this command (shown below) every 30 minutes.
```bash
## Crontab
*/30 * * * * flock -n /tmp/sync-music.lock /root/sync-music.sh > /root/sync-music.log 2>&1
## */30 * * * *: scheduled to run every 30 minutes, every hour, every day
## flock -n /tmp/sync-music.lock: if a run is triggered, it will always check if there is a current run ongoing by checking the .lock file. if there is one, the triggered run would just quit and exit.
## /root/sync-music.sh > /root/sync-music.log: all the output messages will be written to /root/sync-music.log (overwritten each run)
## 2>&1: all the error messages (channel 2) are directed to normal output (channel 1), which will be written to the log. the & means "channel 1," not a file named 1
```

## 6. Troubleshooting Stories
### Wrong Gateway Config on OPNsense
On a stormy night, the wifi was spotty for my family. Some devices loaded normally, while other devices took longer for some applications to load. Eventually, I noticed the Opal was blinking blue, which was concerning because the Proxmox host can only reach the internet through the Opal's extender mode. I tried to access the Opal's and Proxmox host web UIs, but both failed to load. Thankfully I noticed that I implemented the OPNsense firewall the day before, so I checked to make sure the configurations didn't break anything. After going through the OPNsense's settings, I realized that I accidentally set the OPNsense LAN's IP as 192.168.1.1, which is the same as my Fios router, the home network's gateway. To fix the issue, I reassigned the interfaces on OPNsense. LAN moved from vtnet0 at 192.168.1.1/24 to vtnet1 at 10.10.10.1/24, and vtnet0 became the WAN interface at 192.168.1.153/24 (a DHCP reservation on the Fios router; later changed to 192.168.1.165). Afterward, the Opal's indicator turned white and the wifi was stable again. Thankfully, I was able to diagnose the issue right away, before my family noticed anything more than a slow connection.

### Opal's Extender Mode Feature
When I was setting up the switch, I had it plugged in to power and to the Opal's LAN port via an Ethernet cable. To access the switch's web GUI, I needed to figure out which IP address my Fios router gave the switch. That's where I learned ARP (Address Resolution Protocol), a protocol where a device broadcasts a message to the other devices located on the same LAN asking which device has this IP. Once it learns the MAC address [5] associated with that IP from the reply, it stores it in an ARP cache table, so the next time it sends a frame it knows which device to forward it to, preventing an unnecessary broadcast message. When I ran arp-scan on the Proxmox shell, the output showed 6 different IP addresses with the same MAC address. This was unusual because each device has its own MAC address, making them distinguishable from each other. That's when I realized the devices shared the same MAC address as the Opal. In extender mode, when my Fios router sends an ARP request for a device behind the Opal, the Opal answers on behalf of the host devices that are located behind it using its own MAC (Proxy ARP); hence, why the devices have the same MAC address as the Opal.

### Jellyfin outage????
While confirming my suspicions about past issues, I rebooted the Opal. Afterward, the Opal kept blinking blue, meaning the Opal's connection to the Fios router was unstable. Even so, I could still reach Proxmox's UI, and OPNsense, Tailscale, and the Jellyfin container were still active. But when I opened the Symfonium app, Jellyfin is shown offline, even though Tailscale showed my pve node to be online. Like checking continuity in a circuit, I went through each part of the data path using my homelab diagram. Running systemctl status jellyfin command showed Jellyfin service to be healthy and active, so the break had to be between my clients and the Jellyfin container. Since Jellyfin lives on VLAN 10 behind the firewall, and the firewall was implemented recently, I narrowed it down to OPNsense. First and foremost, in the Proxmox shell, I ran curl against OPNsense's WAN address, which port forwards to Jellyfin:

- FAILED: `curl -I http://192.168.1.165:8096` - sends an HTTP request to OPNsense's WAN address on Jellyfin's port, which OPNsense should forward to Jellyfin at 10.10.10.50. Result: "Failed to connect"

So I hypothesized that OPNsense was the cause of the Jellyfin "outage". According to the OPNsense logs, there were many failure messages starting right when I rebooted the Opal:

- `dhclient-script: Reason FAIL on vtnet1 executing` (LAN side of the firewall)
- `dhclient-script: Reason FAIL on vtnet0 executing` (WAN side of the firewall)

Dhclient is a DHCP client, where it requests for an IP address for an interface. According to the vtnet0 errors, OPNsense was unable to renew its WAN lease, and the Fios router never replied back because the Opal's link was down. Once the Opal's link was back online, even then, the port forward to Jellyfin stayed broken.

As for the vtnet1 failures, the LAN was configured with a static IP, so dhclient should never have been running on it. I filtered the running processes for dhclient:

- `ps aux | grep dhclient`: ps aux prints every running process, where the output is fed to the next command: grep dhclient and outputs processes containing the keyword: dhclient.

It showed leftover DHCP clients from October, when I reassigned interfaces during an earlier outage. Having contradictory settings for the LAN interface made OPNsense's recovery messier. Reloading all OPNsense services (console option 11) brought Jellyfin back. To ensure this doesn't happen again, I decided to manually configure OPNsense WAN IPv4 address to a static IP: 192.168.1.165/24 with gateway: 192.168.1.1, so during future outages, OPNsense won't have to ask the Fios router for its address. Then, I rebooted OPNsense to clear the leftover DHCP clients. After one more Opal reboot, the Opal came back solid white, and Symfonium showed Jellyfin online again.

Takeaways:
- Don't rely on DHCP servers for infrastructure the rest of the network depends on.
- Clean up after emergency fixes because leftovers can cause problems later.

## 7. Glossary

   [1] Tailnet - Tailscale creates an encrypted network between my devices that have Tailscale enabled: my phone + Proxmox host. 
   
   [2] Prune Job - PBS deletes the table of contents/index of every past backup that isn't kept by the configured schedule backup policy: 7 daily, 4 weekly, 3 monthly.
   
[3] Garbage Collection - PBS deletes the data chunks, stored on the ASUS laptop's disk, that no backup index points to anymore.

[4] 802.1Q – IEEE standard for VLAN tagging, where it adds a 4 byte tag to each Ethernet frame that holds the VLAN ID. This is particularly useful because switches and routers know which VLAN the frame belongs to.

[5] MAC address – a unique hardware address etched into every network card, used for delivering frames within the same LAN. IP addresses identify devices across networks; MAC addresses identify them on the local link.
