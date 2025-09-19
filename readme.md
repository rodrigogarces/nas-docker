# Setup a NAS server from scratch with ubuntu server

## Disable lid sensor detection (for notebook)
```bash
sudo sed -i '/^HandleLidSwitch/d;/^HandleLidSwitchDocked/d;/^HandleLidSwitchExternalPower/d' /etc/systemd/logind.conf && \
echo -e "HandleLidSwitch=ignore\nHandleLidSwitchDocked=ignore\nHandleLidSwitchExternalPower=ignore" | sudo tee -a /etc/systemd/logind.conf > /dev/null && \
sudo systemctl restart systemd-logind
```

## Updates packages and dependencies
```bash
sudo apt update
sudo apt upgrade -y
```

## Configure timezone
```bash
sudo dpkg-reconfigure tzdata
 ```

## Install btop for system monitoring
```bash
sudo apt install btop -y
```

---
### Install local hostname resolver on network
```bash
sudo apt install avahi-daemon -y
```
```bash
sudo systemctl enable avahi-daemon
sudo systemctl start avahi-daemon
```
Edit the `/etc/hosts` file to include the local machine name, if you haven't already done so.
```bash
sudo nano /etc/hosts
```

Add, if the lines below do not exist, replacing `host name` with the name of the machine on the local network:
```bash
127.0.0.1 localhost
127.0.1.1 <nome do host>
```

Example configured with `nas` as hostname:
```bash
127.0.0.1 localhost
127.0.1.1 nas
```

---
## Increase swap size to 2gb to prevent memory shortage (optional)

1. Disable swap to prevent an inconsistent state (WAIT)
```bash
sudo dphys-swapfile swapoff
```

2. Open the `/etc/dphys-swapfile` file to edit the swap size
```bash
sudo nano /etc/dphys-swapfile
```

3. Locate the `CONF_SWAPSIZE` line and replace the original value of `512` with `2048`

```bash
CONF_SWAPSIZE=2048
```

4. Re-enable swap (WAIT)
```bash
sudo dphys-swapfile swapon
```

5. Check with btop if the swap partition with a value of 2GB appeared
```bash
btop
```
---

## Add HDD parameters to enable sleep when inactive (if necessary)
### Get physical disk path and mountpoint
```bash
ls -l /dev/disk/by-id/ \
  | grep -E 'ata-|usb-' \
  | grep -v -- '-part' \
  | while read -r line; do
      BYID=$(echo "$line" | awk '{print $9}')
      DEV=$(echo "$line" | awk '{print $11}' | sed 's#\.\./\.\./##')
      MODEL=$(lsblk -ndo MODEL "/dev/$(basename "$DEV")")
      printf "%s -> %s [%s]\n" "$BYID" "$DEV" "$MODEL"
    done
```

Get the disk ID of the disk you need, is something like `ata-WDC_WD5000LPVX-00V0TT0....`

Test beforehand if the disk is correct with the comand below:
```bash
sudo hdparm -BC /dev/disk/by-id/<disk id>
```
If its the correct disk, then proceed.

### Create the usb disk standby script
```bash
sudo nano /etc/systemd/system/hdparm-disks.service
```
```bash
[Unit]
Description=Set hdparm settings for usb discs
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/hdparm -B 128 -S 120 /dev/disk/by-id/usb-damy_USB_3.0_Destop_H_00000000000000000000-0:0
ExecStart=/usr/sbin/hdparm -B 128 -S 120 /dev/disk/by-id/usb-ASMT_USB_3.0_Destop_H_0000000000AB-0:0

[Install]
WantedBy=multi-user.target
```

