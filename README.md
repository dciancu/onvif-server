# Virtual Onvif Proxy
This is a simple Virtual Onvif Proxy that was originally developed to work around limitations in the third party support of Unifi Protect.
It takes an existing RTSP Stream and builds a virtual Onvif device for it, so the stream can be consumed by Onvif compatible clients.

Currently only Onvif Profile S (Live Streaming) is implemented with limited functionality.

## Credits
Thank you Daniela Hase for just randomly dropping this amazingly simple code to the public!
Original repository https://github.com/daniela-hase/onvif-server

It has truly inspired me and gave me so many ideas! 
I couldnt resist but to fork your original code so that I could add all these ideass

## Unifi Protect
Unifi Protect 5.0 introduced support for third party cameras that allow the user to add Onvif compatible cameras to their Unifi Protect system.

Unifi Protect seems to only support h264 video streams at the moment. So ensure your real camera encodes videos with h264 in normal or high profile. Do not use h264+

At the time of writing this, version 5.0.34 of Unifi Protect unfortunately has some limitations and does only support cameras with a single high- and low quality stream. Unfortunately video recorders that output multiple cameras (e.g. Hikvision / Dahua XVR) or cameras with multiple internal cameras are not properly supported.


Your Virtual Onvif Devices should now automatically show up for adoption in Unifi Protect as the name specified in the config. The username and password are the same as on the real Onvif device.
=======
# Roadmap
- Make it work in docker
  - Simplify the newtwork configuration
  - Add more explanations into stream detection
  - a few more things around docker in my head that will make things easier
- Create an Action pipeline
- Learn about the ONVIF Profile S
  - Implement snapshot functionality?
  - Implement some other features


# Docker
A prebuilt docker image can be found in the packages section of this repository.
All you need to do is to mount your `config.yaml` to `/onvif.yaml` inside the container.

Example of running the image in a temporary container:
```bash
docker run --rm -it -v /path/to/my/config.yaml:/onvif.yaml kulasolutions/rtsp-to-onvif:latest
```


## Wrapping an RTSP Stream
This tool is  used to create Onvif devices from regular RTSP streams by creating the following configuration

**RTSP Example:**
Assume you have this RTSP stream:
```txt
rtsp://192.168.1.32:554/Streaming/Channels/101/
       \__________/ \_/\______________________/
            |       Port    |
         Hostname           |
                          Path
```
If your RTSP url does not have a port it uses the default port 554.

Your RTSP url may contain a username and password - those should NOT be included in the config file.
Instead you will have to enter them in the software that you plan on consuming this Onvif camera in, for example during adoption in Unifi Protect.

Next you need to figure out the resolution and framerate for the stream. If you don't know them, you can use VLC to open the RTSP stream and check the _Media Information_ (Window -> Media Information) for the _"Video Resolution"_ and _"Frame rate"_ on the _"Codec Details"_ page, and the _"Stream bitrate"_ on the _"Statistics"_ page. The bitrate will fluctuate quite a bit most likely, so just pick a number that is close to it (e.g. 1024, 2048, 4096 ..).

Let's assume the resolution is 1920x1080 with 30 fps and a bitrate of 1024 kb/s, then the `config.yaml` for that stream would look as follows:

```yaml
onvif:
  - mac: a2:a2:a2:a2:a2:a1                        # The virtual MAC address for the server to run on
    ports:
      server: 8081                                # The port for the server to run on
      rtsp: 8554                                  # The port for the stream passthrough, leave this at 8554
    name: FrontDoor                               # A user define name that will show up in the consumer device
    uuid: 1714a629-ebe5-4bb8-a430-c18ffd8fa5f6    # A randomly chosen UUID (see below)
    highQuality:
      rtsp: /Streaming/Channels/101/              # The RTSP Path
      width: 1920                                 # The Video Width
      height: 1080                                # The Video Height
      framerate: 30                               # The Video Framerate/FPS
      bitrate: 1024                               # The Video Bitrate in kb/s
      quality: 4                                  # Quality, leave this as 4 for the high quality stream.
    target:
      hostname: 192.168.1.32                      # The Hostname of the RTSP stream
      ports:
        rtsp: 554                                 # The Port of the RTSP stream
```

