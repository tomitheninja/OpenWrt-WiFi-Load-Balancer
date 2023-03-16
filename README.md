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

## Performance evaluation

### Performance metrics

### Experimental results and analysis

### Results visualization with Grafana

## Conclusion and future work

### Summary of the thesis

### Contributions and limitations

### Future research directions

## References
