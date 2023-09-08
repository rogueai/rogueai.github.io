+++
title = "Proton VPN on FritzBox routers"
date = 2023-09-08
description = "Use a FritzBox router as a Proton VPN client"
draft = false
toc = false
author = "rogueai"
categories = ["post"]
tags = ["fritzbox", "proton", "vpn"]
license = "cc-by-nc-sa-4.0"
+++

With FritzBox now supporting WireGuard from FirtzOS version 7.50 [^1], I wondered if it would be possible for it to be setup as Proton VPN client. 
Unfortunately, it seems the new feature is only intended to be used as a way to allow for external access into the local network. For this reason, the router requires either DynDns settings or FritzBox Account to be configure.

I had neither, and only wanted to tunnel all traffic through WireGuard/Proton VPN. I'm basing this guide on this Reddit post: https://www.reddit.com/r/fritzbox/comments/ymshoj/tutorialworkaround_how_to_use_fritzbox_wireguard/

## My setup
- FritzBox 4040
- Proton VPN, although it should probably work for any VPN service supporting WireGuard

## Steps

- From Proton VPN: create a download a new WireGuard configuration for routers
- Access FritzBox > Internet > Permit Access > DynDNS
  - Select "Use DynDNS"
  - Update URL: http://httpstat.us/200?user=&password=&host=&ip=&ip6=
  - Domain name: `localhost.local` (it doesn't need to be an existing domain name)
  - Username: anything
  - Password: anything
- Access FritzBox > Internet > Permis Access > VPN (WireGuard)
  - Select `Add Connection`
  - Select `Connect networks or establish special connections`, `Next`
  - Select `Yes` on `Has this WireGuard connection already been set up at the remote connection?`, `Next`
  - Provide a name for the connection
  - Upload the exported Proton VPN WireGuard
  - Select `Send all IPv4 network traffic via the VPN connection`
  - (Optional) Select `Only certain devices in the home network are to be accessible over this WireGuard® connection`
  - `Finish`
  
And we are done!

> Note that this won't allow any external access into the network. 
> The `httpstat.us` service can be self hosted for increased privacy.
  


[^1]: https://en.avm.de/service/vpn/connecting-the-fritzbox-to-a-vpn-provider-via-wireguard/
