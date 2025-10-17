# Arcadyan-AW1000-Sopek-OWRT
Some tips to set up Sopek OWRT that might differ from other OWRT setup

<h1>Common Errors:</h1>  

Error ```tee: "/dev/fd/64": No such file or directory``` will be the most common, especially when initializing adblocker blocklist.  
On Sopek OWRT, ```/dev/fd``` is not linked properly and this is used as a temp folder for downloads.  
To solve this, run ```ln -sf /proc/self/fd /dev/fd``` then initialize blocklist downloads.  
To have this persist across reboots, run ```nano /etc/rc.local```, then add ```ln -sf /proc/self/fd /dev/fd``` above ```exit 0```.

<h1>Migrating from Cloudflare Tunnel to Tailscale</h1>  

Cloudflare Tunnel as of October 2025 is very slow (at least for me), sometimes taking up to 1 minute to load a locally hosted webpage.  
Tailscale offers better security and speed. The downside is that only devices within your tailnet can access your webpages.  
It can be used with your own domain name and reverse proxy (eg. Traefik with SSL encryption and https), just like Cloudflare Tunnel.  
Using tailnet DNS name is much easier, all that needs to be done is to enable MagicDNS and HTTPS Certificates.  
However, tailnet DNS names are all random long words and there's no option to choose your own. So, I opt to use my own domain name.  

1. Make sure Traefik is set up, and Cloudflare Tunnel is disabled  
2. Set up hostnames on openwrt at`Network > DHCP and DNS > Hostnames, where the ip address is pointed to Traefik  
   <img width="600" height="400" alt="image" src="https://github.com/user-attachments/assets/4c10a1ac-98d6-41b5-990b-3a324d869e82" />  
3. Set up Tailscale on openwrt and expose subnets, as shown below  
   <img width="400" height="400" alt="image" src="https://github.com/user-attachments/assets/67385021-76e6-42d2-8d36-7c055c463847" />  
4. In Tailscale admin console, edit route settings and accept subnet routes from openwrt Tailscale  
5. Go to DNS, under Nameservers, add nameserver and select custom  
   <img width="250" height="250" alt="image" src="https://github.com/user-attachments/assets/341be6bd-1f32-4cc3-a8b9-dc469e063120" />  
6. Enable Restrict to Domain (Split DNS), key in the openwrt local ip and the domain to be used  
   <img width="400" height="400" alt="image" src="https://github.com/user-attachments/assets/e37ff452-cfb1-4f96-9a3a-6cd2dd9f0aa0" />  
7. Click Save and now all your locally hosted webpages should work on any devices within the same tailnet  
