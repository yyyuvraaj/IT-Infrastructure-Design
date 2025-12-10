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
sudo groupadd (groupname)
sudo adduser (username)
sudo usermod -a -G (groupname) (username)
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
	   usershare allow guests = yes
	   hosts allow = 192.168.100.0/24
	   hosts deny = 0.0.0.0/0
```

```

```
