#### Enable at boot
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now hdparm-disks.service
```

#### Check status
```bash
systemctl status hdparm-disks.service
```

You should see
```bash
Active: inactive (dead) since ...
Process: ... ExecStart=... (code=exited, status=0/SUCCESS)
```

---
## Add disk mount in fstab (no raid) - Option 1
1. Check the disk UUID (unique disk identifier, which does not change regardless of how many disks are identified)

2. Get the disk partition using the disk tree for easy viewing
```bash
sudo lsblk
```

Example command output (`sda1` is the partition of interest)
```bash
nas@raspberrypi:~ $ sudo lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 465.8G  0 disk 
├─sda1        8:1    0 416.4G  0 part /mnt/dados
├─sda2        8:2    0   512M  0 part 
└─sda3        8:3    0  48.8G  0 part 
mmcblk0     179:0    0  59.5G  0 disk 
├─mmcblk0p1 179:1    0   512M  0 part /boot/firmware
└─mmcblk0p2 179:2    0    59G  0 part /
```


3. Get the UUID of the disk of interest
```bash
sudo blkid
```

Example command output (`239bab2a-450f-482a-84b8-f5af3cd29eb8` is the UUID of interest)
```bash
nas@raspberrypi:~ $ sudo blkid
/dev/mmcblk0p1: LABEL_FATBOOT="bootfs" LABEL="bootfs" UUID="4EF5-6F55" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="b6cb601c-01"
/dev/mmcblk0p2: LABEL="rootfs" UUID="ce208fd3-38a8-424a-87a2-cd44114eb820" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="b6cb601c-02"
/dev/sda2: LABEL_FATBOOT="BOOT" LABEL="BOOT" UUID="EEA9-58F6" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="c9ca2012-02"
/dev/sda3: LABEL="raspberry" UUID="04f16269-78bd-4ea1-a5de-783e39a13b53" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="c9ca2012-03"
/dev/sda1: UUID="239bab2a-450f-482a-84b8-f5af3cd29eb8" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="c9ca2012-01"
```


4. Add to fstab for automatic mounting (replace `<UUID>` with the correct one in your case). The `nofail 0 2` argument ensures that the boot process will continue if the partition cannot be successfully mounted.
```bash
sudo echo "UUID=<UUID> /mnt/data ext4 defaults,nofail 0 2 #boot even if volume is unavailable" > /etc/fstab
```

Example output from command with UUID `239bab2a-450f-482a-84b8-f5af3cd29eb8`
```bash
echo "UUID=239bab2a-450f-482a-84b8-f5af3cd29eb8 /mnt/data ext4 defaults,nofail 0 2 #boot even if volume is unavailable" | sudo tee -a /etc/fstab > /dev/null
```

5. Restart the fstab service to verify that the mount point is set correctly.
```bash
sudo mount -a
```

---
## Add disk mount in fstab (snapraid) - Option 2
1. Check the disk UUID (unique disk identifier, which does not change regardless of how many disks are identified)

2. Get the disk partition using the disk tree for easy viewing
```bash
sudo lsblk
```

Example command output (`sda1` is the partition of interest)
```bash
nas@raspberrypi:~ $ sudo lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 465.8G  0 disk 
├─sda1        8:1    0 416.4G  0 part /mnt/dados
├─sda2        8:2    0   512M  0 part 
└─sda3        8:3    0  48.8G  0 part 
mmcblk0     179:0    0  59.5G  0 disk 
├─mmcblk0p1 179:1    0   512M  0 part /boot/firmware
└─mmcblk0p2 179:2    0    59G  0 part /
```


3. Get the UUID of the disk of interest
```bash
sudo blkid
```

Example command output (`239bab2a-450f-482a-84b8-f5af3cd29eb8` is the UUID of interest)
```bash
nas@raspberrypi:~ $ sudo blkid
/dev/mmcblk0p1: LABEL_FATBOOT="bootfs" LABEL="bootfs" UUID="4EF5-6F55" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="b6cb601c-01"
/dev/mmcblk0p2: LABEL="rootfs" UUID="ce208fd3-38a8-424a-87a2-cd44114eb820" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="b6cb601c-02"
/dev/sda2: LABEL_FATBOOT="BOOT" LABEL="BOOT" UUID="EEA9-58F6" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="c9ca2012-02"
/dev/sda3: LABEL="raspberry" UUID="04f16269-78bd-4ea1-a5de-783e39a13b53" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="c9ca2012-03"
/dev/sda1: UUID="239bab2a-450f-482a-84b8-f5af3cd29eb8" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="c9ca2012-01"
```

4. Create mount points for every disk in the array
```bash
sudo mkdir -p /mnt/dados1000wd 
sudo mkdir -p /mnt/dados1000st 
sudo mkdir -p /mnt/dados500wd
```

5. Add automount to individual disks on fstab
```bash
#Individual disks
UUID=0c2ee9ed-8beb-480c-b9c7-78793549e99a /mnt/dados1000wd ext4 defaults,nofail,noatime 0 2 #boot even if volume is unavailable"
UUID=4af966dd-2a68-4b7f-981c-a6c98d28836c /mnt/dados1000st ext4 defaults,nofail,noatime 0 2 #boot even if volume is unavailable"
UUID=239bab2a-450f-482a-84b8-f5af3cd29eb8 /mnt/dados500wd ext4 defaults,nofail,noatime 0 2 #boot even if volume is unavailable"
```

<details>
  <summary>6. Formatting disks IF NECESSARY (OPTIONAL)</summary>

 Unmount disks, if mounted
```bash
sudo umount /dev/sdx* 2>/dev/null
```

Delete file system on disk (This will delete all data un that disk)

```bash
sudo wipefs -a /dev/sdx
```

Create new partition table
```bash
sudo parted -s /dev/sdx mklabel gpt
```
Create a single partition using entire disk
```bash
sudo parted -s /dev/sdx mkpart primary ext4 0% 100%
```

Format partition as EXT4
```bash
sudo mkfs.ext4 -L dados1 /dev/sdx1
```

Create partition path
```bash
sudo mkdir -p /mnt/dados1
```

Mount created partition
```bash
sudo mount /dev/sda1 /mnt/dados1
```

</details>

### Snapraid

Install necessary packages
```bash
sudo apt update
sudo apt install -y snapraid mergerfs
```

Configure snapraid array
```bash
sudo nano /etc/snapraid.conf
```

```bash
# Parity disk
parity /mnt/dados1000wd/snapraid.parity

