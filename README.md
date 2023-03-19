# OpenWrt WiFi Load Balancer

## Introduction

### The aim of this project

### Overview of Wi-Fi and 802. protocols

### OpenWrt overview

## Literature review

### Load balancing techniques

### Assisted roaming in wireless networks

### Related works on load balancing and assisted roaming on OpenWrt

## Load balancing and assisted roaming on OpenWrt

### Load balancing and assisted roaming architecture

### Network topology

### Algorithms and policies

#### My bucket based algorithm

#### Rate limiting

### Implementation details

#### Client selection algorithms

## Client-assisted load balancing

### overview

### Client behavior and metrics

## Testing environment

### Description of the testing environment

#### Installing OpenWrt on an access point

I attempted to install OpenWrt on a Linksys E3000 access point. Unfortunately this failed because the device only had 3 MB of flash memory. Even with a custom build, I could not reduce the image size to fit in the flash memory, while still keeping the hostapd module. I decided not to try network booting.

![install-failed](./images/openwrt-install-failed.png)

### Setting up OpenWrt in a virtual machine

The testing environment for the project was a virtual machine.
I have chosen virt-manager for this purpose, because it is easy to use and supports many virtualization technologies.
This approach later even offered several benefits, such as the ability to easily capture and restore snapshots.
The host system is a MSI GF65 laptop running the Linux 6.1 kernel, with an Intel(R) Wi-Fi 6 AX201 160MH wireless network adapter and a network manager component integrated into the CPU.

