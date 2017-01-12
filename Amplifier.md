# HifiBerry AMP+
## Config
[You should look at this doc](https://github.com/project-owner/Peppy.doc/wiki/HiFiBerry%20Amp)  

Configuration for Raspbian Jessie:
````
vim /boot/config.txt
8------------------------------------------------------------------------8
 - dtparam=audio=on
 + #dtparam=audio=on
 +
 + # HifiBerry
 + dtoverlay=hifiberry-amp
8------------------------------------------------------------------------8

vim /etc/asound.conf
8------------------------------------------------------------------------8
pcm.!default  {
 type hw card 0
}
ctl.!default {
 type hw card 0
}
8------------------------------------------------------------------------8
````

## Details
I use Hifiberry AMP+ card to enjoy a really good sound.  
The integrated sound card is not so good but can be OK for a small extra sound device.

[From very good doc:](https://github.com/project-owner/Peppy.doc/wiki/Amplifier)
> [HiFiBerry Amp+](https://www.hifiberry.com/ampplus/) is the integrated amplifier module. That means that instead of using traditional components for a digital audio system: DAC - Pre-Amp - Power-Amp you can use just one - Amp+. It has Class-D amplifier on board which provides high quality powerful output: 25W on 4 Ohm speakers. As any other Class-D amplifier it's also very efficient meaning that there is no need to use heat sink or fan.

Everything is well explain in that doc. You should read it.