# Data disk
data d1 /mnt/dados1000st
data d2 /mnt/dados500wd

# Content files
content /mnt/dados1000st/snapraid.content
content /mnt/dados500wd/snapraid.content
content /mnt/dados1000wd/snapraid.content

# Exclude from sync (optional)
exclude *.bak
exclude /tmp/
exclude cache/
exclude lost+found/
```

Create mergefs unified mount point (only on current session)
```bash
sudo mkdir -p /mnt/dados
sudo mergerfs /mnt/dados1000st:/mnt/dados500wd /mnt/dados -o defaults,allow_other,use_ino,category.create=mfs
```

Automount on boot (persistent)
```bash
sudo nano /etc/fstab
```

```bash
#Array
/mnt/dados1000st:/mnt/dados500wd /mnt/dados fuse.mergerfs defaults,allow_other,use_ino,category.create=mfs,nofail,noatime 0 2
```

Exec first snapraid sync (may take some hours to complete)
```bash
sudo snapraid sync
```

Exec fisrt snapraid scrub to garantee data integrity (may take some hours to complete again...)
```bash
sudo snapraid -p 100 -o 0 scrub
```

#### Schedule sync with cron


Script to stop all active containers (docker) and restart only the ones that were running before

```bash
sudo nano /usr/local/bin/snapraid_sync.sh
```

```bash
#!/bin/bash

#!/bin/bash

echo "[INFO] $(date) - Collecting active containers..."
# ACTIVE_CONTAINERS=$(docker ps -q)
ACTIVE_CONTAINERS=$(docker compose -f /home/rodrigo/nas-docker/compose/media.yaml -p media ps -q)

if [ -z "$ACTIVE_CONTAINERS" ]; then
    echo "[INFO] No containers running. Moving on..."
else
    echo "[INFO] Stopping containers..."
    for id in $ACTIVE_CONTAINERS; do
        name=$(docker inspect --format='{{.Name}}' "$id" | cut -c2-)
        echo "[INFO] Stopping container: $name ($id)"
        docker stop "$id" > /dev/null
    done
fi

echo -e "\n\n[INFO] Starting snapraid sync..."
/usr/bin/snapraid sync

echo -e "\n\n"
if [ -z "$ACTIVE_CONTAINERS" ]; then
    echo "[INFO] No containers to restart."
else
    echo -e "[INFO] Restarting containers..."
    for id in $ACTIVE_CONTAINERS; do
        name=$(docker inspect --format='{{.Name}}' "$id" | cut -c2-)
        echo "[INFO] Starting container: $name ($id)"
        docker start "$id" > /dev/null
    done
fi

# echo -e "\n\n[INFO] Waiting 1 minute before putting /dev/sdb to sleep..."
# sleep 60
# sudo hdparm -y /dev/sdb
# echo "[INFO] Forced /dev/sdb (internal SATA) to sleep"

echo "[INFO] Process completed at $(date)"

