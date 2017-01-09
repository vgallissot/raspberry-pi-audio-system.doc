**What**: doc to play sound on PI  
**How**: via Bluetooth + PulseAudio  
**Aim**: Synced Multi-room Audio receiver/issuer that works with as many equipment as possible  

This documentation aims people who got lost trying to make audio stuff with raspberry PI.  
A lot of internet docs about PI & audio are deprecated because of old PI or old Operating System.  
This one is to be used for Raspberry PI 3 + Raspbian Jessie.

This doc will often update to add new features and stuff.  
See changelog and roadmap.  

### Why Bluetooth?
I tried several options and Bluetooth **really** is the best.  

1. It works with all kind of modern equipments (Android, iStuff, laptops, etc.)
2. It's stable for years
3. Already packaged for Linux with PulseAudio support

### Why PulseAudio?
1. Already packaged for Linux
2. It has **a lot** of modules to play with like blluetooth and RTP to stream audio over network
