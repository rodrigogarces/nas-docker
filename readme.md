## Setup a NAS server from scratch with ubuntu server

### Disable lid sensor detection (for notebook)
```bash
sudo sed -i '/^HandleLidSwitch/d;/^HandleLidSwitchDocked/d;/^HandleLidSwitchExternalPower/d' /etc/systemd/logind.conf && \
echo -e "HandleLidSwitch=ignore\nHandleLidSwitchDocked=ignore\nHandleLidSwitchExternalPower=ignore" | sudo tee -a /etc/systemd/logind.conf > /dev/null && \
sudo systemctl restart systemd-logind
```

### Updates packages and dependencies
```bash
sudo apt update
sudo apt upgrade -y
```

### Install btop for system monitoring
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
### Increase swap size to 2gb to prevent memory shortage (optional)

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
### Add disk mount in fstab
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
Enter the folder containing the docker compose file ([docker-compose.yaml](docker-compose.yaml))

Run the command below to start the containers
```bash
docker compose up
``` 

---

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