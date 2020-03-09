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
### 2. Using sudo with created user to perform operation requiring special rights
**1. Install `sudo`**
```
su
apt-get update
apt-get install sudo
```
**2. Add the non-root user to the `sudo` group**
```
sudo usermod -aG sudo ykalashn
```
**3. Test the `sudo` access**

Switch to the newly created user:
```
su - ykalashn
```
Use the sudo command to run the whoami command:
```
sudo whoami
```
If the user has sudo access then the output of the `whoami` command will be 
`root`.
### 3. Create a static IP and a Netmask in \30
In VirtualBox go to `Settings` > `Network` > `Attached to:` and choose `Bridged Adapter`.

We are about to edit some files. I prefer using `vim` editor, so let's install it:
```
sudo apt-get install vim
``` 
By default you will find the following configuration within the `/etc/network/interfaces` network config file:
```
source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug eth0
iface eth0 inet dhcp
```
Update the `# The primary network interface`. The resulting content of `/etc/network/interfaces` network config file should look similar to the one below:
```
source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto enp0s3
```
Go to `/etc/network/interfaces.d` and create a file `enp0s3`. 
```
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
 ```
 sudo service networking restart
 ip a
 ```
### 4. Change the default port of the SSH service, SSH access has to be done with publickeys. SSH root access should not be aloved directly

**Let's change our default port**
- Edit `/etc/ssh/sshd_config` file:
```
sudo vim /etc/ssh/sshd_config
```
- Update the line `# Port 22` by removing `#` and typing a new port number:
```
Port 45678
```
> :point_up: Make sure you choose a random port, preferably higher than **1024** (the superior limit of standard well-known ports). The maximum port that can be setup for for SSH is **65535/TCP**.

- Save the file, and restart the **sshd service**:
```
sudo service sshd restart
```
- Now, try to log in with your **ssh**:
```
ssh ykalashn@10.12.180.52 -p 45678
```
**Let's create `SSH publickey`.**

- Run from your **host terminal** and then set a **passphrase**:
```
ssh-keygen
```
- Copy the publickey to VM:
```
ssh-copy-id ykalashn@10.12.180.52 -p 45678
```
- To disable root SSH login, edit `/etc/ssh/sshd_config` file on your VM.
```
sudo vi /etc/ssh/sshd_config
```
- Change the line `#PermitRootLogin` to `PermitRootLogin no`, and `#PasswordAuthentication` to `PasswordAuthentication no`. 

- Restart the SSH daemon: 
```
sudo service sshd restart
```
### 5. Set the rules for firewall
- Install and enable `UFW` (Uncomplicated Firewall)
```
sudo apt-get install ufw
sudo ufw enable
```






















