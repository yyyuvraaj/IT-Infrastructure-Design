# Static IP Configuration for Ubuntu Server 24.04

This guide covers setting static IP addresses on all three VMs using **Netplan**, the default network manager in Ubuntu Server 24.04.

---

## Prerequisites

- All VMs running Ubuntu Server 24.04 LTS
- Access to terminal on each VM
- Know your network details (gateway IP, DNS servers)

---

## Step 1: Find Your Network Interface Name and Details

On **each VM**, run:

```bash
ip a
```

Look for your interface name (examples: `enp0s3`, `ens33`, `eth0`). Note it down.

Find your gateway:

```bash
ip route | grep default
# Output example: default via 192.168.1.1 dev enp0s3
```

Check DNS (optional):

```bash
cat /etc/resolv.conf
```

---

## Step 2: Locate and Edit Netplan Config

Find the Netplan config file:

```bash
ls /etc/netplan/
```

Common filenames:
- `00-installer-config.yaml`
- `50-cloud-init.yaml`

Edit the file (usually `00-installer-config.yaml`):

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

---

## Step 3: Configure Static IP for Each VM

**Important:** Replace `enp0s3` with YOUR actual interface name from Step 1.

### For node1 (File Server 1)

Replace the entire file contents with:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.1.151/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 192.168.1.1
          - 8.8.8.8
```

Press `Ctrl+X`, then `Y`, then `Enter` to save.

### For node2 (File Server 2)

Replace the entire file contents with:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.1.152/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 192.168.1.1
          - 8.8.8.8
```

Press `Ctrl+X`, then `Y`, then `Enter` to save.

### For webserver (Web Server)

Replace the entire file contents with:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.1.153/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 192.168.1.1
          - 8.8.8.8
```

Press `Ctrl+X`, then `Y`, then `Enter` to save.

---

## Step 4: Apply Network Changes

On **each VM**, after editing:

### Test First (Safe Method)

```bash
sudo netplan try
```

You have **120 seconds** to press `Enter` to confirm. If no input, it reverts automatically.

### Apply Directly (If Test Passes)

```bash
sudo netplan apply
```

### Verify the Changes

```bash
# Check IP address
ip a

# Test external connectivity
ping -c 3 8.8.8.8

# Test DNS
ping -c 3 google.com
```

---

## Step 5: Update /etc/hosts for Hostname Resolution (Recommended)

On **all three VMs**, edit `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Find the line with `127.0.1.1` and add these lines after it:

```
192.168.1.151   node1
192.168.1.152   node2
192.168.1.153   webserver
```

Save with `Ctrl+X`, `Y`, `Enter`.

Now you can use hostnames instead of IPs:

```bash
ping node1
ping node2
ping webserver
```

---

## Step 6: Update GlusterFS Mount Points (If Already Configured)

### On all VMs, check your `/etc/fstab`:

```bash
cat /etc/fstab
```

**Remove any duplicate lines** and ensure you have only **one** GlusterFS mount entry.

On **node1 and node2** (optional, use localhost):

```bash
sudo nano /etc/fstab
```

Change to:

```
localhost:/file-volume /mnt/glusterfs glusterfs defaults,_netdev 0 0
```

On **webserver**, use node1's IP:

```bash
sudo nano /etc/fstab
```

Change to:

```
192.168.1.151:/file-volume /mnt/glusterfs glusterfs defaults,_netdev 0 0
```

Or with automatic failover to node2:

```
192.168.1.151:/file-volume /mnt/glusterfs glusterfs defaults,_netdev,backupvolfile-server=192.168.1.152 0 0
```

Remount:

```bash
sudo umount /mnt/glusterfs
sudo mount -a
mount | grep gluster
```

---

## Step 7: Verify All VMs Can See Each Other

From **any VM**, test connectivity to the others:

```bash
# From any VM:
ping -c 3 node1
ping -c 3 node2
ping -c 3 webserver
```

All should respond with no packet loss.

---

## Reference Table

| VM Name | Hostname | Static IP | Purpose |
|---------|----------|-----------|---------|
| VM 1 | node1 | 192.168.1.151 | File Server 1 (Samba + GlusterFS) |
| VM 2 | node2 | 192.168.1.152 | File Server 2 (Samba + GlusterFS) |
| VM 3 | webserver | 192.168.1.153 | Web Server (Nginx) |

**Network Details:**
- Gateway: 192.168.1.1
- DNS Primary: 192.168.1.1 (or your router IP)
- DNS Secondary: 8.8.8.8
- Subnet Mask: /24 (255.255.255.0)

---
