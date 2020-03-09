![Hive Helsinki](https://miro.medium.com/max/3200/1*IszpKRN_x7RbKDClj6oqhQ.png)

# VM Part
**The properties of VM:**
- [x] hypervisor: VirtualBox; 
- [x] Linux OS: Debian (64-bit);
- [x] RAM size: 2048 MB;
- [x] size of the hard disk is 8.00 GB (VDI, fixed size);
- [x] image: [debian-10.1.0-amd64-netinst.iso](https://www.debian.org/distrib/);
- [x] partitioning method: manual;
- [x] 4.2 GB partition (set to 4.5 GB during installation).

**Installed software:**
- [x] SSH server;
- [x] standart system utilities.
# Network and Security Part
### 1. Creating a non-root user to connect to the machine and work
The user was created when we set up the VM. Enter login and password to log in.
### 2. [Using `sudo` with created user](https://hostadvice.com/how-to/how-to-create-a-non-root-user-on-ubuntu-18-04-server/) to perform operation requiring special rights

**1. Install `sudo`**
```bash
su
apt-get update
apt-get install sudo
```
**2. Add the non-root user to the `sudo` group**
```bash
sudo usermod -aG sudo ykalashn
```
**3. Test the `sudo` access**

Switch to the newly created user:
```bash
su - ykalashn
```
Use the sudo command to run the whoami command:
```bash
sudo whoami
```
If the user has sudo access then the output of the `whoami` command will be 
`root`.
### 3. Create a [static IP](https://linuxconfig.org/how-to-setup-a-static-ip-address-on-debian-linux) and a [Netmask in \30](https://www.aelius.com/njh/subnet_sheet.html)
In VirtualBox go to `Settings` > `Network` > `Attached to:` and choose `Bridged Adapter`.

We are about to edit some files. I prefer using `vim` editor, so let's install it:
```bash
sudo apt-get install vim
``` 
By default, you will find the following configuration within the `/etc/network/interfaces` network config file. Update the `# The primary network interface`:
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
Go to `/etc/network/interfaces.d` and create a file `enp0s3`: 
```bash
cd interfaces.d
sudo vim enp0s3
```
Create a new network configuration file with any arbitrary file name eg. `enp0s3` and include the `enp0s3` IP address configuration shown below:
```
# cat /etc/network/interfaces.d/enp0s3
iface enp0s3 inet static
      address 10.12.180.52
      netmask 255.255.255.252       // Netmask in \30
      gateway 10.12.254.254
 ```
 Now you can see result by first `restarting the network service`, and then running command `ip a`:
 ```bash
 sudo service networking restart
 ip a
 ```
### 4. Change the default port of the SSH service, SSH access has to be done with publickeys. SSH root access should not be aloved directly

**Let's change our default port**
Edit `/etc/ssh/sshd_config` file:
```bash
sudo vim /etc/ssh/sshd_config
```
Update the line `# Port 22` by removing `#` and typing a new port number:
```diff
- # Port 22
+ Port 45678
```
> :point_up: Make sure you choose a random port, preferably higher than **1024** (the superior limit of standard well-known ports). The maximum port that can be setup for for SSH is **65535/TCP**.

Save the file, and restart the **sshd service**:
```bash
sudo service sshd restart
```
Now, try to log in with your **ssh**:
```bash
ssh ykalashn@10.12.180.52 -p 45678
```
**Let's create `SSH publickey`.**

Run from your **host terminal** and then set a **passphrase**:
```bash
ssh-keygen
```
Copy the publickey to VM:
```bash
ssh-copy-id ykalashn@10.12.180.52 -p 45678
```
To disable root SSH login, edit `/etc/ssh/sshd_config` file on your VM.
```bash
sudo vi /etc/ssh/sshd_config
```
Change the line `#PermitRootLogin` to `PermitRootLogin no`, `#PubkeyAuthentication` to `PubkeyAuthentication yes`, and `#PasswordAuthentication` to `PasswordAuthentication no`. 

Restart the SSH daemon: 
```bash
sudo service sshd restart
```
### 5. Set the rules for [firewall](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-debian-9)
Install and enable `UFW` (Uncomplicated Firewall):
```bash
sudo apt-get install ufw
sudo ufw enable
```
To check the list of all services, enter:
```bash
less /etc/services
```
To set the defaults used by UFW, use these commands:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
To allow `ssh`, `http`, and `https` use these commands:
```bash
sudo ufw allow 45678/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```
You can check status of the firewall by entering:
```bash
sudo ufw status
```
### 6. Set a DOS protection on your open ports of the VM
Install fail2ban
```bash
sudo apt-get install iptables fail2ban apache2
```
Copy `jail.conf` into `jail.local`:
```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
Now we need to edit the `/etc/fail2ban/jail.conf` file, the `JAILS` part inside the file should look like this:
```gitattributes
[sshd]
enabled = true
port    = ssh 
logpath = %(sshd_log)s
backend = %(sshd_backend)s
maxretry = 3
bantime = 600
```
After `HTTP servers`, add:
```
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
















