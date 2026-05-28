# Learning-GCP
Knowledge repository for learning GCP 

# Disk Creation in GCP

## Creating extra disk and attaching it to a compute instance

* Follow the steps and create a new disk with the desired storage capacity. \
{NOTE: Make sure to select the same zone where the instance is present.} This is done to ensure no issues in attaching disk.
* Edit the settings of the instance and attach existing disk.
* Now the disk is attached physically but logically it must be properly formatted and mounted on the instance.

The following commands are to be used after attaching disk.

To check the file system.
```bash
lsblk
```
```bash
sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0 discard /dev/sdb
```
```bash
sudo mkdir -p /mnt/data
```
```bash
sudo mount -o discard,defaults /dev/sdb /mnt/data
```
```bash
df -h
```
```bash
sudo resize2fs /dev/sdb
```


# Startup script for a managed instance group
```bash
#!/bin/bash
apt-get update
apt-get install -y apache2
systemctl start apache2
systemctl enable apache2
HOSTNAME=$(hostname)
echo "<html><body><h1>Hello from $HOSTNAME</h1><p>This page is served by VM: $HOSTNAME</p></body></html>" |tee /var/www/html/index.html