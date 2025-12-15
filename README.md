# IT-Infrastructure-Design

## PLEASE MAKE IPs STATIC BEFORE CONTINUING (staticIP_config.txt)

This is a repository to simulate an IT Infrastructure I've designed.

## Creating a System Infrastructure

**Base OS:** Ubuntu Server 24.04 LTS  
Download: https://ubuntu.com/download/server

**VM hostnames:**
- node1
- node2
- webserver  

For simplicity in the lab, the passwords are the same as the server names.

---

## Phase 1 – Topology

(ADD IMAGE OF TOPOLOGY HERE)

Suggested topology:

- `node1` and `node2`:
  - GlusterFS replica volume (shared storage)
  - Samba server exporting the replicated volume
- `webserver`:
  - Nginx web server
  - Mounts the GlusterFS volume for web-based file access

---

## Phase 2 – Base Prep on All VMs

Run on **node1**, **node2** and **webserver**:

```bash
sudo apt update && sudo apt upgrade -y

# (Optional but recommended)
sudo timedatectl set-ntp true
```

Check basic connectivity (change IPs/hostnames to match your lab):

```bash
ping -c 3 node1
ping -c 3 node2
ping -c 3 webserver
```

---

## Phase 3 – GlusterFS Cluster (node1 and node2)

### 3.1 Install GlusterFS

On **both node1 and node2**:

```bash
sudo apt update
sudo apt install glusterfs-server -y
sudo systemctl start glusterd
sudo systemctl enable glusterd
```

Create the brick directory:

```bash
sudo mkdir -p /data/brick1/gv0
```

### 3.2 Form the Cluster and Create Volume

On **node1** (replace IPs with your own):

```bash
# Probe node2 from node1
sudo gluster peer probe 192.168.1.152   # node2 IP

# Check peer status
sudo gluster peer status
```

Create a **replica 2** volume (accept the split-brain warning with `y`):

```bash
sudo gluster volume create file-volume replica 2 \
  192.168.1.151:/data/brick1/gv0 \
  192.168.1.152:/data/brick1/gv0 \
  force

sudo gluster volume start file-volume
sudo gluster volume info
```

### 3.3 Mount GlusterFS on node1 and node2

On **both node1 and node2**:

```bash
sudo mkdir -p /mnt/glusterfs
echo "localhost:/file-volume /mnt/glusterfs glusterfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
sudo mount -a
```

Test:

```bash
mount | grep gluster
sudo ls -ld /mnt/glusterfs
```

Create a test directory and files (on node1):

```bash
sudo mkdir -p /mnt/glusterfs/clientshare
sudo touch /mnt/glusterfs/clientshare/file1_from_node1.txt
sudo touch /mnt/glusterfs/clientshare/file2_from_node1.txt
```

Verify on node2:

```bash
sudo ls /mnt/glusterfs/clientshare
```

You should see the same files on both nodes.

---

## Phase 4 – Samba File Server (node1 and node2)

### 4.1 Install Samba

On **both node1 and node2**:

```bash
sudo apt update
sudo apt install samba -y
sudo systemctl stop smbd
```

### 4.2 Backup and Create Samba Config

```bash
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.original
sudo touch /etc/samba/smb.conf
sudo nano /etc/samba/smb.conf
```

Add:

```ini
[global]
    server role = standalone server
    map to guest = Bad User
    usershare allow guests = yes
    hosts allow = 192.168.1.0/24
    hosts deny = 0.0.0.0/0

[testshare]
    comment = Gluster-backed test file share
    path = /mnt/glusterfs/clientshare
    browseable = yes
    read only = no
    guest ok = no
    valid users = @smbgroup
    create mask = 0660
    directory mask = 0770
```

Check syntax:

```bash
testparm
```

### 4.3 Create Samba Users and Permissions

On **both node1 and node2**:

```bash
# Group and user
sudo groupadd smbgroup
sudo useradd -M -s /usr/sbin/nologin -G smbgroup smbuser

# Linux password (for file permissions)
sudo passwd smbuser

# Samba password
sudo smbpasswd -a smbuser
sudo smbpasswd -e smbuser

# Set directory ownership/permissions
sudo chown -R root:smbgroup /mnt/glusterfs/clientshare
sudo chmod 2770 /mnt/glusterfs/clientshare
```

Start and enable Samba:

```bash
sudo systemctl start smbd
sudo systemctl enable smbd
```

### 4.4 Test from a Client

From another Linux machine (or another VM):

```bash
sudo apt install smbclient -y
smbclient //192.168.1.151/testshare -U smbuser
# Then test with ls, put, get, pull if needed

smbclient //192.168.1.152/testshare -U smbuser
```

---

## Phase 5 – Web Server (webserver VM)

### 5.1 Install Nginx

On **webserver**:

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

Test:

```bash
curl http://localhost
```

### 5.2 Mount GlusterFS on webserver

Install Gluster client and mount:

```bash
sudo apt install glusterfs-client -y

sudo mkdir -p /mnt/glusterfs
echo "192.168.1.151:/file-volume /mnt/glusterfs glusterfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
sudo mount -a
```

Check:

```bash
mount | grep gluster
sudo ls /mnt/glusterfs/clientshare
```

You should see the same files created on node1/node2.

### 5.3 Fix Permissions for Nginx

Ensure Nginx (`www-data`) can traverse and read:

```bash
sudo chmod 755 /mnt
sudo chmod 755 /mnt/glusterfs
sudo chmod -R 755 /mnt/glusterfs/clientshare
```

(For a lab this is fine; in production you would lock this down more.)

### 5.4 Nginx Config to Serve the Share

Create a new site:

```bash
sudo nano /etc/nginx/sites-available/files
```

Content:

```nginx
server {
    listen 80;
    server_name _;

    root /mnt/glusterfs/clientshare; # make sure this is actaully the correct directory
    autoindex on;
    autoindex_exact_size off;
    autoindex_localtime on;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Enable the site and disable the default:

```bash
sudo ln -s /etc/nginx/sites-available/files /etc/nginx/sites-enabled/files
sudo rm /etc/nginx/sites-enabled/default  # remove default site if present
sudo nginx -t
sudo systemctl reload nginx
```

Test:

```bash
curl -I http://localhost/
# Expect: HTTP/1.1 200 OK
```

Then from your host browser:

- Go to `http://<webserver-ip>/`  
You should see a directory listing of the files in `/mnt/glusterfs/clientshare`.

---

## Phase 6 – Basic Redundancy Tests

1. **Create a new file on node1:**

   ```bash
   sudo touch /mnt/glusterfs/clientshare/redundancy_test_from_node1.txt
   ```

2. **Verify on node2 and webserver:**

   ```bash
   # node2
   sudo ls /mnt/glusterfs/clientshare

   # webserver
   sudo ls /mnt/glusterfs/clientshare
   ```

3. **Check via Browser and SMB:**

   - Browser: `http://<webserver-ip>/` → file should appear.
   - SMB: `smbclient //192.168.1.151/testshare -U smbuser` → `ls`.

4. **Simulate node failure:**

   - Power off `node1`.
   - Access:
     - `smbclient //192.168.1.152/testshare -U smbuser`
     - `http://<webserver-ip>/`
   - Files should still be available via node2.

---
