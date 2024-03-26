# Proxmox stuff

This is a collection of stuff that I wrote for Proxmox. Its possble to use the [Ansible roles](#ansible) I wrote or to use the [bash scripts](#bash-scripts) for the backup & restore tasks.

---

# Ansible

Small Ansible playbook and role collection for Proxmox related stuff.

# prox_config_backup

I just wrote this script quick and dirty.
Some people use it on a daily basis (including me).

There might be a PBS backup feature to backup PVE cluster config in the near future provided by the Proxmox team.
But since this was only mentioned on the roadmap we still have to wait.

Meanwhile I manage all PVE nodes with Ansible and usually have no need to restore configuration unless all cluster
nodes failed at once. But having a full cluster config backup is still useful and makes PVE admins sleep well at night (or day).

The script must be run as root, and can be run from cron or an interactive terminal.

## Backup - interactive terminal

* Download the [script](https://raw.githubusercontent.com/daNutzzzzz/proxmox-stuff/master/prox_config_backup.sh)  
```wget -qO- https://raw.githubusercontent.com/daNutzzzzz/proxmox-stuff/master/prox_config_backup.sh -O /etc/cron.daily/pve_config_backup```
* Set the permanent backups directory environment variable ```export BACK_DIR="/path/to/backup/directory"``` or edit the script to set the `$DEFAULT_BACK_DIR` variable to your preferred backup directory
* Make the script executable ```chmod 0744 /etc/cron.daily/pve_config_backup```
* Shut down ALL VMs + LXC Containers if you want to go the safe way. (Not required)
* Run the script ```./prox_config_backup```

### Cron

* To set up a automatic cron job on a monthly (```/etc/cron.daily``` or ```/etc/cron.weekly``` can be used to!) schedule, running the prox_config_backup script, follow these steps:
```wget https://raw.githubusercontent.com/DerDanilo/proxmox-stuff/master/prox_config_backup.sh -O /etc/cron.daily/pve_config_backup```
* Make the script executable ```chmod 0744 /etc/cron.daily/pve_config_backup```
* Change ```DEFAULT_BACK_DIR="/home/pve"``` and ```MAX_BACKUPS=7``` to the values you want!

Optional: [Execute run-parts](https://superuser.com/questions/402781/what-is-run-parts-in-etc-crontab-and-how-do-i-use-it) to see if it contains errors:
```run-parts -v --test /etc/cron.daily```
```run-parts -v --test /etc/cron.weekly```
```run-parts -v --test /etc/cron.monthly```

### Notification

The script supports [healthchecks.io](https://healthchecks.io) notifications, either to the hosted service, or a self-hosted instance. The notification sends during the final cleanup stage, and either returns 0 to tell Healthchecks that the command was successful, or the exit error code (1-255) to tell Healthchecks that the command failed. To enable:
* Set the `$HEALTHCHECK` variable to 1
* Set the `$HEALTHCHECK_URL` variable to the full ping URL for your check. Do not include anything after the UUID, the status flag will be added by the script.

# Restore
❗ **ONLY USE THIS SCRIPT ON THE SAME NODE / PROXMOX VERSION, OTHERWISE IT WILL BREAK YOUR FRESH PROXMOX INSTALLATION. IT WILL ALSO FAIL IF YOU ARE RUNNING A CLUSTER!** ❗

For more info also see #5.

# prox_config_restore

## Restore Options

### Manually

On my machine, you end up with a GZipped file of about 1-5 MB with a name like "proxmox_backup_proxmoxhostname_2017-12-02.15.48.10.tar.gz".  
Depending upon how you schedule it and the size of your server, that could eventually become a space issue so don't  
forget to set up some kind of archive maintenance.

To restore, move the file back to proxmox with cp, scp, webmin, a thumb drive, whatever.  
I place it back into the /var/tmp directory from where it came. 

```
# Unpack the original backup
tar -zxvf proxmox_backup_proxmoxhostname_2017-12-02.15.48.10.tar.gz
# unpack the tared contents
tar -xvf proxmoxpve.2017-12-02.15.48.10.tar
tar -xvf proxmoxetc.2017-12-02.15.48.10.tar
tar -xvf proxmoxroot.2017-12-02.15.48.10.tar

# If the services are running, stop them:
for i in pve-cluster pvedaemon vz qemu-server; do systemctl stop $i ; done

# Copy the old content to the original directory:
cp -avr /var/tmp/var/tmp/etc /etc
cp -avr /var/tmp/var/tmp/var /var
cp -avr /var/tmp/var/tmp/root /root

# And, finally, restart services:
for i in qemu-server vz pvedaemon pve-cluster; do systemctl start $i ; done
```

If nothing goes wrong, and you have separately restored the VM images using the default Proxmox process.  
You should be back where you started. But let's hope it never comes to that.


### Script

* Download the [script](https://raw.githubusercontent.com/daNutzzzzz/proxmox-stuff/master/prox_config_restore.sh)  
```cd /root/; wget -qO prox_config_restore.sh https://raw.githubusercontent.com/daNutzzzzz/proxmox-stuff/master/prox_config_restore.sh```
* Make the script executable ```chmod 0744 ./prox_config_restore.sh```
* Run the script `./prox_config_restore.sh proxmox_backup_proxmoxhostname_2017-12-02.15.48.10.tar.gz`
* Press `ctrl + c` to exit instead of reboot or `Enter` to Reboot

## Other Hardware Restore Process

### Install Proxmox

Install Proxmox onto new Hardware, values entered don’t matter as they will be replaced, just ensure you have network configured for temporary access.

### Access Proxmox

Access Proxmox fresh install via temp IP/PW.

### Change password (if temp used)
```passwd root```

### Disable Swap
```swapoff -a```

### Get current disk values (if restoring to new host)
```
# Add disk to fstab
cat /etc/fstab

# Get UUID of disk
blkid

# Get disk structure
lsblk
```

* Download the [script](https://raw.githubusercontent.com/daNutzzzzz/proxmox-stuff/master/prox_config_restore.sh)  
```cd /root/; wget -qO prox_config_restore.sh https://raw.githubusercontent.com/daNutzzzzz/proxmox-stuff/master/prox_config_restore.sh```
* Make the script executable ```chmod 0744 ./prox_config_restore.sh```
* Run the script `./prox_config_restore.sh proxmox_backup_proxmoxhostname_2017-12-02.15.48.10.tar.gz`
* Press `ctrl + c` to exit instead of reboot or `Enter` to Reboot

### Fix disk issues (if restoring to new host)
```
# Get UUID of disk
blkid # or note UUID from mkfs

# Add disk to fstab
sudo nano /etc/fstab
```
### Fix repos
```rm /etc/apt/sources.list.d/pve-enterprise.list```
```echo "# deb https://enterprise.proxmox.com/debian/ceph-quincy bookworm enterprise" > /etc/apt/sources.list.d/ceph.list```

### Fix network interface issues (if restoring to new host)
#### Show current interfaces
```ifconfig -a``` or ```ip addr``` or ```ip link show```

#### Correct Network Interace Config
```nano /etc/network/interfaces```

#### Restart Network Service
```systemctl restart networking.service``` or ```systemctl restart networking``` or ```systemctl restart systemd-networkd``` or ```service network-manager restart```

#### Bring interfaces UP if they are DOWN
```ip link set dev <interface> up```

### Re-permission backup files
```chmod 0744 /home/rclone/rclone/rclone; chmod 0744 /home/rclone/rclonesync; chmod 0744 /home/Scripts/update_upgrade.sh```

### Remove Proxmox Subscription Notice
```sed -Ezi.bak "s/(Ext.Msg.show\(\{\s+title: gettext\('No valid sub)/void\(\{ \/\/\1/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js && systemctl restart pveproxy.service```

### Add Fake Subscriptions
```
cd /root/; wget -qO pve-fake-subscription_0.0.9+git-1_all.deb https://github.com/Jamesits/pve-fake-subscription/releases/download/v0.0.9/pve-fake-subscription_0.0.9+git-1_all.deb
dpkg -i pve-fake-subscription_*.deb
echo "127.0.0.1 shop.maurer-it.com" | tee -a /etc/hosts
rm pve-fake-subscription*.deb
```
### Reboot Node
```reboot```

### Reinstall Netdata Agent if required
```https://app.netdata.cloud/spaces/```

#### If you get the error **Failed to claim node because the Cloud thinks it is already claimed.**
```rm /var/lib/netdata/registry/netdata.public.unique.id```

### Install sensors
```apt-get install lm-sensors -y```
```sensors-detect```
```/etc/init.d/kmod start```
```systemctl restart netdata```

### Resinatall Tools
```
apt install nvme-cli -y && 
```


### Reboot Node
```reboot```

## Sources
http://ziemecki.net/content/proxmox-config-backups


