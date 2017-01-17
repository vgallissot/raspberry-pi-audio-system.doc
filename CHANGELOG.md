# ChangeLog

## 2.0 (2017-01-17)
Enable notifications

### Enhancements

- Use udev rule to trigger actions
- Play notifications at startup / shutdown / device connect / disconnect

### Bug fixes

- Bad sample rates used with hifiberry card
- Random wlan names
- Raspberry wlan card still active


## 1.0 (2017-01-12)
First release.

### Enhancements

- Start PulseAudio as a daemon
- Start Bluetooth at startup
- Load Bluetooth modules from PulseAudio
- Disable wlan0 card that interracts with Bluetooth [Issue 1](https://github.com/raspberrypi/linux/issues/1342) + [Issue 2](https://github.com/raspberrypi/linux/issues/1402)
- Manual pairing via command line.
- Not that bad audio quality config in Pulseaudio

