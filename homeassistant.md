# Setup home assistant (container)

## Mosquitto configuration (container)
```bash
#create config folder
sudo mkdir -p mosquitto/config

#give current user permission to edit
sudo chown $USER mosquitto/config 

#create mosquitto configuration file
sudo echo 'listener 1883
allow_anonymous false
password_file /mosquitto/config/passwords

listener 9001
protocol websockets' > mosquitto/config/mosquitto.conf

#revert permissions to original
sudo chown 1883:1883 mosquitto/config 

#create and correct permissions on mosquitto password file
sudo touch mosquitto/config/passwords
sudo chmod o-r mosquitto/config/passwords
sudo chown root:root mosquitto/config/passwords
```

With both `mosquitto.conf` and `passwords`(currently blank), the container will start correctly.

Start manually mosquitto contaniner (via [home assistant](compose/homeassistant.yaml) stack)(`docker compose -f compose/homeassistant.yaml -p homeassistant up -d`).

Exec this command on a regular terminal to create a mosquitto user insede the mosquitto container that your smart devices will use. On this example is created a `mosquitto-ha` user. You need to prompt a password for this user. Take note, you will need this user and password for every device you want to integrate on the mqtt broker.

```bash
docker exec -it mosquitto sh -c "\
chown root:root mosquitto/config/passwords && \
mosquitto_passwd -c /mosquitto/config/passwords mosquitto-ha && \
chown 1883:1883 mosquitto/config/passwords" #prompts for password
```
This command ensure correct permissions to edit `passwords`file, add the desired user and revert permissions to default to prevent permissions conflicts.


## Home assistant (container)
### Install integrations (Configurations -> Devices and services -> add integrations)
1. MQTT integration (used by tasmota)
2. Tasmota integration
3. Zigbee integration (if you have a compatible zigbee hub)


### Install HACS
Install with this simgle command on a regular terminal
```bash
docker exec -it homeassistant sh -C "wget -O - https://get.hacs.xyz | bash -"
```

Restart homeassistant container to ensure that all modifications are applied.
```bash
docker restart homeassistant
```
Go again to Configurations -> Devices and services -> add integrations. Hit <kbd>Ctrl</kbd> + <kbd>F5</kbd> to force reload without cache. This step is nedded to update integrations list and show HCAS.

UNDER CONSTRUCTION