+++
date = "2023-08-28T09:22:37+09:00"
draft = false 
title = "How to simulate Restricted Cone NAT locally"
comments = true	# set false to hide Disqus
share = true	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = ""
+++

Reproducing specific network topology is important in network protocol development. If a specific network topology can be reproduced on the developer's local machine, development efficiency can be greatly improved through test automation and other means. This post describes how to reproduce Restricted Cone NAT network on your local machine.

<!--more-->

Edit: I've published a Docker-based one - https://github.com/ackintosh/discv5-hole-punching. I hope it helps you.

## Network topology

The diagram below shows network topology that this post describes. There are three segments: `10.0.0.0/8`, `192.168.1.0/24` and `192.168.2.0/24`. In this example, we assume `10.0.0.0/8` is public network, and `192.168.1.0/24` and `192.168.2.0/24` are private network that behind NAT.

![network topology](https://github.com/ackintosh/udp-hole-punching/assets/1885716/3acf9460-b4ec-4eca-be6d-21f6d858e550)

## Settings for router

In this example, the router (`nat1` and `nat2` in the diagram) is not actual router, but a Linux machine/container that configured iptables to simulate behaviours of router. `nat1` and `nat2` needs to be configured iptables as bellow:

> iptables -t nat -A POSTROUTING -o `<public interface>` -p udp -j SNAT --to-source `<public ip>`

This enables source IP address translation to `<public ip>` for outbound packets sent from `<public interface>`. By this setting, source IP address of a packet that sent from a host  behind NAT to external network is translated to the `<public ip>` address.

> iptables -t nat -A PREROUTING -i `<public interface>` -p udp -j DNAT --to-destination `<private host ip>`

This enables destination IP address translation to `<private host ip>` for inbound packets to `<public interface>`. By this setting, a packet sent from external network can reach a host in private network via the router.

> iptables -A FORWARD -i `<public interface>` -p udp -m state --state ESTABLISHED,RELATED -j ACCEPT

This setting pertains to packet forwarding. iptables can be configured to determine its behavior for forwarding based on the state of  a session. Of course, UDP doesn't have sessions. The session mentioned here refers to iptables's own functionality. By this setting, iptables forwards inbound packets from `<public interface>` if the state of session is ESTABLISHED or RELATED.

> iptables -A FORWARD -i `<public interface>` -p udp -m state --state NEW -j DROP

iptables drops inbound packets from `<public interface>` if the state of session is NEW.

These settings above enable the important characteristics of Restricted Cone NAT. All requests from the same internal IP address and port are mapped to the same external IP address and port. Furthermore, packets from external host that were not initiated by the internal host are dropped.

## Mininet

Building the network topology needs various configurations in addition to the iptables settings. These can be scripted using [Mininet](https://mininet.org/). I've published the script to build the network topology described in this post:

[ackintosh/udp-hole-punching: This repository provides a simulated Restricted Cone NAT environment.](https://github.com/ackintosh/udp-hole-punching)

This post does not explain how to use Mininet. Please refer to [Get Started](https://mininet.org/download/) and [network.py](https://github.com/ackintosh/udp-hole-punching/blob/main/network.py) for details.

## UDP Hole Punching

The repository includes Python scripts that run UDP Hole Punching. You can check how the NAT hosts behave with the scripts. Here is a sequence diagram that illustrates how UDP Hole Punching works with the scripts. See [README](https://github.com/ackintosh/udp-hole-punching/blob/main/README.md) for how to run the scripts.

![UDP Hole Punching](https://github.com/ackintosh/udp-hole-punching/assets/1885716/307d9944-5885-44c9-a51e-84e72449b46c)

