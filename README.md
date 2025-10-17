# Arcadyan-AW1000-Sopek-OWRT
Some tips to set up Sopek OWRT that might differ from other OWRT setup

Error ```tee: "/dev/fd/64": No such file or directory openwrt``` will be the most common, especially when initializing adblocker blocklist.  
On Sopek OWRT, ```/dev/fd``` is not linked properly and this is used as a temp folder for downloads.  
To solve this, run ```ln -sf /proc/self/fd /dev/fd``` then initialize blocklist downloads.  
To have this persist accross reboots, run ```nano /etc/rc.local```, then add ```ln -sf /proc/self/fd /dev/fd``` above ```exit 0```.

Tailscale
