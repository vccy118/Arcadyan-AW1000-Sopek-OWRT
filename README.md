# Arcadyan-AW1000-Sopek-OWRT
Some tips to set up Sopek OWRT that might differ from other OWRT setup

<h1>Common Errors:</h1>  

Error ```tee: "/dev/fd/64": No such file or directory``` will be the most common, especially when initializing adblocker blocklist.  
On Sopek OWRT, ```/dev/fd``` is not linked properly and this is used as a temp folder for downloads.  
To solve this, go to the terminal and run ```ln -sf /proc/self/fd /dev/fd``` then initialize blocklist downloads.  
To have this persist across reboots, run ```nano /etc/rc.local```, then add ```ln -sf /proc/self/fd /dev/fd``` above ```exit 0```.

<h1>Email Notification for SMS:</h1>  

Since this specific router model and Sopek OWRT is for 5G internet, SMS notifications are very important.  
Even more so if you use a prepaid card or have low data quota cap. SMS reminders from mobile providers is a top priority.  
The default luci-app-sms-tool and sms_tool only has slow blinking LED for notification, very easy to miss.  
It is also quite surprising that there's no opkg packages to easily enable email notification features for SMS.  
I managed to implement it and it works, but it is quite complicated and not at all elegant.  
Hopefully someone will come up with a better implementation in the future.  

Step by step instructions:
1. Go to Modem > SMS messages in openwrt  
   <img width="422" height="171" alt="image" src="https://github.com/user-attachments/assets/836e8d27-f0a0-4718-a113-684ee7efa62d" />  
   Then Configuration > SMS Settings and set Message storage area to "SIM card", then Notification Settings and turn off Notify New Messages  
   <img width="545" height="310" alt="image" src="https://github.com/user-attachments/assets/327996f0-ac02-4416-9eb2-4ee28fa3c0d5" />  
   <img width="545" height="229" alt="image" src="https://github.com/user-attachments/assets/45354299-26e7-443c-bfb4-474ce065cbbf" />  
   Make sure to Save and Apply. This is to avoid any accidental reading of SMS being received by the router.
2. Go to the terminal and install msmtp, a smtp client, to enable email functionality on openwrt.
   ```
   opkg update
   opkg install msmtp msmtp-mta
   ```
3. Next, configure msmtp using ```nano /etc/msmtprc``` and enter the following:
   ```
   defaults
   auth on
   tls on
   tls_trust_file /etc/ssl/certs/ca-certificates.crt
   logfile /var/log/msmtp.log

   account gmail
   host smtp.gmail.com
   port 587
   from youremail@gmail.com
   user youremail@gmail.com
   password <app password>

   account default : gmail
   ```
   Enter your gmail for both "from" and "user". For "app password" go to <url>https://myaccount.google.com/apppasswords</url>  
   You will need to enable 2fa and generate a new app password for msmtp.  
4. Test msmtp using ```echo "Test email from SoPEK router" | msmtp recipient@example.com```  
5. Send an SMS to the openwrt sim card and run ```echo -e "AT+CMGF=1\r" > /dev/ttyUSB2 && sleep 1 && cat /dev/ttyUSB2```  
   This is to set SMS to be in text mode. The return should be "OK".  
   Then run ```echo -e "AT+CMGL=\"ALL\"\r" > /dev/ttyUSB2 && sleep 2 && cat /dev/ttyUSB2```, the return should be the new SMS.  
6. If during step 5, there are multiple return "OK" or looping, we need to disable the echo feedback.  
   Run ```echo -e "ATE0\r" > /dev/ttyUSB2``` then ```{ echo -e "AT&W\r"; sleep 1; } > /dev/ttyUSB2``` to save the modem profile.  
   This will make ATE0 persist accross reboots and avoid buffer overflow errors on the SMS port. Run ATE1 to revert back if needed.  
7. Go to Modem > SMS messages to delete all the SMS to avoid complications with the next step.  
8. In the terminal, run ```nano /usr/bin/sms-notify.sh``` and enter the following:  
   ```
   #!/bin/sh
   # sms-notify.sh — reads unread SMS, emails them, then deletes them
   
   MODEM_PORT="/dev/ttyUSB2"
   TMPFILE="/tmp/sms_dump"
   EMAIL="yourname@example.com"

   # Set text mode (safe to run every time)
   echo -e "AT+CMGF=1\r" > $MODEM_PORT
   sleep 1

   # Read unread messages using background cat
   cat $MODEM_PORT > $TMPFILE &
   PID=$!
   echo -e "AT+CMGL=\"REC UNREAD\"\r" > $MODEM_PORT
   sleep 3
   # Kill background cat if still running
   kill -0 $PID 2>/dev/null && kill $PID 2>/dev/null

   # Parse and process each SMS
   grep -n "+CMGL:" $TMPFILE | while IFS=: read -r lineno line; do
       # Get message index from +CMGL line
       INDEX=$(echo "$line" | awk -F',' '{print $1}' | awk '{print $2}')
       # Get sender
       SENDER=$(echo "$line" | awk -F',' '{print $3}' | tr -d '"')

       # Compose email
       SUBJECT="New SMS from $SENDER"
       BODY="From: $SENDER\nMessage:\n$(cat $TMPFILE)"
   
       # Send email
       echo -e "Subject: $SUBJECT\n\n$BODY" | msmtp "$EMAIL"
   
       # Delete message from modem memory
       echo -e "AT+CMGD=$INDEX\r" > $MODEM_PORT
       sleep 1
   
       logger -t sms-notify "Sent email for SMS from $SENDER"
   done
   
   # Cleanup
   rm -f $TMPFILE
   ```
   Replace yourname@example.com with the receiver email. This will also delete any SMS received after an email notification is sent.  
   Additional notes: ```cat $MODEM_PORT > $TMPFILE &``` is needed because AT+CMGL doesn't include an EOF at the end of the response.  
   This will cause the modem port buffer to build up over time and eventually become full.  
   When that happens AT+CMGL will return nothing and the script will stop working.  
   ```kill -0 $PID 2>/dev/null && kill $PID 2>/dev/null``` is needed for the same reason. As the response from AT+CMGL has no EOF at  
   the end of the response, the cat PID will persist until it is killed. Again, cat process has a PID limit, the script will stop.  
10. Run ```chmod +x /usr/bin/sms-notify.sh``` to make the script executable.  
11. Send another SMS to the openwrt sim card and run ```/usr/bin/sms-notify.sh``` to test if the script works.  
    Check your email for the SMS notification and run ```logread | grep sms-notify``` and it should return  
    ```sms-notify: Sent email for SMS from +60123456789```  
12. Next, go to System > Scheduled Tasks and add in ```*/1 * * * * /usr/bin/sms-notify.sh```  
    This sets up a cron job to run the script every minute.  
13. Reboot openwrt and send another SMS to the openwrt sim card, you should receive an SMS notification in your email within a minute.  
14. If you check ```logread | grep sms-notify``` in the terminal, it will return  
    ```cron.err crond[15681]: USER root pid 16473 cmd /usr/bin/sms-notify.sh``` every minute.
    This is normal whenever there is no new SMS.

<h1>Migrating from Cloudflare Tunnel to Tailscale:</h1>  

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