There are official OpenWrt images for x86_64. I used the this one: [openwrt-22.03.2-x86-64-generic-ext4-combined](https://downloads.openwrt.org/releases/22.03.2/targets/x86/64/openwrt-22.03.2-x86-64-generic-ext4-combined.img.gz)

### Network configuration for the virtual NIC

When I booted up OpenWrt, I could not connect to the internet. I tried to ping a website and an ip address, but both times I received an error message indicating that the network was unreachable.

```console
root@OpenWrt:/# ping vanenet.hu
ping: bad address 'vanenet.hu'
root@OpenWrt:/# ping 1.1.1.1
PING 1.1.1.1 (1.1.1.1): 56 data bytes
ping: sendto: Network unreachable
```

I decided to check the network configuration. I opened the file '/etc/config/network' and saw that the interface is configured to use a static ip address. I changed the protocol to dhcp and restarted the service with the `/etc/init.d/network restart` command.

```diff
config interface 'lan'
    option device 'br-lan'
+    option proto 'dhcp'
-    option proto 'static'
-    option ipaddr '192.168.1.1'
-    option netmask '255.255.255.0'
-    option i6assign '60'
```

I tried again, and this time I managed to connect to the server.

```console
root@OpenWrt:/# ping vanenet.hu
PING vanenet.hu (185.33.54.12): 56 data bytes
64 bytes from 185.33.54.12: seq=0 ttl=54 time=8.351 ms
64 bytes from 185.33.64.12: seq=1 ttl=54 time=8.104 ms
64 bytes from 185.33.64.12: seq=2 ttl=54 time=7.570 ms
```

### Setting up the wireless network

In order to test the wireless networking features of OpenWrt, it was necessary to have low level access to the wireless network adapter.
I used PCI pass-through to achieve this.

#### PCI passthrough of the wireless network adapter

To simplify the process I used the `gpu-passthrough-manager` tool.
I created a new grub boot entry with the kernel parameter `vfio-pci.ids=8086:06f` to enable passthrough of the wireless network controller. I then rebooted the host and added the network adapter to the virtual machine using the virt-manager GUI.

#### Installing the wireless driver

I installed the `kmod-iwlwifi` package using the opkg package manager and loaded the driver with the `modprobe` command.

However, after rebooting the VM, I didn't see the interface.

I checked the logs using 'dmesg' and found the following lines:

```log
[ 4.953303] iwlwifi 0000:07:00.0: Direct firmware load for iwlwifi-QuZ-a0-hr-b0-39.ucode failed with error -2
[ 4.957677] iwlwifi 0000:07:00.0: minimum version required: iwlwifi-QuZ-a0-hr-b0-39
[ 4.958681] iwlwifi 0000:07:00.0: maximum version supported: iwlwifi-QuZ-a0-hr-b0-66
```

To solve the problem, I copied the iwlwifi-QuZ-a0-hr-b0-66.ucode firmware file from the host machine and placed it in the /lib/firmware folder.

After rebooting the VM, I could see the new interface.

#### Issues with the Intel wireless network adapter

Newer Intel wireless network adapters do not support AP mode on the 5 GHz band.
This is due to LAR (location aware regulatory) restrictions, which require manufacturers to authorize adapters because the frequency range is also used by other wireless technologies.

Additionally, the speed was unstable and asymmetric on the 2.4 GHz band.

TODO: jitter test

#### Switching to an USB wireless network adapter

I removed the Intel adapter from the VM along it's configuration in the `/etc/config/wireless` file and connected a TP-Link TL-WN722N v1 USB wireless network adapter.

The two packages I needed to install to get it working were `kmod-ath9k-htc` and `kmod-rtl8192cu` as described in the [OpenWrt wiki](https://forum.openwrt.org/t/add-tp-link-tl-wn722n-wifi-usb-adaptor/24253). Furthermore, I had to restart the host machine.

TODO: speedtest

#### Configuring the wireless network

I used the `ip a` command to check the ip address of the network interfaces.
I then opened the luci web interface on the given ip address and logged in with the root user.

I saw the network adapter and a wireless network called OpenWrt.
By default, the network was disabled and the encryption open network, because the hostapd package was not installed.

I installed the hostapd package and restarted the VM. This time I was able to enable enable security and started the network.

## Performance evaluation

### Performance metrics

#### Installing prometheus and grafana

Prometheus is an open-source systems monitoring and alerting toolkit [x](https://prometheus.io/docs/introduction/overview/). Grafana is an open-source dashboard and data visualization tool [x](https://grafana.com/grafana/). It can be configured to use Prometheus as a data source. It is easy to use, well documented and has a large community. [x](https://grafana.com/blog/2021/02/09/how-i-monitor-my-openwrt-router-with-grafana-cloud-and-prometheus/)

The recommended packages can be installed using this command

```bash
opkg install prometheus-node-exporter-lua \
prometheus-node-exporter-lua-nat_traffic \
prometheus-node-exporter-lua-netstat \
prometheus-node-exporter-lua-openwrt \
prometheus-node-exporter-lua-wifi \
prometheus-node-exporter-lua-wifi_stations
```

By default it is only accessible from localhost. To test it, I used the curl command.

```console
root@OpenWrt:~# curl localhost:9100/metrics
...
wifi_station_signal_dbm{mac="4E:C1:16:8C:AC:01",ifname="wlan0"} -39
wifi_station_inactive_milliseconds{mac="4E:C1:16:8C:AC:01",ifname="wlan0"} 4150
wifi_station_transmit_kilobits_per_second{mac="4E:C1:16:8C:AC:01",ifname="wlan0"} 72200
wifi_station_receive_kilobits_per_second{mac="4E:C1:16:8C:AC:01",ifname="wlan0"} 6000
wifi_station_transmit_packets_total{mac="4E:C1:16:8C:AC:01",ifname="wlan0"} 103446
wifi_station_receive_packets_total{mac="4E:C1:16:8C:AC:01",ifname="wlan0"} 187999
wifi_stations{ifname="wlan0"} 2
node_scrape_collector_duration_seconds{collector="wifi_stations"} 0.00032901763916016
node_scrape_collector_success{collector="wifi_stations"} 1
```

As you can see, my phone was connected to the wireless network when the command was executed and was capable of transmitting with roughly 72 Mbps and receiving with 6 Mbps.

Note that this snippet is only a small part of the output.

### Experimental results and analysis

### Results visualization with Grafana

## Conclusion and future work

### Summary of the thesis

### Contributions and limitations

### Future research directions

## References

```

```