```

Make the script executable
```bash
sudo chmod +x /usr/local/bin/snapraid_sync.sh
```

Schedule on cron
```bash
sudo crontab -e
```

Runs sync daily at 3am
```bash
0 3 * * * /usr/local/bin/snapraid_sync.sh >> /var/log/snapraid-sync.log 2>&1
```

#### Schedule scrub with cron
Weekly Scrub on Sundays 4am, checks 25% of files

```bash
sudo nano /usr/local/bin/snapraid_scrub.sh
```

```bash
#!/bin/bash

# Coverage parameter (default: 25% of files)
SCRUB_PERCENT=25

echo "[INFO] $(date) - Collecting active containers..."
# ACTIVE_CONTAINERS=$(docker ps -q)
ACTIVE_CONTAINERS=$(docker compose -f /home/rodrigo/nas-docker/compose/media.yaml -p media ps -q)

if [ -z "$ACTIVE_CONTAINERS" ]; then
    echo "[INFO] No containers running. Continuing..."
else
    echo "[INFO] Stopping containers..."
    for id in $ACTIVE_CONTAINERS; do
        name=$(docker inspect --format='{{.Name}}' "$id" | cut -c2-)
        echo "[INFO] Stopping container: $name ($id)"
        docker stop "$id" > /dev/null
    done
fi

echo -e "\n\n[INFO] Starting snapraid scrub -p $SCRUB_PERCENT ..."
/usr/bin/snapraid scrub -p $SCRUB_PERCENT

echo -e "\n\n"
if [ -z "$ACTIVE_CONTAINERS" ]; then
    echo "[INFO] No containers to restart."
else
    echo "[INFO] Restarting containers..."
    for id in $ACTIVE_CONTAINERS; do
        name=$(docker inspect --format='{{.Name}}' "$id" | cut -c2-)
        echo "[INFO] Starting container: $name ($id)"
        docker start "$id" > /dev/null
    done
fi

# echo "[INFO] Waiting 1 minute before putting /dev/sdb to sleep..."
# sleep 60
# sudo hdparm -y /dev/sdb
# echo "[INFO] Forced /dev/sdb (internal SATA) to sleep"

echo "[INFO] Process completed at $(date)"
```

Make it executable
```bash
sudo chmod +x /usr/local/bin/snapraid_scrub.sh
```

Schedule in cron (e.g. every Sunday at 4am)
```bash
sudo crontab -e
```

```bash
4 0 * * 0 /usr/bin/snapraid scrub -p 25 > /var/log/snapraid-scrub.log 2>&1
```
---

## Docker
### Install docker dependencies
```bash
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
```

### Download the docker installation script
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
```

### Install docker
```bash
sudo sh get-docker.sh
```

### Adds the current user to the docker group
```bash
sudo usermod -aG docker $USER
```

### Check if docker was installed successfully
```bash
docker --version
```

### Add docker as a service (starts on boot)
```bash
sudo systemctl enable docker
```

### Install docker compose
```bash
sudo apt install -y python3-pip
# sudo pip3 install docker-compose #dependency error on raspberry pi
sudo apt install docker-compose -y
```

### Check if docker compose was installed successfully
```bash
docker compose --version
```

### Configure docker containers
Enter the folder containing the docker compose files ([compose](compose))

Run the command below to start the container stack you want to activate (portainer NEED to be the first because it contains all networks).
```bash
docker compose -f <file> -p <stack name> up -d
``` 
```bash
cd compose
docker compose -f portainer.yaml -p portainer up -d
docker compose -f media.yaml -p media up -d
docker compose -f homeassistant.yaml -p homeassistant up -d
``` 

---

## File sharing (SMB)
### Install file sharing via smb (with wsdd to identify on the network in win11)
```bash
sudo apt install samba wsdd -y
```

### Add user for file sharing
```bash
sudo smbpasswd -a <user> #add samba user
```

### Configure file sharing
Copy the smb.conf file ([smb.conf](smb.conf)) already configured to the `/etc/samba` directory with the command below
```bash
sudo cp smb.conf /etc/samba/smb.conf
```

## Tailscale
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --advertise-exit-node --accept-dns --advertise-routes=192.168.1.0/24
```
`Advertise exit node` to add this machine as exit node (like a proxy) <br>
`Accept dns` do redirect dns requests to internal lan dns <br>
`Advertive routes` to acess local lan
