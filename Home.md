# TL;DR
**What**: doc to play sound on PI  
**How**: via Bluetooth + PulseAudio  
**Aim**: Synced Multi-room Audio receiver/issuer that works with as many equipment as possible  

# Versions
This doc will often update to add new features and stuff.  
See 
* [CHANGELOG](https://github.com/vgallissot/raspberry-pi-audio-system.doc/blob/master/CHANGELOG.md)
* [ROADMAP](https://github.com/vgallissot/raspberry-pi-audio-system.doc/blob/master/ROADMAP.md)

# ToC
* [TL;DR](#tldr)
* [Versions](#versions)
* [ToC](#toc)
* [Configuration](#configuration)
  * [Hardware](#hardware)
  * [Software](#software)
    * [OS](#os)
    * [Configure Hifiberry AMP  (optionnal)](#configure-hifiberry-amp-optionnal)
    * [Force static names for wlan interfaces](#force-static-names-for-wlan-interfaces)
    * [Disable default wireless and enable the dongle](#disable-default-wireless-and-enable-the-dongle)
    * [Test sound](#test-sound)
    * [Bluetooth support](#bluetooth-support)
    * [Configure PulseAudio](#configure-pulseaudio)
    * [Custom Bluetooth name](#custom-bluetooth-name)
    * [Manual pairing](#manual-pairing)
    * [Notification sounds](#notification-sounds)
      * [Sounds](#sounds)
      * [Udev triggered script](#udev-triggered-script)
      * [Allow users to play sound](#allow-users-to-play-sound)
      * [Add udev rule](#add-udev-rule)
* [Result](#result)
* [Details on project](#details-on-project)
  * [Why Bluetooth?](#why-bluetooth)
  * [Why PulseAudio?](#why-pulseaudio)

# Configuration
## Hardware
I use:

1. Raspberry PI 3
2. Hifiberry AMP+
3. 16GB SD Card (8GB is enough)
4. USB WiFi dongle (because there's an issue on using Bluetooth and WiFi together)
5. Power Supply: Meanwell GS60A18-P1J
6. Speakers: 2x Scandyna MicroPod SE (not a good idea, one should prefer Eltax Monitor III)

## Software
### OS
I use current stable release of Raspbian: Jessie  
You just have to install it on a SD flash card:

1. [Download latest Raspbian LITE release](https://www.raspberrypi.org/downloads/raspbian/)
2. [Follow steps to install it on your SD card](https://www.raspberrypi.org/documentation/installation/installing-images/linux.md)

### Configure Hifiberry AMP+ (optionnal)
* [Go to the dedicated page](https://github.com/vgallissot/raspberry-pi-audio-system.doc/wiki/Amplifier)  

### Force static names for wlan interfaces
Whitelist wlan devices.
```
vim /lib/udev/rules.d/75-persistent-net-generator.rules
8------------------------------------------------------------------------8
- KERNEL!="ath*|msh*|ra*|sta*|ctc*|lcs*|hsi*", \
+ KERNEL!="ath*|wlan*[0-9]|msh*|ra*|sta*|ctc*|lcs*|hsi*", \
8------------------------------------------------------------------------8
```
``/etc/udev/rules/70-persistent-net.rules`` file will be created after reboot containing fixed wlan names according to MAC addresses.  


### Disable default wireless and enable the dongle
Unload the kernel module of raspberry3 wireless card
```
modprobe -r -v brcmfmac
vim /etc/modprobe.d/raspi-blacklist.conf
8------------------------------------------------------------------------8
#wifi
blacklist brcmfmac
#blacklist brcmutil
8------------------------------------------------------------------------8
```

Configure wlan0 which is now the only wlan card
```
vim /etc/network/interfaces
8------------------------------------------------------------------------8
auto wlan0
allow-hotplug wlan0
iface wlan0 inet manual
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
8------------------------------------------------------------------------8
```
Edit wpa_supplicant.conf according to your WiFi specs.  

Reboot and be ready
```
shutdown -r now
```

### Test sound
scp some mp3 file to the PI and play it: 

```
apt-get update
apt-get install mplayer -y
amixer sset 'Master'  -- 70%
mplayer 02.\ I\ Fink\ U\ Freeky.mp3
```

=> Sound should be OK.  
``aplay -l`` should output only 1 card.


### Bluetooth support
Install bluez + dependency to pulseaudio
```
apt-get install -y pulseaudio-module-bluetooth bluez-tools
```

Make the Bluetooth up at startup
```
vim /etc/systemd/system/bluetooth.target.wants/bluetooth.service
8------------------------------------------------------------------------8
    - ExecStart=/usr/lib/bluetooth/bluetoothd
    + ExecStart=/usr/lib/bluetooth/bluetoothd --noplugin=sap
8------------------------------------------------------------------------8
```

Configure class of BT. It defines what capabilities your BT will have (audio only / audio + micro / etc.).
```
vim /etc/bluetooth/main.conf
8------------------------------------------------------------------------8
# Changing name here has no effect
Class = 0x0c0420
8------------------------------------------------------------------------8
```

### Configure PulseAudio
We need the 'pulse' user to access bluetooth via dbus
```
vim /etc/dbus-1/system.d/pulseaudio-bluetooth.conf
8------------------------------------------------------------------------8
<busconfig>

  <policy user="pulse">
    <allow send_destination="org.bluez"/> 
  </policy>

</busconfig>
8------------------------------------------------------------------------8
```

Restart dbus with ``systemctl restart dbus``


PulseAudio on a desktop is not started as a deamon ([Check out why](https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/WhatIsWrongWithSystemWide/)). For a standalone sound system, we need it to be started as a system-wide daemon.
```
vim /etc/systemd/system/pulseaudio.service
8------------------------------------------------------------------------8
[Unit]
Description=Pulse Audio

[Service]
Type=simple
ExecStart=/usr/bin/pulseaudio --system --disallow-exit --disable-shm

[Install]
WantedBy=multi-user.target
8------------------------------------------------------------------------8

systemctl daemon-reload
```

Load bluetooth modules for pulseaudio
```
vim /etc/pulse/system.pa
8------------------------------------------------------------------------8
### Automatically load driver modules for Bluetooth hardware
.ifexists module-bluetooth-policy.so
load-module module-bluetooth-policy
.endif

.ifexists module-bluetooth-discover.so
load-module module-bluetooth-discover
.endif
8------------------------------------------------------------------------8
```

As the PI is used as a dedicated device for audio : No Restrictions on PI's consumptions = Best Audio quality possible (improvements are welcome)
```
vim /etc/pulse/daemon.conf
8------------------------------------------------------------------------8
cpu-limit = no
high-priority = yes
nice-level = -18
realtime-scheduling = yes
realtime-priority = 5

resample-method = ffmpeg
enable-remixing = no
enable-lfe-remixing = no

default-sample-format = s32le
default-sample-rate = 44100
alternate-sample-rate = 48000
default-sample-channels = 2
8------------------------------------------------------------------------8

systemctl --system enable pulseaudio.service
systemctl --system start pulseaudio.service
```

### Custom Bluetooth name
```
echo 'PRETTY_HOSTNAME=$myawesomename' >> /etc/machine-info
service bluetooth restart
```

### Manual pairing
example from [archlinux doc](https://wiki.archlinux.org/index.php/bluetooth#Bluetoothctl)
```
bluetoothctl
# bluetoothctl 
[NEW] Controller 00:10:20:30:40:50 pi [default]
[NEW] Device 00:12:34:56:78:90 myLino
[CHG] Device 00:12:34:56:78:90 LegacyPairing: yes
[bluetooth]# pair 00:12:34:56:78:90
Attempting to pair with 00:12:34:56:78:90
[CHG] Device 00:12:34:56:78:90 Connected: yes
[CHG] Device 00:12:34:56:78:90 Connected: no
[CHG] Device 00:12:34:56:78:90 Connected: yes
Request PIN code
[agent] Enter PIN code: 1234
[CHG] Device 00:12:34:56:78:90 Paired: yes
Pairing successful
[CHG] Device 00:12:34:56:78:90 Connected: no
[bluetooth]# connect 00:12:34:56:78:90
Attempting to connect to 00:12:34:56:78:90
[CHG] Device 00:12:34:56:78:90 Connected: yes
Connection successful
```

### Notification sounds
Play some sounds when:
* PI starts
* PI shuts down
* Bluetooth device connects
* Bluetooth device disconnects

#### Sounds
scp some **short** samples to PI.  
I used ``/usr/local/share/sounds/`` dir.  
I force a lower volume to avoid unwanted_crazy_loud notifications.  

#### Udev triggered script
I ran ``udevadm monitor --environment`` to monitor udev and kernel events.  

```
vim /usr/local/bin/bluetooth-udev
8------------------------------------------------------------------------8
#!/bin/bash
# This script will play sound depending of event

NORMAL_SOUND_LEVEL=$(/usr/bin/amixer get Master | awk '$0~/%/{print $5}' |head -1 | tr -d '[]')
NOTIFICATION_SOUND_LEVEL=55%

# Reduce sound
/usr/bin/amixer sset 'Master' -- ${NOTIFICATION_SOUND_LEVEL}

# Trigger correct sound for each situation
if [ "$1" == 'device' ]
then
    case $ACTION in
        add)  # when device connects
            /usr/bin/mplayer --really-quiet /usr/local/share/sounds/portal/Hello.ogg
        ;;
        remove)  # when device disconnects
            /usr/bin/mplayer --really-quiet /usr/local/share/sounds/portal/Goodbye.ogg
        ;;
        *)  # unknown action of device
            /usr/bin/mplayer --really-quiet /usr/local/share/sounds/portal/Who_are_you.ogg
        ;;
    esac
elif [ "$1" == 'card' ]
then
    case $ACTION in
        add)  # when card is activated
            /usr/bin/mplayer --really-quiet /usr/local/share/sounds/portal/Activated.ogg
        ;;
        remove)  # when card is desactivated
            /usr/bin/mplayer --really-quiet /usr/local/share/sounds/portal/Shutting_down.ogg
        ;;
        *)   # unknown action of card
            /usr/bin/mplayer --really-quiet /usr/local/share/sounds/portal/Space.ogg
        ;;
    esac
fi

# Put sound at its original level
/usr/bin/amixer sset 'Master' -- ${NORMAL_SOUND_LEVEL}

exit 0
8------------------------------------------------------------------------8

chmod +x /usr/local/bin/bluetooth-udev
```

#### Allow users to play sound
Only members of 'pulse-access' group can connect to pulse audio daemon (therefore playing sound).  
Allow root (udev will executes the script as root) to connect to pulseaudio
```
usermod -a -G pulse-access root
```

#### Add udev rule
Trigger the script via some udev rules
```
vim /etc/udev/rules.d/99-custom-bluetooth.rules
8------------------------------------------------------------------------8
# Triggered when bluetooth card is enabled
ACTION=="add", SUBSYSTEM=="bluetooth", KERNEL=="hci[0-9]*", RUN+="/usr/local/bin/bluetooth-udev card"

# Triggered when bluetooth card is disabled
ACTION=="remove", SUBSYSTEM=="bluetooth", KERNEL=="hci[0-9]*", RUN+="/usr/local/bin/bluetooth-udev card"

# Triggered when bluetooth device connects
ACTION=="add", SUBSYSTEM=="bluetooth", KERNEL=="hci0:[0-9]*", RUN+="/usr/local/bin/bluetooth-udev device"

# Triggered when bluetooth device DISconnects
ACTION=="remove", SUBSYSTEM=="bluetooth", KERNEL=="hci0:[0-9]*", RUN+="/usr/local/bin/bluetooth-udev device"
8------------------------------------------------------------------------8
```



# Result
At this point, you have:

1. Autonomous system: just power it up and you can connect to it via bluetooth
2. Audio is playing on raspi speakers with A2DP protocol
3. Control volume/songs/everything with bluetooth device
4. PI plays sample sounds when desired


# Details on project
This documentation aims people who got lost trying to make audio stuff with raspberry PI.  
A lot of internet docs about PI & audio are deprecated because of old PI or old Operating System.  
This one is to be used for Raspberry PI 3 + Raspbian Jessie.
The Idea is to make a Sonos like multiroom soundsystem.

## Why Bluetooth?
I tried several options and Bluetooth **really** is the best.  

1. It works with all kind of modern equipments (Android, iStuff, laptops, etc.)
2. It's stable for years
3. Already packaged for Linux with PulseAudio support

## Why PulseAudio?
1. Already packaged for Linux
2. It has **a lot** of modules to play with like bluetooth and RTP to stream audio over network


