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
     
To stream my music library away from my home Wifi, I installed Tailscale on the Proxmox host (Linux Debian OS), enabling a private and encrypted network connection between the devices via tailnet. Using Tailscale, I set up my Proxmox host as a subnet router by having it forward any incoming traffic from the tailnet to devices on my home network, such as Jellyfin and OPNsense (which will be discussed later). So every time my phone sends traffic to Jellyfin, it knows to send it through the tailnet tunnel to the Proxmox host because the host advertises my home network: 192.168.1.0/24 (private subnet), to the tailnet. 

(diagram)

   - PBS

Proxmox Backup Server acts as a disaster recovery center, where in case my Proxmox server experiences downtime, it is easily accessible to restore Jellyfin media by the recent stored Jellyfin backups. Each backup is set to be created daily at 12:00AM. To manage backups from reaching 100% limited storage, PBE runs scheduled prune jobs, where each backup is discarded except for:
   - 
     
   - OPNsense VLANS and firewall rules
     
   - Switch trunk setup on NETGEAR MANAGED SWITCH
     
5. music automation pipeline

6. Troubleshoot Stories
   - Wrong Gateway Config on OPNsense
   - Routing Issues: Opal misconfig -> Can't access proxmox
   - Inside jellyfin container (CT 1001), dns issues

7. Next Steps
