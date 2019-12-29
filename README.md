# Home Assistant integration: Xiaomi Vacuum v1 & v2

If you're here probably you want to integrate Xiaomi Vacuum v1 & v2 with Home Assistant without loosing some useful functions such as live maps.


## On the Vacuum

###### Root the vacuum

If you don't know how to gain SSH access to your vacuum please read [the article](https://github.com/dgiese/dustcloud/wiki/VacuumRobots-manual-update-root-Howto) on dustcloud:


###### Install on the vacuum

- Create SSH keypair with the command: **ssh-keygen**

- Copy the SSH public key to the Home Assistant host with the command: **ssh-copy-id -p ##HOMEASSISTANT_SSHPORT## homeassistant@##HOMEASSISTANT_IPADDRESS##**

- Copy the file vacuum/etc/rc.local to your vacuum, replacing the existing /etc/rc.local.

- Copy the file vacuum/opt/rockrobo/scripts/maps_to_ha.sh to your vacuum, at the destination /opt/rockrobo/scripts/maps_to_ha.sh.
    * Fill RHOST with ##HOMEASSISTANT_IPADDRESS##
    * Fill RPORT with ##HOMEASSISTANT_SSHPORT##

- Make sure to give to the file the right permissions with the command: **chown 755 /opt/rockrobo/scripts/maps_to_ha.sh**.
- Resart the vacuum. (Type `reboot` when logged in the the vacuum.)

###### Install on Home Assistant (Hass.io)

The directories in the scripts are preconfigured according to the directories in a Hass.io Home Assistant installation. So it should not be required to modify the scripts.

- Use the "SSH & Web Terminal" AddOn to create the directories **/config/scripts**, **/config/www**, **/config/vacuum** if not exist. Make sure to change the ownership of the **/config/vacuum** directory to the user **hassio**.
```
$ mkdir /config/scripts
$ mkdir /config/www
$ mkdir /config/vacuum
$ chown hassio /config/vacuum
```
**Checkpoint 1:** At this point of the installation you can try out whether the setup of the vacuum was done correctly. To do so, just start the cleaning of the vacuum. After a few seconds, files (SLAM_fprintf.log and navmap<< some number >>.ppm) will appear in the **/config/vacuum** folder of your Home Assistant.

![Image of Vacuum folder](/images/vacuum_folder.png?raw=true)

If this is not the case, stop your vacuum, ssh login to your vacuum, execute `/opt/rockrobo/scripts/maps_to_ha.sh` manually, start the vacuum and take a look at the output of the scripts. (Common problems: wrong IP address of the Home Assistant configured, ssh key not properly set up, permission problem with /config/vacuum (ownership not changed)). 
**You need to get this running before going on with the installation.***

- Copy the script homeassistant/config/scripts/build_maps.py to /config/scripts/build_maps.py.
- Copy the script homeassistant/config/scripts/incrond_vacuum_maps.sh to /config/scripts/incrond_vacuum_maps.sh.
- Make sure to give to the file the right permissions with the command: **chmod 755 /config/scripts/incrond_vacuum_maps.sh**.
- Add the following to your **/config/configuration.yaml**
```yaml
shell_command:
  xiaomi_miio_build_map: /config/scripts/incrond_vacuum_maps.sh
```

**Checkpoint 2:** At this point you can try out whether the shell/python scripts to create the image are working. Assuming you performed checkpoint 1 and thus have a file **navmap<< some number >>.ppm** in your **/config/vacuum** with a size of 3072kB (you need to let the vacuum clean for a while): Just trigger the service `shell_command.xiaomi_miio_build_map`, a file **navmap.png** will be created in the **/config/www** directory of your Home Assistant, which is the image of the created map.
If the image is not created, set the log level for Home assistant shell commands to debug (restart necessary) and trigger the service again. You should see log outputs in the logs of Home Assistant.
```
logger:
  default: warn
  logs:
    homeassistant.components.shell_command: debug
```
- Add the following automation to your **/config/automations.yaml** (update the entity_id of your vacuum). Whenever the vaccum changes from `docked` to `cleaning` the automation will trigger the script to update the map.
```
- id: vacuum_map_trigger
  alias: Vacuum Map Trigger
  trigger:
  - entity_id: vacuum.pikachu
    from: docked
    platform: state
    to: cleaning
  action:
  - entity_id: script.vacuum_map_update
    service: script.turn_on
```
- Add the following scripts to your **/config/scripts.yaml** (update the entity_id of your vacuum). The `vacuum_map_update` script triggers the shell command to update the map, then waits 5s (this prevents a lock between the scripts when they call each other to fast) and then triggers the `vacuum_map_loop` script which checks whether the vacuum is still cleaning and if so triggers `vacuum_map_update` and else does nothing (thus stopping the map update loop).
```
'vacuum_map_update':
  alias: Vacuum Map Update
  sequence:
  - alias: ''
    data: {}
    service: shell_command.xiaomi_miio_build_map
  - delay: 00:00:05
  - entity_id: script.vacuum_map_loop
    service: script.turn_on
'vacuum_map_loop':
  alias: Vacuum Map Loop
  sequence:
  - condition: state
    entity_id: vacuum.pikachu
    state: cleaning
  - data: {}
    entity_id: script.vacuum_map_update
    service: script.turn_on
```

**Checkpoint 3:** At this point everything is set up. Try out by starting the cleaning of your vacuum. You should see how new files **navmap<< some number >>.ppm** appear in your **/config/vacuum** directory and that **/config/www/navmap.png** is updated regularly. 

###### Show the map on Home Assistant

Add the following component to your Home Assistant **/config/configuration.yaml**:

```
camera:
  - platform: generic
    name: Vacuum Map
    still_image_url: http://127.0.0.1:8123/local/navmap.png
    content_type: image/png
    framerate: 1
```
**Note:** Change `http` to `https` in case you set up an ssl key (you also might need to set `verify_ssl: false` as the ssl key is only valid for your domain).

In your lovelace configuration, add a new `picture_entity` card (update the entity_id of your vacuum).
```
name: Vacuum Map
camera_image: camera.vacuum_map
entity: vacuum.pikachu
type: picture-entity
```
The result should look like this: :)

![Image of Vacuum map](/images/vacuum_map.png?raw=true)
