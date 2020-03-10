![Hive Helsinki](https://miro.medium.com/max/3200/1*IszpKRN_x7RbKDClj6oqhQ.png)

## VM Part
#### The properties of VM:
- [x] hypervisor: VirtualBox; 
- [x] Linux OS: Debian (64-bit);
- [x] RAM size: 2048 MB;
- [x] size of the hard disk is 8.00 GB (VDI, fixed size);
- [x] image: [debian-10.1.0-amd64-netinst.iso](https://www.debian.org/distrib/);
- [x] partitioning method: manual;
- [x] 4.2 GB partition (set to 4.5 GB during installation).

#### Installed software:
- [x] SSH server;
- [x] standart system utilities.
## Network and Security Part
### 1. Creating a non-root user to connect to the machine and work
The user was created when we set up the VM. Enter login and password to log in.
### 2. [Using `sudo` with created user](https://hostadvice.com/how-to/how-to-create-a-non-root-user-on-ubuntu-18-04-server/) to perform operation requiring special rights
- Install `sudo`
```sh
su
apt-get update
apt-get install sudo
```
- Add the non-root user to the `sudo` group
```sh
sudo usermod -aG sudo ykalashn
```
#### Test the `sudo` access

- Switch to the newly created user:
```sh
su - ykalashn
```
- Use the sudo command to run the whoami command:
```sh
sudo whoami
```
_If the user has sudo access then the output of the `whoami` command will be 
`root`._
### 3. Create a [static IP](https://linuxconfig.org/how-to-setup-a-static-ip-address-on-debian-linux) and a [Netmask in \30](https://www.aelius.com/njh/subnet_sheet.html)
- In VirtualBox go to `Settings` > `Network` > `Attached to:` and choose `Bridged Adapter`.

- We are about to edit some files. I prefer using `vim` editor, so let's install it:
```sh
sudo apt-get install vim
``` 
By default, you will find the following configuration within the `/etc/network/interfaces` network config file.

- Update the `# The primary network interface`:
```diff
source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
- allow-hotplug eth0
- iface eth0 inet dhcp
+ auto enp0s3
```
- Go to `/etc/network/interfaces.d` and create a file `enp0s3`: 
```sh
cd interfaces.d
sudo vim enp0s3
```
- Create a new network configuration file with any arbitrary file name eg. `enp0s3` and include the `enp0s3` IP address configuration shown below:
```sh
# cat /etc/network/interfaces.d/enp0s3
iface enp0s3 inet static
      address 10.12.180.62
      netmask 255.255.255.252       # Netmask in \30
      gateway 10.12.254.254
 ```
 _Now you can see result by first `restarting the network service`, and then running command `ip a`:_
```sh
 sudo service networking restart
 ip a
```
### 4. Change the default port of the SSH service, SSH access has to be done with publickeys. SSH root access should not be aloved directly

#### Change our default port
- Edit `/etc/ssh/sshd_config` file:
```sh
sudo vim /etc/ssh/sshd_config
```
- Update the line `# Port 22` by removing `#` and typing a new port number:
```diff
- # Port 22
+ Port 45678
```
> :point_up: Make sure you choose a random port, preferably higher than **1024** (the superior limit of standard well-known ports). The maximum port that can be setup for for SSH is **65535/TCP**.

- Save the file, and restart the **sshd service**:
```sh
sudo service sshd restart
```
- Now, try to log in with your **ssh**:
```sh
ssh ykalashn@10.12.180.62 -p 45678
```
#### Let's create `SSH publickey`.

- Run from your **host terminal** and then set a **passphrase**:
```sh
ssh-keygen
```
- Copy the publickey to VM:
```sh
ssh-copy-id ykalashn@10.12.180.62 -p 45678
```
- To disable root SSH login, edit `/etc/ssh/sshd_config` file on your VM:
```sh
sudo vi /etc/ssh/sshd_config
```
- Change the following lines:
```diff
- #PermitRootLogin
+ PermitRootLogin
- #PubkeyAuthentication
+ PubkeyAuthentication yes
- #PasswordAuthentication
+ PasswordAuthentication no
```
- Restart the SSH daemon: 
```sh
sudo service sshd restart
```
### 5. Set the rules for [firewall](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-debian-9)
- Install and enable `UFW` (Uncomplicated Firewall):
```sh
sudo apt-get install ufw
sudo ufw enable
```
- To check the list of all services, enter:
```sh
less /etc/services
```
- To set the defaults used by UFW, use these commands:
```sh
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
- To allow `ssh`, `http`, and `https` use these commands:
```sh
sudo ufw allow 45678/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```
_You can check status of the firewall by entering:_
```sh
sudo ufw status
```
### 6. Set a DOS protection on your open ports of the VM
- Install fail2ban
```sh
sudo apt-get install iptables fail2ban apache2
```
- Copy `jail.conf` into `jail.local`:
```sh
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
- Now, edit the `/etc/fail2ban/jail.conf` file, the `JAILS` part inside the file should look like this:
```sh
[sshd]
enabled = true
port    = ssh 
logpath = %(sshd_log)s
backend = %(sshd_backend)s
maxretry = 3
bantime = 600
```
- After `HTTP servers`, add:
```sh
[http-get-dos]
enabled = true
port = http,https
filter = http-get-dos
logpath = %(apache_error_log)s
maxretry = 300
findtime = 300
bantime = 600
action = iptables[name=HTTP, port=http, protocol=tcp]
```
- Create a filter:
```
sudo vi /etc/fail2ban/filter.d/http-get-dos.conf
```
_The output should look like this:_
```sh
# cat /etc/fail2ban/filter.d/http-get-dos.conf
[Definition]
failregex = ^<HOST> -.*"(GET|POST).*
ignoreregex =
```
- Let's reload the `firewall` and restart our `fail2ban` service:
```sh
sudo ufw reload
sudo service fail2ban restart
```
_Now we can check the status of `fail2ban`:_
```sh
sudo fail2ban-client status
```
### 7. Set a [protection against scans](https://en-wiki.ikoula.com/en/To_protect_against_the_scan_of_ports_with_portsentry) on the VM's open ports
- First, install `portsentry`:
```sh
sudo apt-get install portsentry
```
- Edit the `/etc/default/portsentry` file:
```diff
- TCP_MODE="tcp"
+ TCP_MODE="atcp"
- UDP_MODE="udp"
+ UDP_MODE="audp"
```
- Modify the `/etc/portsentry/portsentry.conf` file:
```diff
##################
# Ignore Options #
##################
# 0 = Do not block UDP/TCP scans.
# 1 = Block UDP/TCP scans.
# 2 = Run external command only (KILL_RUN_CMD)

- BLOCK_UDP="0"
+ BLOCK_UDP="1"
- BLOCK_TCP="0"
+ BLOCK_TCP="1"
```
- In the same file, comment all the current `KILL_ROUTE` commands out and **uncomment the following one**:
```sh
KILL_ROUTE="/sbin/iptables -I INPUT -s $TARGET$ -j DROP"
```
- Also, comment this command out:
```sh
KILL_HOSTS_DENY="ALL: $TARGET$ : DENY
```
- Start the `portsentry` service and it will begin blocking the `port scans`:
```sh
sudo /etc/init.d/portsentry start
```
### 8. [Stop the services](https://www.linux.com/tutorials/cleaning-your-linux-startup-process/) you don't need
- To check all the **enabled services**, enter:
```
sudo systemctl list-unit-files --type=service | grep enabled
```
- To disable `keyboard-setup.service` and `console-setup.service`, enter:
```
sudo systemctl stop keyboard-setup.service
sudo systemctl disable keyboard-setup.service
sudo systemctl stop console-setup.service
sudo systemctl disable console-setup.service
```
### 9. Script that updates all the sources of package and logs in /var/log/update_script.log, scheduled task every week at 4 am and every time at reboot
- Create a file and give it exectutable rights:
```sh
sudo touch up-to-date.sh
sudo chmod 777 up-to-date.sh
```
- Open the `up-to-date.sh` file enter the commands which will update the sources of package and the packages: 
```sh
sudo apt-get update -y >> /var/log/update_script.log
sudo apt-get upgrade -y >> /var/log/update_script.log
```
- Now, let's open `crontab` file..
```sh
sudo crontab -e
```
.. and add the scheduled tasks:
```sh
@reboot sudo /home/ykalashn/up-to-date.sh
0 4 * * 1 sudo /home/ykalashn/up-to-date.sh
```
### 10. Make a script to monitor changes of the /etc/crontab file and [sends an email](https://www.cmsimike.com/blog/2011/10/30/setting-up-local-mail-delivery-on-ubuntu-with-postfix-and-mutt/) to root, add a scheduled task
```sh
sudo touch croncheck.sh
sudo chmod 777 croncheck.sh
```
- The `croncheck.sh` file should contain:
```sh
#!/bin/bash

sudo touch /home/ykalashn/cronchekki
sudo chmod 777 /home/ykalashn/cronchekki
f1="$(md5sum '/etc/crontab' | awk '{print $1}')"
f2="$(cat '/home/ykalashn/cronchekki')"
echo ${f1}
echo ${f2}

if [ "f1" != "f2" ] ; then
	md5sum /etc/crontab | awk '{print $1}' > /home/ykalashn/cronchekki
	echo "croncheck: modified" | mail -s "croncheck result" root@debian.lan
fi
```
- Edit the `crontab` file by adding a new line:
```sh
0 0 * * * root /home/ykalashn/croncheck.sh
```
- To use command `mail`, we need to install `bsd-mailx`:
```sh
sudo apt install bsd-mailx
```
- To recieve emails we need to install `postfix`:
```sh
sudo apt-get install postfix
```
- In `postfix` setup choose the following:
```sh
General type of mail configuration: Local only
System mail name: debian.lan
Root and postmaster mail recipient: root@localhost
Other destinations to accept mail for: debian.lan, localhost.lan, localhost
Force synchronous updates on mail queue? No
Local networks: # press enter
Mailbox size limit (bytes): 0
Local address extension character: # press enter
Internet protocols to use: all
```
- If you want to change these settings after the initial install you can with: 
```sh
sudo dpkg-reconfigure postfix
```
- Now to configure where email notifications are sent to:
```sh
sudo vim /etc/aliases
```
- Enter:
```sh
root: root
```
- The new alias need to be loaded into the hashed alias database (/etc/aliases.db) with the following command:
```sh
sudo newaliases
```
Then change the home mailbox directory:
```
sudo postconf -e "home_mailbox = mail/"
```
Restart the `postfix` service:
```sh
sudo service postfix restart
```
Install the non-graphical mail client `mutt`:
```sh
sudo apt install mutt
```
Create a file `.muttrc` for `mutt` in the `/root/` directory (`su` to access) and edit it:
```sh
set mbox_type=Maildir
set folder="/root/mail"
set mask="!^\\.[^.]"
set mbox="/root/mail"
set record="+.Sent"
set postponed="+.Drafts"
set spoolfile="/root/mail"
```
Start `mutt`:
```sh
mutt

# press 'q' to exit
```
- Finally, send an email to the root user testing our setup is working:
```sh
echo "Bonjour, une baguette s’il vous plaît!" | mail -s "Bonjour from `hostname`" root
```
_Login as a root user and run command `mutt`. The mail should now be there._

## Web Part
![Web Part](https://i.ibb.co/xDHxYtG/Screen-Shot-2020-03-10-at-12-21-31-PM.png)

- Generate [`SSL self-signed key` and `certificate`](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-apache-in-ubuntu-16-04):
```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt
```
Here's an example:
```
Country name: FI
State or Province Name:
Locality Name: Helsinki
Organization Name: Hive Helsinki
Organizational Unit Name:
Common Name: 10.12.180.62 (VM IP address)
Email Address: root@debian.lan
```
- Create the file `/etc/apache2/conf-available/ssl-params.conf` and edit it:
```
SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
SSLProtocol All -SSLv2 -SSLv3
SSLHonorCipherOrder On

Header always set X-Frame-Options DENY
Header always set X-Content-Type-Options nosniff

SSLCompression off
SSLSessionTickets Off
SSLUseStapling on
SSLStaplingCache "shmcb:logs/stapling-cache(150000)"
```
- Edit the `/etc/apache2/sites-available/default-ssl.conf` file:
```diff
<IfModule mod_ssl.c>
	<VirtualHost _default_:443>
-		ServerAdmin webmaster@localhost
+ 		ServerAdmin root@localhost
+		ServerName 10.12.180.62
		
		DocumentRoot /var/www/html
		
		ErrorLog ${APACHE_LOG_DIR}/error.log
		CustomLog ${APACHE_LOG_DIR}/access.log combined
		
		SSLEngine on
		
-		SSLCertificateFile	/etc/ssl/certs/ssl-cert-snakeoil.pem
+ 		SSLCertificateFile	/etc/ssl/certs/apache-selfsigned.crt
-		SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key
+		SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
		
		<FilesMatch "\.(cgi|shtml|phtml|php)$">
				SSLOptions +StdEnvVars
		</FilesMatch>
		<Directory /usr/lib/cgi-bin>
				SSLOptions +StdEnvVars
		</Directory>
	</VirtualHost>
</IfModule>
```
- Add a redirect rule to `/etc/apache2/sites-available/000-default.conf` to redirect HTTP to HTTPS:
```diff
+ Redirect "/" "https://192.168.10.42/"
```

