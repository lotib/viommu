# check sysfs

find /sys/kernel/iommu_groups/

# check kernel parameter

grep iommu /proc/cmdline
grep Hug /proc/meminfo


## grub

grep iommu /et/default/grub

GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt hugepages=1024 hugepagesz=2m "
update-grub


# Identify Network Controller

``` shell
lscpi | grep Ether
lspci -n | grep "0200:"

OVSDEV_PCIID="00:1f.6"


```


# configure dpdk

## assign linux VFIO drivers

``` shell
modprobe vfio-pci

# unbind network driver, you should disable your interface : 
# ip link set down dev enp2s0
# ip link set down dev enp0s31f6
dpdk-devbind.py -u "${OVSDEV_PCIID}"
# bind vfio-pci driver
dpdk-devbind.py --bind=vfio-pci "${OVSDEV_PCIID}"
# check driver assignment
dpdk-devbind.py --status
lspci -k

```
# configure openvswitch

## ubuntu

https://ubuntu.com/server/docs/openvswitch-dpdk

```shell
# using iommu
ovs-vsctl set Open_vSwitch . other_config:vhost-iommu-support=true
# dpdk init
ovs-vsctl set Open_vSwitch . "other_config:dpdk-init=true"
# run on core 0 only
ovs-vsctl set Open_vSwitch . "other_config:dpdk-lcore-mask=0x2"
# Allocate 2G huge pages (not Numa node aware)
ovs-vsctl set Open_vSwitch . "other_config:dpdk-alloc-mem=2048"
# limit to one whitelisted device
ovs-vsctl set Open_vSwitch . "other_config:dpdk-extra=--pci-whitelist=${OVSDEV_PCIID}"
# restart openvswitch
systemctl restart openvswitch-switch
```


## Attaching DPDK ports to OpenVswitch


``` shell


ovs-vsctl add-br ovsdpdkbr0 -- set bridge ovsdpdkbr0 datapath_type=netdev
ovs-vsctl add-port ovsdpdkbr0 dpdk0 -- set Interface dpdk0 type=dpdk "options:dpdk-devargs=${OVSDEV_PCIID}"

ovs-vsctl add-port ovsdpdkbr0 vhost-user-1 -- set Interface vhost-user-1 type=dpdkvhostuserclient "options:vhost-server-path=/var/run/vhost-user-socket-1"



```

# configure libvirt

## ubuntu

https://ubuntu.com/server/docs/virtualization-libvirt

apt install qemu-kvm libvirt-daemon-system


## Download VM image

https://cloud-images.ubuntu.com/

## libvrit iommu domain

https://raw.githubusercontent.com/libvirt/libvirt/master/tests/qemuxml2argvdata/intel-iommu-device-iotlb.xml


## DEBUG

ovs-vsctl get Open_vSwitch . dpdk_initialized
ovs-vsctl get Open_vSwitch . dpdk_version

