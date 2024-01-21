# Measurement setup for the examination projects

## TOC

- [Objectives/Tasks](#objectivestasks)
- [Setup overview](#setup-overview)
- [Network architecture](#network-architecture)
    - [Access to your device](#access-to-your-device)
    - [Network namespaces setup](#network-namespaces-setup)
- [Using the setup](#using-the-setup)
    - [How to use iperf3 in this setup](#how-to-use-iperf3-in-this-setup)
- [How to switch between forwarding techniques](#how-to-switch-between-forwarding-techniques)
    - [IP forwarding](#ip-forwarding)
    - [eBPF](#ebpf)

## Objectives/Tasks

Goal of the project is to measure and compare the performance of several forwarding techniques on different devices.

Compare the following four forwarding techniques/implementions regarding their performance:
- eBPF (TC or XDP)
- IP forwarding
- IP forwarding with software offloading
- IP forwarding with hardware offloading

> There are two eBPF implementations (TC or XDP) that we will use for comparison, however not every team uses both implementations.
See the assignment table below to see which team uses which of them.

Use the below explained setup to fulfil the task. Use iperf3 for traffic generation and reception, and tcpdump to collect data during your measurement. For analysis and evaluation, you can use e.g. Python or R with their provided data analysis and plotting facilities.

## Setup overview


                                        Muxer
                                         or
                                one-to-rule-them-all
                                     ___________
                                    |           |
                                    |           |
                      ______________|           |<--------- ssh access via 195.37.88.180 port 58400 with user teamX
                     |              |           |
                     |              |           |
                     |              |___________|
                     |
                     |
                 ____|______________________________________________
                |  __|__    __   __   __   __   __   __   __   __   |
                | | 10G |  |__| |__| |__| |__| |__| |__| |__| |__|  |
                | |_____|    |    |    |    |    |    |    |    |   |
                |------------|----|----|----|----|----|----|----|---|
                             |    |    |    |    |    |    |    |
                             |    |    |    |    |    |    |    |
             ________________|    |    |    |    |    |    |    |________________
            |    _________________|    |    |    |    |    |_________________    |
            |   |                ______|    |    |    |______                |   |
        ____|___|____           |   ________|    |________   |           ____|___|____
       |   Banana    |      ____|__|____              ____|__|____      |    Xiaomi   |
       |     Pi      |     |   FRITZ    |            |            |     |    MiR 4A   |
       |_____________|     |    BOX     |            |   TP-LINK  |     |_____________|
              1            |____________|            |____________|            4
                                 2                         3

The device `Muxer`/`one-to-rule-them-all` is a regular x86_64-based Desktop-PC acting as controller for the whole setup and also as a point of access for all of you. The controller has a 10-Gigabit connection to the switch.

We will use 4 different devices for the project:
- device 1: [Sinovoip Banana Pi R64](https://openwrt.org/toh/sinovoip/bananapi_bpi-r64_v1.1)
- device 2: [AVM FritzBox 7360 V2](https://openwrt.org/toh/avm/fritz.box.wlan.7360)
- device 3: [TP-Link WDR4900 v1](https://openwrt.org/toh/tp-link/tl-wdr4900)
- device 4: [Xiaomi Mi Router 4A Gigabit Edition (V1)](https://openwrt.org/inbox/toh/xiaomi/xiaomi_mi_router_4a_gigabit_edition)

These devices are different regarding their CPU architecture, their network controllers, their performance, etc. See the links to get further information about those devices and their specs.

Hereby, devices and task variants are assigned to the teams as follows:
| Team | Device  | eBPF variant |
|:---- |:------- |:------------- |
| Team 1 | Device 1 (Banana Pi R64) | TC |
| Team 2 | Device 1 (Banana Pi R64) | XDP |
| Team 3 | Device 2 (FritzBox 7360) | TC |
| Team 4 | Device 2 (FritzBox 7360) | XDP |
| Team 5 | Device 3 (TP-Link WDR4900) | TC |
| Team 6 | Device 3 (TP-Link WDR4900) | XDP |
| Team 7 | Device 4 (Xiaomi MiR 4A) | TC |
| Team 8 | Device 4 (Xiaomi MiR 4A) | XDP |

> You should see that we have 8 teams but only four devices. This means, two teams need to share one device setup. Organize yourself to ensure all teams can do proper experiments!

## Network architecture

Each of these devices is connected with two cables to the same switch to which the controller is also connected, thus they are in a network.

                                                           ________________________
     ___________________________                          |  Port X   Port X+1       switch
    |                      __   |                         |   __        __
    |    device     lan 1 |__|-------------------------------|__|      |__|
    |                      __   |                         |              |         ....
    |               lan 2 |__|--------------------------------------------
    |___________________________|                         |________________________


To separate the devices and their networks from each other, they are grouped into [VLANs](https://en.wikipedia.org/wiki/VLAN).
- device 1, Switch ports 1 + 2: VLAN 21
- device 2, Switch ports 3 + 4: VLAN 22
- device 3, Switch ports 5 + 6: VLAN 23
- device 4, Switch ports 7 + 8: VLAN 24

The controller has access to all of these VLANs, and thus has a connection to all of the devices (using 4 virtual interfaces).

Analoguous to the device and VLAN numbering, each "virtual network" between the device and the controller has its own IP space according to the following list. This list also shows which IP addresses are assigned to the devices and the controller's virtual interfaces:
- device 1/VLAN 21: `10.21.1.0/24`
    - IP of controller (on `ens1.21`):      `10.21.1.1`
    - IP of device 1 (on `lan1`):           `10.21.1.2`
- device 2/VLAN 22: `10.22.1.0/24`
    - IP of controller (on `ens1.22`):      `10.22.1.1`
    - IP of device 2 (on `lan1`):           `10.22.1.2`
- device 3/VLAN 23: `10.23.1.0/24`
    - IP of controller (on `ens1.23`):      `10.23.1.1`
    - IP of device 3 (on `lan1`):           `10.23.1.2`
- device 4/VLAN 24: `10.24.1.0/24`
    - IP of controller (on `ens1.24`):      `10.24.1.1`
    - IP of device 4 (on `lan1`):           `10.24.1.2`

> Rule: the switch port with the odd number always carries the connection over which you can access the device.

---
### Access to your device

In order to switch between forwarding techniques, which is explained later, you need to access your device. Each team is only allowed to access the device assigned to them.
Each team has an SSH key in `~/.ssh/private/device.key` on the controller which can only be used on the assigned device. You can access the device by manually specifying the IP (see list above), user (root) and the key file.
However, we already created an SSH config entry (in `~/.ssh/config`) for convenient access.

TLDR: To access you device, just log in to the controller with your team account and type:
```bash
ssh device
```

---
### Network namespaces setup

As we discussed in the lecture, it is essential to have a well-designed setup for your experiments that allows for valid and reproducible measurements.
One important part of such a good design - especially for network performance experiments - is to ensure your device and its operation is not influenced by any undesired and avoidable side-effects.

An important part of our experiment is the use of artificial traffic to saturate the network links and measure/judge their performance.
Looking at the devices we use in this experiment and estimating how "powerful" they are, it should be obvious that performing the traffic generation on the device itself would introduce such aforementioned side-effects and thus distort our measurement.
The reason for this is quite simple, traffic generation is a heavy task which causes a high system load. In combination with the system load that is caused by the different forwarding implementations we want to use,
the device will reach it's overall performance limits quite fast, resulting in decreased forwarded network traffic and thus distorted measurement results.

To circumvent this, we outsource traffic generation and reception to a much more powerful device(s), in our case only one powerful device taking care of both. This is the controller mentioned in the beginning.

However, there is something special about the networking setup on the chosen controller. The initial thought could be that the controller has two ethernet interfaces, you just connect both to the device that you want to forward traffic over,
and you just start the traffic generation on one of the interfaces and receive the traffic on the other one. If you do that, you will likely see an extremely high throughput which is impossible for the device to handle over usual network connections.
The reason for that is, that the operating system obviously detects that traffic source and sink are the same device and just forwards the traffic using the controllers own forwarding backplane, not using the link(s) to the device.

The Linux kernel, which we are using here, has a feature called `namespaces`, which can be used to exactly address this issue.   
The general idea of `namespaces` is to partition the available kernel resources for different sets of processes (`namespaces` are used e.g. for containers). To learn more about `namespaces`, see the wiki page [here](https://en.wikipedia.org/wiki/Linux_namespaces).   
In a nutshell, `namespaces` allow us to create multiple network interfaces on our controller which have no idea of each other, also allowing us to generate and receive traffic on the same device.

If we leave out the switch in our setup for a moment, this essentially boils down to the following connections between the controller and the device:

                                                               ___________________________________________
                                                              |                         Namespace 1       |
                                                              |                       _______________     |
                                                              |                      |    _______    |    |
         ___________________________                          |    ________          |   |  if0  |   |    |
        |                      __   |                         |   |        |         |   |_______|   |    |
        |    device     lan 1 |__|--------------------------------|  phys. |---------|_______________|    |
        |                      __   |                         |   | inter- |          _______________     |
        |               lan 2 |__|--------------------------------|  face  |---------|    _______    |    |
        |___________________________|                         |   |________|         |   |  if1  |   |    |
                                                              |                      |   |_______|   |    |
                                                              |                      |_______________|    |
                                                              |                         Namespace 2       |
                                                              |___________________________________________|

According to the schematic, the interfaces on the controller still transmit data over the common physical link. But since the namespaces completely isolate the interfaces from each other,
we are able to send out traffic on `if0` and receive it on `if1` over the real link, not the forwarding backplane of the controller.

> TLDR: For each device there are 2 namespaces (corresponding to the two cable connections between the device and the switch) which you will use for traffic generation and reception.

The namespaces are generated during startup of the controller, resulting in:
- device 1:
    - `itsys-d1-source`
    - `itsys-d1-sink`
- device 2:
    - `itsys-d2-source`
    - `itsys-d2-sink`
- device 3:
    - `itsys-d3-source`
    - `itsys-d3-sink`
- device 4:
    - `itsys-d4-source`
    - `itsys-d4-sink`

> This setup is not part of what you have to do in the exam, it is provided and maintained by us. This means you should not change anything of it!

## Using the setup

Now that you know how the setup in general looks like and how we achieve what we want using `namespaces`, it's time for you to know how you can use it. For that you basically need to know which IP addresses to use when running `iperf3` as client and server.   
So let's look at the connections between one of our devices and the controller on the Layer-3 viewpoint, the IP layer:

                                            source subnet
         ______________________                                           ___________________________
        |                      | 10.XX.10.2                   10.XX.10.1 |                           |
        |   device      lan1  <> ======================================= <> ul-source   controller   |
        |         ~~~~~~~~~~~~~|~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~             |
        |               lan2  <> ======================================= <> ul-sink                  |
        |______________________| 10.XX.20.2                   10.XX.20.1 |___________________________|

                                             sink subnet

> The namespaces setup that we provide also includes that IP routes are properly set up. This means you should be able out-of-the-box to reach `10.XX.20.1` from `ul-source`
and `10.XX.10.1` from `ul-sink`.

The two connections between your device and the controller use two different subnets for better distinction. They are called the `source subnet` and the `sink subnet`, within each there is an own IP address space according to the following list:
- device 1:
    - source subnet:        `10.21.10.0/24`
    - sink subnet:          `10.21.20.0/24`
- device 2:
    - source subnet:        `10.22.10.0/24`
    - sink subnet:          `10.22.20.0/24`
- device 3:
    - source subnet:        `10.23.10.0/24`
    - sink subnet:          `10.23.20.0/24`
- device 4:
    - source subnet:        `10.24.10.0/24`
    - sink subnet:          `10.24.20.0/24`

Using that table and the schematic above you can see which IP addresses you need to use on the controller for your team setup. For example, team 1 would need to use `10.21.10.1` as their traffic source IP and `10.21.20.1` as their traffic sink IP.

---
### How to use iperf3 in this setup

We cannot completely avoid that you somehow have to deal with the namespaces here. Running plain commands like `ping 10.21.10.2` won't work here.
The reason for this is, that Linux just tries to use the default namespace if you do not specify a different namespace to use.
Since the `source` and `sink` subnets do not live within the default namespaces but in our separate namespaces, we always need to tell Linux that it has to use a specific namespace for the command we wan't to execute.

The general approach for this is using the `ip` command (btw. the `ip` is also used to set up namespaces) with its dedicated subcommand 'netns':
```bash
ip netns exec <namespace> <command>
```
To execute a command in a specific namespace you use the syntax above, telling the `ip` command which namespace to use followed by your command.

The usage of `iperf3` is in general quite easy. `iperf3` uses a client-server-model and by default the client generates and transmits data to the server, corresponding to traffic source and sink respectively.
To learn more about `iperf3`, try to run `man iperf3` in the command line or have a look at the online version of its manpage [here](https://manpages.org/iperf3).

1. Start `iperf3` server in your sink namespace + subnet:
```bash
ip netns exec itsys-dX-sink iperf3 -s
```
> Without specifying further options, `iperf3` will listen for incoming connections on all interfaces, including `ul-sink` with its IP `10.XX.20.1`

2. Start `iperf3` client in you source namespace + subnet:
```bash
ip netns exec itsys-dX-source iperf3 -c 10.YY.20.1
```

You should see that the client connects to the server, transmits data and both client and server should print statistics each second about the achieved throughput.

---
## How to switch between forwarding techniques

As already mentioned, we are using multiple forwarding techniques that we want to compare. To do that, you need to switch between them for your experiments. Here's how to do that.

The eBPF implementations need to be enabled/disabled using two scripts the we provide to you. On each device, you can find the scripts `tc` and `xdp` in the home directory of user root (`/root`).

The other techniques are enabled/disabled via the firewall configuration.

---
### IP forwarding

At first, open the firewall configuration file `/etc/config/firewall` with an editor (e.g. `vim`).
Then look for a config block called `defaults` which should be similar to:
```
config defaults
    option input 'REJECT'
    option output 'ACCEPT'
    option forward 'REJECT'
    option flow_offloading '0'
    option flow_offloading_hw '0'
```

| Goal | What to do |
|:---- |:---------- |
|enable 'IP forwarding with software offloading' | change the value of `flow_offloading` to `1`. |
|disable 'IP forwarding with software offloading' | change the value of `flow_offloading` to `0`. |
|enable 'IP forwarding with hardware offloading' | change the value of `flow_offloading_hw` to `1`. |
|disable 'IP forwarding with hardware offloading' | change the value of `flow_offloading_hw` to `0`. |
|use 'IP forwarding without offloading' | change both options `flow_offloading` and `flow_offloading_hw` to `0`. |

After making the changes, save the file, exit the editor and run `/etc/init.d/firewall restart` to restart the firewall and apply the changes.

---
### eBPF

Make sure that the above-mentioned options `flow_offloading` and `flow_offloading_hw` are set to `0` when using eBPF and do a `/etc/init.d/firewall restart` to ensure that they are applied.

Before you start your measurement, ensure with `bpftool prog show` that no eBPF program is currently loaded. If you see an output similar to the following:
```
# bpftool prog show
44: xdp  name router_map  tag 34007055141dda23  gpl
	loaded_at 2024-01-21T17:42:34+0000  uid 0
	xlated 1360B  jited 1584B  memlock 4096B  map_ids 11
	btf_id 66
48: xdp  name router_map  tag 34007055141dda23  gpl
	loaded_at 2024-01-21T17:42:34+0000  uid 0
	xlated 1360B  jited 1584B  memlock 4096B  map_ids 12
	btf_id 72
```
... then there is still an eBPF program loaded, which you have to turn off/unload first. If there is no output, then no eBPF program is loaded, so you are good to go.

To turn **on** eBPF-TC, run `/root/tc load`.   
To turn **off** eBPF-TC, run `/root/tc unload`.

To turn **on** eBPF-XDP, run `/root/xdp load`.   
To turn **off** eBPF-XDP, run `/root/xdp unload`.

Make sure to unload the eBPF program after you finish your measurement so that you don't ruin the experiment of the other team.

These scripts will use the `*.o` compiled binaries which are in the same folder and contain the eBPF program. If you are interested in how the programs work, have a look [here](https://github.com/tk154/eBPF-Tests/blob/main/programs/kernel/router_map.c).
