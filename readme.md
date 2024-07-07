# RAK5146 gateway and ChirpStack on Raspberry Pi Zero 2 W

*This tutorial was initially written on 07/07/2024. Before following the tutorial, you may want to check if [ChirpStack Gateway OS](https://www.chirpstack.io/docs/chirpstack-gateway-os/hardware-support.html) supports the Pi Zero 2, or if [RAK's automated script](https://github.com/RAKWireless/rak_common_for_gateway) has been updated to work with ChirpStack v4.*

This tutorial shows how to setup a Raspberry Pi Zero 2 W as a LoRaWAN gateway and server, using the RAK5146 module as the gateway and ChirpStack as the server. The tutorial focuses on using the headless version of the Raspberry OS.

At the time of writing, it was not possible to use ChirpStack's image ([ChirpStack Gateway OS](https://www.chirpstack.io/docs/chirpstack-gateway-os/hardware-support.html)), as the Raspberry Pi Zero 2 W was not supported. Simply using Raspberry Pi Zero's image does not seem to work.

Another option is to use [RAK's automated script](https://github.com/RAKWireless/rak_common_for_gateway) to completely install ChirpStack, the packet forwarder and all the required packages. The script runs smoothly and works, except that the GPS did not seem to work. Moreover, the script uses ChirpStack v3, and it would be preferable to use ChirpStack v4 for all the updates and improvements.

## Flashing the SD card

Raspberry Pi Imager v1.8.5 was used the SD card with the lite version of the Raspberry Pi OS. The following settings were used:

- Device: Raspberry Pi Zero 2 W
- OS: Raspberry Pi OS Lite (64-bit) Bookworm, released 2024-07-04

System version when ssh'ing to the Pi:

```
Linux raspberrypi 6.6.31+rpt-rpi-v8 #1 SMP PREEMPT Debian 1:6.6.31-1+rpt1 (2024-05-29) aarch64
```

## Raspi config

Log in to your Pi and run

```sh
sudo raspi-config
```
In the `Interface Options`

- Enable SPI
- Enable I2C
- In `Serial Port`, disable login shell and enable the serial port hardware

No reboot required at the moment.

## Disable bluetooth

Bluetooth needs to be disabled because it is in the same port as the GPS module of the concentrator. (For more information, [check this discussion](https://forum.chirpstack.io/t/does-chirpstack-os-on-rpi-have-bluetooth/8824).)

To disable bluetooth, run

```sh
sudo nano /boot/firmware/config.txt
```

At the end of the file, insert a new line and add

```
dtoverlay=disable-bt
```

Save the file, and then run

```sh
sudo systemctl disable hciuart
```

## Increase swap size

Sometimes the Pi Zero can get stuck during updates because the default swap size is too small (see [here](https://forums.raspberrypi.com/viewtopic.php?t=359240) and [here](https://forums.raspberrypi.com/viewtopic.php?t=369819)).

To increase the swap size, run

```sh
sudo nano /etc/dphys-swapfile
```

and change `CONF_SWAPSIZE=200` to `CONF_SWAPSIZE=2048`. Save the file and reboot:

```sh
sudo reboot -h now
```

## Update Pi

Update Pi by running

```sh
sudo apt update
sudo apt upgrade
sudo apt full-upgrade
sudo apt autoremove
```

## Install Git

Raspberry OS Lite does come with Git, so we have to install it:

```sh
sudo apt install git-all
```

## Install ChirpStack

More detailed information on how to install ChirpStack can be found in the [official documentation](https://www.chirpstack.io/docs/). The steps here were based on the steps described [here](https://www.chirpstack.io/docs/getting-started/debian-ubuntu.html).

Start by installing the dependencies

```sh
sudo apt install mosquitto mosquitto-clients redis-server redis-tools postgresql
```

Run postgres

```sh
sudo -u postgres psql
```

Execute the queries in sequence
```sql
create role chirpstack with login password 'chirpstack';

create database chirpstack with owner chirpstack;

\c chirpstack

create extension pg_trgm;

\q
```

### Setting up the software repository

Run

```sh
sudo apt install apt-transport-https dirmngr
```

```sh
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 1CE2AFD36DBCCA00
```

```sh
sudo echo "deb https://artifacts.chirpstack.io/packages/4.x/deb stable main" | sudo tee /etc/apt/sources.list.d/chirpstack.list
```

```sh
sudo apt update
```

### Installing the gateway bridge
Now, install the gateway bridge

```sh
sudo apt install chirpstack-gateway-bridge
```

Adapt the bridge's configuration file to your region by running  
```sh
sudo nano /etc/chirpstack-gateway-bridge/chirpstack-gateway-bridge.toml
```

and under `[integration.mqtt]`, prepend your region to `event_topic_template` and `command_topic_template`. For example, for EU868, this would be:

```
event_topic_template="eu868/gateway/{{ .GatewayID }}/event/{{ .EventType }}"
```
```
command_topic_template="eu868/gateway/{{ .GatewayID }}/command/#"
```

Save the file and enable and start the bridge service:

```sh
sudo systemctl enable chirpstack-gateway-bridge
```

```sh
sudo systemctl start chirpstack-gateway-bridge
```

### Installing ChirpStack

Install ChirpStack by running
```sh
sudo apt install chirpstack
```

Edit `chirpstack.toml` to enable your region:
```sh
sudo nano /etc/chirpstack/chirpstack.toml
```

Under `[network]`, edit `enabled_regions` to contain your region.

After saving the file, enable and start the ChirpStack service:

```sh
sudo systemctl enable chirpstack
```

```sh
sudo systemctl start chirpstack
```

At this point, the web interface should be available. More information [here](https://www.chirpstack.io/docs/chirpstack-gateway-os/guides/getting-started.html#access-chirpstack-web-interface).

## Installing the packet forwarder

The last step is installing the packet forwarder. We'll install the HAL library for the SX1302 module. Although the RAK5146 module has the SX1303 chip, the HAL should work the same. In a directory of your preference, clone the HAL library for the SX1302 module (the tutorial was tested with version `V2.1.0r7`):

```sh
git clone https://github.com/brocaar/sx1302_hal
```

This is a fork of [Semtech's original HAl library](https://github.com/Lora-net/sx1302_hal), but has additional fixes related to the onboard temperature sensor.

Navigate to the `sx1302_hal` directory and build the files:

```sh
make clean all
```

Then, install:

```sh
make install
```

```sh
make install_conf
```

The user's password will be required several times during installation. Check the [official Semtech instructions](https://github.com/Lora-net/sx1302_hal) for a way around this, or just type in the password as many times as required. 

### Creating the reset and EUI update files
After installation is complete, we need to create two files in the `packet_forwarder` folder. One file will be used to reset the concentrator hardware, and the other one will be used to update the EUI of the gateway based on Pi's MAC. 

Navigate to the `packet_forwarder` folder and create the reset file:

```sh
nano reset_lgw.sh
```

Paste the following contents into the file:

```sh
#!/bin/sh

# This script is intended to be used on SX1302 CoreCell platform, it performs
# the following actions:
#       - export/unpexort GPIO23 and GPIO18 used to reset the SX1302 chip and to enable the LDOs
#
# Usage examples:
#       ./reset_lgw.sh stop
#       ./reset_lgw.sh start

# GPIO mapping has to be adapted with HW
#

SX1302_RESET_PIN=17

WAIT_GPIO() {
    sleep 0.1
}

init() {
    # setup GPIOs
    #echo "$SX1302_RESET_PIN" > /sys/class/gpio/export; WAIT_GPIO

    # set GPIOs as output
    #echo "out" > /sys/class/gpio/gpio$SX1302_RESET_PIN/direction; WAIT_GPIO
    pinctrl set $SX1302_RESET_PIN op dl; WAIT_GPIO; WAIT_GPIO;
}

reset() {
    #echo "CoreCell reset through GPIO$SX1302_RESET_PIN..."

    #echo "1" > /sys/class/gpio/gpio$SX1302_RESET_PIN/value; WAIT_GPIO
    #echo "0" > /sys/class/gpio/gpio$SX1302_RESET_PIN/value; WAIT_GPIO
    pinctrl set $SX1302_RESET_PIN op dh; WAIT_GPIO
    pinctrl set $SX1302_RESET_PIN op dl; WAIT_GPIO
}

term() {
    # cleanup all GPIOs
    #if [ -d /sys/class/gpio/gpio$SX1302_RESET_PIN ]
    #then
    #    echo "$SX1302_RESET_PIN" > /sys/class/gpio/unexport; WAIT_GPIO
    #fi
    pinctrl set $SX1302_RESET_PIN ip pu; WAIT_GPIO
}

case "$1" in
    start)
    term # just in case
    init
    reset
    ;;
    stop)
    reset
    term
    ;;
    *)
    echo "Usage: $0 {start|stop}"
    exit 1
    ;;
esac

exit 0
```

This file is a modified version of [this one](https://github.com/RAKWireless/rak_common_for_gateway/blob/3f210eeca37e4e0fee16f1d56591611cf2f6e472/lora/rak5146/reset_lgw.sh
). The main change is related to setting the I/O pins; in the newest Bookworm version of the Pi OS, `echo` cannot be used to set I/O pins anymore, `pinctrl` must be used instead.

After creating the file, change its permission:

```sh
chmod +x reset_lgw.sh
```

Now, create the second file:

```sh
nano update_gwid.sh
```

and paste the following contents into it:

```sh
#!/bin/bash

# This script is a helper to update the Gateway_ID field of given
# JSON configuration file, as a EUI-64 address generated from the 48-bits MAC
# address of the device it is run from.
#
# Usage examples:
#       ./update_gwid.sh ./local_conf.json

iot_sk_update_gwid() {
    # get gateway ID from its MAC address to generate an EUI-64 address
    GATEWAY_EUI_NIC="eth0"
    if [[ `grep "$GATEWAY_EUI_NIC" /proc/net/dev` == "" ]]; then
        GATEWAY_EUI_NIC="wlan0"
    fi
    
    if [[ `grep "$GATEWAY_EUI_NIC" /proc/net/dev` == "" ]]; then
        GATEWAY_EUI_NIC="usb0"
    fi

    if [[ `grep "$GATEWAY_EUI_NIC" /proc/net/dev` == "" ]]; then
       echo "ERROR: No network interface found. Cannot set gateway ID."
       exit 1
    fi

    GWID_MIDFIX="fffe"
    GWID_BEGIN=$(ip link show $GATEWAY_EUI_NIC | awk '/ether/ {print $2}' | awk -F\: '{print $1$2$3}')
    GWID_END=$(ip link show $GATEWAY_EUI_NIC | awk '/ether/ {print $2}' | awk -F\: '{print $4$5$6}')

    # replace last 8 digits of default gateway ID by actual GWID, in given JSON configuration file
    sed -i 's/\(^\s*"gateway_ID":\s*"\).\{16\}"\s*\(,\?\).*$/\1'${GWID_BEGIN}${GWID_MIDFIX}${GWID_END}'"\2/' $1

    echo "Gateway_ID set to "$GWID_BEGIN$GWID_MIDFIX$GWID_END" in file "$1
}

if [ $# -ne 1 ]
then
    echo "Usage: $0 [filename]"
    echo "  filename: Path to JSON file containing Gateway_ID for packet forwarder"
    exit 1
fi 

iot_sk_update_gwid $1

exit 0
```

This file was obtained from [here](https://github.com/RAKWireless/rak_common_for_gateway/blob/3f210eeca37e4e0fee16f1d56591611cf2f6e472/lora/update_gwid.sh).

The script basically uses Pi's MAC to generate the EUI for the concentrator, writing it to  the `global_conf.json` file. There are similar scripts available, but this one works for Pi Zero 2 W because it gets the MAC from `wlan0`, while similar scripts will try to get it only from `eth0` or `eth1`, which are not available in the Pi Zero 2 W.


### Create the `global_conf.json` file

Still in the `packet_forwarder` folder, make a copy of the proper configuration file for your region and gateway type. For EU868 region with SPI interface, run:

```sh
cp global_conf.json.sx1250.EU868 global_conf.json
```

Edit the newly created file

```sh
nano global_conf.json
```

and at the very end, in the `gateway_conf_settings`, set the default server ports to 1700 (`serv_port_up` and `serv_port_down`), and change ` gps_tty_path` to `"/dev/ttyAMA0"/`. Then, run the following:

```sh
bash ./update_gwid.sh ./global_conf.json
```

This will set the EUI of the gateway. You can always find the EUI of the gateway in the `global_conf.json` file.

Finally, run the `packet_forwarder`:

```sh
./lora_pkt_fwd -c global_conf.json
```

At this point, the gateway should be functional, and it should be possible to add it in the ChirpStack web application and even see the GPS coordinates and maybe some LoRa frames (It might take a while for the GPS to start working). Stop the packet forwarder by pressing CTRL+C.

### Creating the packet forwarder service

Running the packet forwarder as described above will only run it once. If you shutdown/reboot the Pi, the packet forwarder will not start again automatically. To start it automatically at boot, it is necessary to create an initialization file and add the service in `systemd`.

Start by creating the initialization file in the `packet_forwarder` folder:

```sh
nano pkt_fwd_init.sh
```

Copy the following contents into the file:

```sh
#!/bin/bash
cd "$(dirname "$0")"

./lora_pkt_fwd -c global_conf.json
```

Then, make it executable:

```sh
sudo chmod +x pkt_fwd_init.sh
```

Now, create the service file for the packet forwarder:

```sh
sudo nano /etc/systemd/system/lora-pkt-fwd.service
```

and write the contents:

```sh
[Unit]
Description=Semtech packet-forwarder

[Service]
WorkingDirectory=/home/pi/sx1302_hal/packet_forwarder
ExecStart=/home/pi/sx1302_hal/packet_forwarder/pkt_fwd_init.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Make sure that `WorkingDirectory` and `ExecStart` match the location of your files.

Finally, enable and start the process:


```sh
sudo systemctl enable lora-pkt-fwd.service
sudo systemctl start lora-pkt-fwd.service
```

