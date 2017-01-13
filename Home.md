## TL;DR
**What**: doc to play sound on PI  
**How**: via Bluetooth + PulseAudio  
**Aim**: Synced Multi-room Audio receiver/issuer that works with as many equipment as possible  

## Versions
This doc will often update to add new features and stuff.  
See 
* [CHANGELOG](https://github.com/vgallissot/raspberry-pi-audio-system.doc/blob/master/CHANGELOG.md)
* [ROADMAP](https://github.com/vgallissot/raspberry-pi-audio-system.doc/blob/master/ROADMAP.md)


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
````
vim /lib/udev/rules.d/75-persistent-net-generator.rules
8------------------------------------------------------------------------8
- KERNEL!="ath*|msh*|ra*|sta*|ctc*|lcs*|hsi*", \
+ KERNEL!="ath*|wlan*[0-9]|msh*|ra*|sta*|ctc*|lcs*|hsi*", \
8------------------------------------------------------------------------8
````
``/etc/udev/rules/70-persistent-net.rules`` file will be created after reboot containing fixed wlan names according to MAC addresses.  


### Disable default wireless and enable the dongle
````
modprobe -r -v brcmfmac
vim /etc/modprobe.d/raspi-blacklist.conf
8------------------------------------------------------------------------8
#wifi
blacklist brcmfmac
#blacklist brcmutil
8------------------------------------------------------------------------8
````


````
vim /etc/network/interfaces
8------------------------------------------------------------------------8
iface wlan0 inet manual

auto wlan1
allow-hotplug wlan1
iface wlan1 inet manual
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
8------------------------------------------------------------------------8
````
Edit wpa_supplicant.conf according to your WiFi specs.  

Reboot and be ready
````
shutdown -r now
````

### Test sound
scp some mp3 file to the PI and play it: 

````
apt-get update
apt-get install mplayer -y
amixer sset 'Master'  -- 70%
mplayer 02.\ I\ Fink\ U\ Freeky.mp3
````

=> Sound should be OK.  
``aplay -l`` should output only 1 card.


### bluetooth support
Install bluez + dependency to pulseaudio
````
apt-get install -y pulseaudio-module-bluetooth bluez-tools
````

Make the Bluetooth up at startup
````
vim /etc/systemd/system/bluetooth.target.wants/bluetooth.service
8------------------------------------------------------------------------8
    - ExecStart=/usr/lib/bluetooth/bluetoothd
    + ExecStart=/usr/lib/bluetooth/bluetoothd --noplugin=sap
    + ExecStartPost=/bin/hciconfig hci0 up
8------------------------------------------------------------------------8
````

Configure class of BT. It defines what capabilities your BT will have (audio only / audio + micro / etc.).
````
vim /etc/bluetooth/main.conf
8------------------------------------------------------------------------8
# Changing name here has no effect
Class = 0x0c0420
8------------------------------------------------------------------------8
````

### Configure PulseAudio
We need the 'pulse' user to access bluetooth via dbus
````
vim /etc/dbus-1/system.d/pulseaudio-bluetooth.conf
8------------------------------------------------------------------------8
<busconfig>

  <policy user="pulse">
    <allow send_destination="org.bluez"/> 
  </policy>

</busconfig>
8------------------------------------------------------------------------8
````

Restart dbus with ``systemctl restart dbus``


PulseAudio on a desktop is not started as a deamon ([Check out why](https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/WhatIsWrongWithSystemWide/)). For a standalone sound system, we need it to be started as a system-wide daemon.
````
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
````

Load bluetooth modules for pulseaudio
````
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
````

As the PI is used as a dedicated device for audio : No Restrictions on PI's consumptions = Best Audio quality possible (improvements are welcome)
````
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
````

### Custom Bluetooth name
````
echo 'PRETTY_HOSTNAME=$myawesomename' >> /etc/machine-info
service bluetooth restart
````

### Manual pairing
example from [archlinux doc](https://wiki.archlinux.org/index.php/bluetooth#Bluetoothctl)
````
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
````

### Start and stop sounds
Play some sounds when PI starts and when it shuts down.  

scp some **short** samples to PI.  
I used ``/usr/local/share/sounds/`` dir.  
I force a low volume when stopping PulseAudio to avoid unwanted crazy notifications.  

````
vim /etc/systemd/system/pulseaudio.service
8------------------------------------------------------------------------8
+ ExecStartPre=-/usr/bin/mplayer --really-quiet /usr/local/share/sounds/portal/Hello.ogg
+ ExecStartPre=-/usr/bin/amixer sset 'Master'  -- 80%%
  ExecStart=/usr/bin/pulseaudio --system --disallow-exit --disable-shm
+ ExecStopPost=-/usr/bin/amixer sset 'Master'  -- 55%%
+ ExecStopPost=-/usr/bin/mplayer --really-quiet /usr/local/share/sounds/portal/Goodbye.ogg
8------------------------------------------------------------------------8
systemctl daemon-reload
````


# Result
At this point, you have:

1. Autonomous system: just power it up and you can connect to it via bluetooth
2. Audio is playing on raspi speakers with A2DP protocol
3. Control volume/songs/everything with bluetooth device
4. PI plays sample sounds when it start and when it shuts down


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