You can either randomly change a few numbers of the UUID, or use a UUIDv4 generator[^3].

If you have a separate low-quality RTSP stream available, fill in the information for the `lowQuality` section above but this shows up as a seperate camera in unify. 

> [!NOTE]
> Since we don't provide a snapshot url you will onyl see the Onvif logo in certain places in Unifi Protect where it does not show the livestream.

# Troubleshooting

- **All cameras show the same video stream in Unifi Protect**

Unifi Protect identifies cameras by their MAC address - if multiple cameras have the same MAC address they will be treated as the same.
It is possible your system is configured for all virtual network interfaces to report the same MAC address, to prevent this run these commands[^4]:
```bash
sudo sysctl -w net.ipv4.conf.all.arp_ignore=1
sudo sysctl -w net.ipv4.conf.all.arp_announce=2
```

- **Error: Wsse authorized time check failed.**

Try updating the date/time on your Onvif device to the current time.

- **I only see snapshots, no live-stream.**

Are you capturing the RTSP streams of your cameras elsewhere already? It is possible that you hit the maximum concurrent RTSP streams that your camera supports.


## Virtual Networks
NB - Old instrucions that wont apply to docker version, Just keeping around as notes

To properly work with Unifi Protect each virtual Onvif device needs to have its own unique MAC address.
The easiest way to achieve this is by creating virtual network interfaces with the MacVLAN[^1] network driver:
```bash
ip link add [NAME] link [INTERFACE] address [MAC_ADDRESS] type macvlan mode bridge
```

> [!TIP]
> It is recommended to reserve fixed IP addresses in your DHCP server for your virtual networks.

Replace `[NAME]` with a name of your choosing (e.g. `onvif-proxy-1`) and `[MAC_ADDRESS]` with a locally administered MAC address[^2] (e.g. `a2:a2:a2:a2:a2:a1`) and `[INTERFACE]` with the name of the parent network interface (e.g. `eth0`).


## Example to create four virtual networks:
```bash
# Setup the first virtual network with name "onvif-proxy-1" and MAC address "a2:a2:a2:a2:a2:a1":
sudo ip link add onvif-proxy-1 link eth0 address a2:a2:a2:a2:a2:a1 type macvlan mode bridge

# Setup the first virtual network with name "onvif-proxy-2" and MAC address "a2:a2:a2:a2:a2:a2":
sudo ip link add onvif-proxy-2 link eth0 address a2:a2:a2:a2:a2:a2 type macvlan mode bridge

# Setup the first virtual network with name "onvif-proxy-3" and MAC address "a2:a2:a2:a2:a2:a3":
sudo ip link add onvif-proxy-3 link eth0 address a2:a2:a2:a2:a2:a3 type macvlan mode bridge

# Setup the first virtual network with name "onvif-proxy-4" and MAC address "a2:a2:a2:a2:a2:a4":
sudo ip link add onvif-proxy-4 link eth0 address a2:a2:a2:a2:a2:a4 type macvlan mode bridge
```

The above configuration creates a virtual Onvif device that listens on port 8081 of the `a2:a2:a2:a2:a2:a1` virtual network and forwards the RTSP video streams and snapshots from `192.168.1.152` (the real Onvif server).







[^1]: [What is MacVLAN?](https://ipwithease.com/what-is-macvlan)
[^2]: [Wikipedia: Locally Administered MAC Address](https://en.wikipedia.org/wiki/MAC_address#:~:text=Locally%20administered%20addresses%20are%20distinguished,how%20the%20address%20is%20administered.)
[^3]: [UUIDv4 Generator](https://www.uuidgenerator.net/)
[^4]: [Virtual Interfaces with different MAC addresses](https://serverfault.com/questions/682311/virtual-interfaces-with-different-mac-addresses)
