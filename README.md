# IT-Infrastructure-Design
This is a repository to simulate an IT Infrastructure I've designed.

# Creating a System Infrastructure


## Phase 1:
Getting VM's needed

## Phase 2:

(ADD image here)
Setting up and configuring machines in a network:

### Creating groups to apply (Principle of Least Priviledge)
Do this as many times as needed (might not be needed skip for now)

```
sudo groupadd {groupname}
sudo adduser {username}
sudo usermod -a -G {groupname} {username}
```
### Setting up the File Server on the *Server* Machine

I will be using Samba over NFS due to it's compatability with both windows and linux clients, and many other advantageous.

This command will check for and install any updates.
```
sudo apt-get udpate
sudo apt-get install samba
```
We will then stop the service before making any configuration changes, and we will restart it again once we are done.
```
sudo systemctl stop smbd
```
We will then need to make a directory we will be sharing, and add files to test.
```
mkdir clientshare
cd clientshare
touch file1 file2
```
We will then back up the current Samba Configuration File incase we mess up somewhere and need to revert back, and create a new empty file.
```
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.original
touch /etc/samba/smb.conf
```
The *[global]* section of the file defines the server roll. This means what access is allowed or restricted on the IP address subnet, and allow mapping access for guests - so we need do edit the new file we made;
```
sudo nano /etc/samba/smb.conf
```
and we need to add the following command: (MIGHT CHANGE LATER ON)
```
[global]
	   server role = standalone server
           map to guest = Bad User
	   usershare allow guests = yes 	# <-- can change this if needed
	   hosts allow = 192.168.100.0/24
	   hosts deny = 0.0.0.0/0
```
The configuration entered previously denies guest access through mapping but allows access to user shares with guest permissions enabled. Then, the network access is limited to specific subnet (192.168.100.0/24).

Once saved we will need to test the configuration using testparm:
```
testparm
Load smb config files from /etc/samba/smb.conf
```
This will output the Global Parameters we set to comfirm everything is working.

Next we will need to set up a Samba Share Name, the [sharename] is a unique, human-readable label the makes the directory show up as a Shared Folder on Unix systems, and a Network Folder to Windows/MacOS systems via the SMB Protocol. We can do multiple of these for different directories independently.

I will create a new [sharename] called [testshare] in the smb.conf file with some other confiuguration settings.
```
[testshare]
	   comment = test file share
	   guest ok = yes
           path = /home/clientshare
	   read only = no
```
Once that is made we will need to test using testparm again to see that the configurations are saved and set. It should output something similar to this.
```
testparm
Load smb config files from /etc/samba/smb.conf
Loaded services file OK.
Weak crypto is allowed

Server role: ROLE_STANDALONE

Press enter to see a dump of your service definitions

 Global parameters
[global]
	server role = standalone server
        map to guest = Bad User
	usershare allow guests = Yes 	# <-- can change this if needed
	idmap config * : backend = tdb
	hosts allow = 192.168.0.0/16
	hosts deny = 0.0.0.0/0


[testshare]
	comment = test file share
	force group = server_group
	force user = server_user
	guest ok = Yes
	path = /home/clientshare
	read only = No

```
Now that we have a Shared Folder set up on the *Server Machine*.


https://www.baeldung.com/linux/file-server-smb-nfs












