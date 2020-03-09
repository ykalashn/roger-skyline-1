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
**1. Install `sudo`:**
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
> If the user has sudo access then the output of the `whoami` command will be 
`root`.
### 3. Create a static IP and a Netmask in \30
In VirtualBox go to `Settings` > `Network` > `Attached to:` and choose `Bridged Adapter`.

We are about to edit some files. I prefer using `vim` editor, so let's install it. Also, let's install `net-tools`, so we can use `ifconfig` command.
```
sudo apt-get install vim
sudo apt-get install net-tools
sudo ifconfig
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
 







![Hive Helsinki](https://miro.medium.com/max/3200/1*IszpKRN_x7RbKDClj6oqhQ.png)


