# Homelab To replace Spotify: Practicing Networking Fundamentals
Repurposed old PCs to serve as a Proxmox VE hypervisor and a separate backup server. Virtualized a router + firewall with OPNsense. 

1. Title and intro

Once upon a time, I was studying for the CCNA and wanted hands-on experience with how real networks are built and maintained: VLANS, firewall rules, NAT, and backups. So I turned 2 old computers into a homelab that replaced an important part of my daily life: Spotify. Now I stream from my own self hosted music server Jellyfin, using Symfonium and SSH button on my Samsung phone. I wrote a Bash script, stored in Proxmox hosted Jellyfin container, that pulls new songs from my YouTube Music playlist into the Jellyfin library. With the SSH Button app, the whole process runs with one tap. 

2. network diagram and design

draw.io diagram

3. hardware + whats runiing

Proxmox Hypervisor VE: 2013 iMac (Specs)
Proxmox Backup Server: Dell Laptop (Specs) 
- storage local node hosted named pve
VLANS 1,10,20: OPNSense (virtual router + firewall), GS308E switch + ethernet wires 

| Hardware | Specs | Role |
|---|---|---|
| 2013 iMac | i5, 16GB RAM, 1TB | Proxmox VE hypervisor |
| ASUS X550L laptop | i3, 8GB RAM, 500GB | Proxmox Backup Server |
| Netgear GS308E Switch | 8 port managed switch | 802.1Q VLAN trunk to the iMac (VLANs 1,10,20) |
| Fios router | - | ISP router, home LAN 192.168.1.0/24 | 
| GL.iNet Opal | - | Wi-Fi extender |

4. configs and why
   - Tailscale subnet routing
     
To stream my music library away from my home Wifi, I installed Tailscale on the Proxmox host (Linux Debian OS), enabling a private and encrypted network connection between the devices via tailnet [1]. Using Tailscale, I set up my Proxmox host as a subnet router by having it forward any incoming traffic from the tailnet to devices on my home network, such as Jellyfin and OPNsense (which will be discussed later). So every time my phone sends traffic to Jellyfin, it knows to send it through the tailnet tunnel to the Proxmox host because the host advertises my home network: 192.168.1.0/24 (private subnet), to the tailnet. 

(diagram)

   - PBS

Proxmox Backup Server (PBS) acts as my disaster recovery center. In case my Proxmox server experiences downtime or the Jellyfin LXC container (CT 1001) is corrupted, I can easily restore the Jellyfin container from the most recent stored Jellyfin backup. The Proxmox host is scheduled to create backups daily at 12:00 AM and sends them to PBS. To manage backups from filling up the limited storage of the backup disk, PBS runs scheduled prune jobs [2] that discard every backup except:
   - 1 per day for the last 7 days
   - 1 per week for the last 4 weeks
   - 1 per month for the last 3 months.
Everyday at 12:00 AM, PBS runs garbage collection [3], which deletes the data left behind by pruned backups from the ASUS laptop's disk.
To ensure the backups work, I restored one to a test Jellyfin container (CT 999), and its media and storage matched the original Jellyfin container.
     
   - OPNsense VLANS and firewall rules

To create a secure homelab, I split my network into 3 separate VLANs (VLAN segmentation):
   - VLAN 1 (default, 192.168.1.0/24): this subnet is a private network configured by my Fios router, so any time a device joins my home Wifi, the router hands out IP addresses in the 192.168.1.0/24 range. Moreover, all the management web UIs (Proxmox, PBS, OPNSense, and the NetGear) live on this default private network.
   - VLAN 10 (10.10.10.0/24): Jellyfin lives here.
   - VLAN 20 (10.10.20.0/24): Reserved for lab experiments.

To manage the traffic between the VLANs, OPNsense, a virtual router + firewall, was implemented on Proxmox host as VM 100. The WAN is configured to face VLAN 1, which is how VLAN 10 and 20 reach VLAN 1 and the internet, and VLAN 10 and 20's gateways are set to OPNsense: 10.10.10.1 and 10.10.20.1, respectively.
The purpose of performing VLAN segmentation is to ensure Jellyfin service doesn't break from lab experiments, which must live on VLAN 20. If a lab experiment service is misconfigured, it could reach Jellyfin service, hence why Jellyfin needs to have its own VLAN. This is the common practice in business networks; for example, it's critical for the IT department to reach the HR department, but there's no reason for the HR department to have access to the IT department traffic. Moreover, VLAN segmentation creates an organized and accessible network topology, where IT admins are able to see where everything is, creating an efficient troubleshooting practice.
     
   - Switch trunk setup on NETGEAR MANAGED SWITCH
Currently, the Netgear Managed Switch has the Opal (Wifi Extender), iMac (Proxmox Host), and ASUS laptop (PBS) connected to ports 1, 2, and 5, respectively. Because this switch acts as a central connection point, any device that connects to the switch is automatically connected to the home network by the Opal (in extender mode) and passes its wireless connection to other devices. In an office setting, the Opal would be similar to a wireless bridge, linking the switch to the main network.
Additionally, it's critical to use a managed switch vs an unmanaged one is because of its ability to forward frames to certain ports, based on the VLAN tag of the frame. For example, when the switch receives a frame with a VLAN tag = 10, it forwards it only to VLAN 10's ports.
To configure the managed switch to be ready for VLAN forwarding, I accessed its web GUI. Firstly, I connected the switch to the LAN port of my Opal, so the switch is automatically connected to my home network. From there, I used the NETGEAR Discovery Tool app to identify which IP address the switch obtained, then accessed the web GUI via that IP address. Lastly, I set a static IP address in the switch's web GUI so its IP address stays the same. 
     
5. music automation pipeline

6. Troubleshoot Stories
   - Wrong Gateway Config on OPNsense
   - Routing Issues: Opal misconfig -> Can't access proxmox
   - Inside jellyfin container (CT 1001), dns issues

7. Next Steps
8. Glossary:

   [1] Tailnet - Tailscale creates an encrypted network between my devices that have Tailscale enabled: my phone + Proxmox host. 
   
   [2] Prune Job - PBS deletes the table of contents/index of every past backup that isn't kept by the configured schedule backup policy: 7 daily, 4 weekly, 3 monthly.
   
   [3] Garbage Collection - PBS deletes the pages, stored on the Asus's laptop disk, that no table of contents/index points to.
   
