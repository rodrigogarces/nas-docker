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
This command ensure correct permissions to edit `passwords` file, add the desired user and revert permissions to default to prevent permissions conflicts.


## Home assistant (container)
### Install integrations (Configurations -> Devices and services -> add integrations)
1. MQTT integration (used by tasmota)
2. Tasmota integration
3. Zigbee integration (if you have a compatible zigbee coordinator)


### Install HACS
Install with this single command on a regular terminal
```bash
docker exec -it homeassistant sh -c "wget -O - https://get.hacs.xyz | bash -"
```

Restart homeassistant container to ensure that all modifications are applied.
```bash
docker stop homeassistant
docker start homeassistant
```
Go again to Configurations -> Devices and services -> add integrations. Hit <kbd>Ctrl</kbd> + <kbd>F5</kbd> to force reload without cache. This step is nedded to update integrations list and show HACS.

Install under the same menu this integrations:
* MQTT
* Tasmota


### HACS (Home assistant) addons
Go to HACS on left menu and install this addons:


* WebRTC camera
* apexcharts-card
* auto-entities
* layout-card
* Advanced Camera Card


### Zigbee 2 MQTT
To add usb support on zigbee container (USB docker permission).\
`sudo usermod -aG dialout $USER`


To map the zigbee coodinator on  zigbee2mqtt\
Get the USB device ID of the coordinator using this command and alter on file [compose/homeassistant.yaml](compose/homeassistant.yaml#L50)
```bash
ls -l /dev/serial/by-id/
```

Example of a coordinator ID
```bash
/dev/serial/by-id/usb-Silicon_Labs_CP2102_USB_to_UART_Bridge_Controller_0001-if00-port0
```
Ensure that is coordinator gets a fixed USB mapping, that persist no matter what USB port you use
