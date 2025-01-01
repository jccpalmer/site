+++
date = '2025-01-01T10:40:15-05:00'
draft = false
title = 'Proxying Docker containers on a separate VLAN'
+++

I'd like to start this off by clearly stating that I'm not a network engineer, nor do I know how to be one. That said, however, a project idea came to me one day as I thought about breaking myself free from Cloudflare Tunnels. 

I built a weird setup with a Raspberry Pi, Nginx Proxy Manager, a separate VLAN, and a series of Docker containers that I wanted to access externally. I wasted hours, days, in pure frustration, only to find out that the solution was so simple.

The secret was the ports. What do I mean? Well, let me tell you about it.

## Coming up with the idea

Don't get me wrong, the service that Cloudflare offers homelab users for *free* is outstanding. Tunnels let me expose services like Nextcloud and Calibre Web without a hassle, but even those could prove troublesome given Cloudflare's terms of service. 

Such terms prevented the use of Tunnels for things like media, so publicly exposing my Jellyfin server was a no-go. Before Cloudflare, I had opened ports `80` and `443` on my router and used Nginx Proxy Manager (NPM) to proxy my services to a public URL. That is rather risky, though, and so I shut the whole thing down and went with Cloudflare.

So what was my idea? Run a proxy manager like NPM on a server that sits on a separate virtual LAN (VLAN) from my main network. That way, if something was compromised, I could minimize the damage done to my network.

## The setup

Alright, I've hinted at how I set this up, so let me explain it as best as I can. Frankly, I don't totally understand it all, as I built it a little bit at a time. So here's an amateur network diagram I drew up.

I have a main network where most of my client devices reside. A while ago, I built a separate network for my Internet of Things (IoT) devices, establishing them on a new VLAN with their own wireless network. The point of this was so that any Wi-Fi IoT devices in my smart home, such as Google speakers, my solar panel system controller, and robot vacuums, could be sectioned off from my main network as a security precaution.

This way, if one of those devices is compromised, I can keep the damage restricted to the IoT VLAN since I set up several firewall rules that keep the smart devices from getting more than the bare essential internet access that they need to function.

So I applied a similar philosophy to the proxy VLAN idea. I took an unused Raspberry Pi and flashed Raspberry Pi OS Lite to an SD card. (I really liked when it was called Raspbian, but I digress.) I installed Docker, Portainer, and Nginx Proxy Manager. 

I went into my router and built a new VLAN that I named NPM. The first order of business was to tell my router to put the Raspberry Pi on the new VLAN instead of the main one. Considering that I was using an unmanaged switch, I had to sacrifice one of the ports on my UniFi Dream Machine to dedicate to the proxy VLAN.

After this and forwarding ports `80` and `443` to the Pi, I started in on the firewall rules. In order to perform remote management from my computer or MacBook, I needed to allow the main LAN to talk to the VLAN. The next rule was to allow the Raspberry Pi to talk to the server that hosts my Docker containers. This is a Debian virtual machine running inside of Proxmox.

Finally, with that rule established, I told my router to drop everything else from the proxy VLAN. That means that the Raspberry Pi can now only talk to the Docker server and nothing else. The easiest way to test this is to attempt to ping a device on the main network while connected remotely to the proxy server.

I hope that makes a bit of sense. Basically, the Raspberry Pi can't talk to anything but itself and the server running my Docker containers so that it can proxy them, and that only through a series of manually opened ports on the Docker host's firewall.

## Where it went wrong

After loading everything up, I powered it all on and built my first proxy host. Then I tried to connect and... nothing. I got a connection failure. After some playing around, I tried to access the public URL outside of my network. Still nothing. I checked DNS settings and learned that my router's NAT options may be messing with the connection, but Ubiquiti, to my knowledge, doesn't allow you to change your router's NAT config.

I tried for hours and didn't make any progress. Several days later, I gave up. Then, while hanging out with a friend as she worked on her own project, I decided to spin this up again. I encountered the same errors. 

That said, we wouldn't be here right now if I hadn't figured it out.

## Success!.. and ports

After toying with my DNS resolver, Unbound — AKA switching to a public one — I suddenly got access to my external URLs while off my network. Huzzah! That was a major step in the right direction, which meant my NPM server was properly communicating with the Docker server. How? Hell if I know.

But internally, the URLs still resolved to NXDOMAIN in `nslookup`. Weird. 

On a whim, I installed UFW, or the Uncomplicated Firewall, on the Docker host and tried passing through the normal SSH, HTTP, and HTTPS ports. That didn't do anything. My frustration grew. I was so close!

Then, a lightbulb went off after a few more hours of searching. I went to the Docker host and manually allowed the port for my Jellyfin container, `8096`, through UFW. And right then, my Jellyfin public URL worked from inside my network.

I might have startled my friend in my joy. I frankly startled myself.

## What I learned

So, what did I learn? A lot, it turns out. From the top, I learned more about VLANs and firewall rules. The proxy VLAN was a more complicated setup than my IoT one, especially since it was meant to be as secure as I can manage with my layman knowledge.

Second, I learned a little bit more about DNS resolving, mostly because I still can't use Unbound on my Pi-Hole installation — I have [documentation](https://docs.jccpalmer.com/en/software/pi-hole) on how to do that, if you're interested — if I want to access the external URLs. While getting to the services locally via their internal IP is fine, it's not ideal for mobile apps like Nextcloud. When I leave my network, I would lose access unless I VPN into my home. (I'm still sorting out Wireguard.)

This I have yet to figure out. If I crack it, I'll let you know.

Third, I thought I knew about the importance of ports before, but boy do I understand them a lot better now. I'd worked with UFW in the past, but I never stopped to consider that it might be the solution, and simultaneously the hindrance, to my issue.

## Wrapping up

I can't believe that the solution this whole time, as it stands, was to use a public DNS resolver and to open each Docker container's port in UFW on the host server. The latter is a bit of nuisance after I create the proxy host in NPM, but it seems, to me at least, to be a more secure setup.

In the end, I could have also opted for the [Self-hosted Gateway](https://github.com/fractalnetworksco/selfhosted-gateway) solution, an open-source alternative to Cloudflare Tunnels. That, however, requires a virtual private server (VPS) to run properly since it needs ports `80` and `443`open to authenticate and get the SSL certificates. At that point, you might as well copy this project and save yourself the hassle and money. (Self-Hosted Gateway can be a pain to use and the documentation needs help the last time I checked; maybe I can lend a hand with that!)

I've had this going for a few days and I've watched logs on my router and Fail2ban on the NPM server. My router has blocked the generic `80` and `443` scans that bots and bad actors do, but nothing terrible has happened. Of course, nothing is foolproof and I will keep an eye on things. 

If you want to replicate this setup, I will have documentation for it ready to go soon, but you can check the [project page](/projects/nginx-proxy-manager-on-separate-vlan). Thanks for reading and happy proxying.