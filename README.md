# qemu/kvm (commnads to work on various systems)

---

1. How to use bridged networking with libvert and KVM
Verify the host’s CPU will support KVM virtualization:
```sh
$ kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used
```

Add your user to the relevant groups. This will allow you to manage VMs without sudo:
```sh
sudo adduser zafar libvirt
sudo adduser zafar kvm
# here zafar is my name, you'll use your username; can be found using: whoami.
```

Check that libvirtd is running:
```sh
$ sudo systemctl status libvirtd
```

### Networking setup 
Run `ip add` and note the name of your Ethernet interface. In my case, it’s `enp0s18`.

Examine the contents of `/etc/netplan`. In my case, there was one file, with these contents:
```yaml
network:
  ethernets:
    enp0s18:
      dhcp4: true
  version: 2
```

If yours differs substantially, take care to understand what this file is doing! Some of the reference links below might help.

Back up the YAML file from `/etc/netplan`, then edit it. Add a bridges section, so the result looks like the following code sample. Note that `enp0s18` should be replaced by the name of your Ethernet interface:
```yaml
network:
  version: 2
  ethernets:
    enp0s18:
      dhcp4: true
  bridges:
    br0:
      dhcp4: yes
      interfaces:
        - enp0s18
```

Test it with `sudo netplan try`. This allows you to test the configuration and make sure it’s working properly. It will offer to make the changes permanent for you, and if you don’t accept that prompt within a minute or two the changes will be reverted automatically.

Assuming that it worked, `br0` is now assigned an IP address by your local DHCP server. You can verify this by running `ip add`.

You’ll need to update DHCP reservations for the host, if you have any, to use the MAC address of the bridge interface. ip add will show you this MAC address.

Finally, create a file called `kvm-hostbridge.xml` in a location of your choice, with the following content:
```xml
<network>
  <name>hostbridge</name>
  <forward mode="bridge"/>
  <bridge name="br0"/>
</network>
```

Create and enable this network by running:
```sh
virsh net-define /path/to/my/kvm-hostbridge.xml
virsh net-start hostbridge
virsh net-autostart hostbridge
```

**iptables considerations for hosts that also run Docker**

Installing Docker enables netfilter for bridge interfaces on the host machine. This is needed because Docker creates and manages iptables rules to isolate bridge networks — the default Docker network type — from each other and allow them access to the network.

These rules will, by default, break networking for your VMs. In my case I wanted all VMs to have access to the network as if they were real, physical machines; and to do this I added a single iptables rule on the host:
```sh
sudo iptables -A FORWARD -p all -i br0 -j ACCEPT
```

## Creating a VM on this network

Assuming nothing has failed so far, you’re ready to create a VM connected to this network.

To use this network when creating a VM, pass `--network network=hostbridge` to `virt-install`. Here’s a complete example which I used successfully:
```
virt-install \
  --name altdns \
  --description "alternate DNS server for the home network" \
  --memory 2048 \
  --vcpus 2 \
  --disk path=/mnt/storage/vm/altdns/disk.qcow2,size=32 \
  --cdrom /mnt/scratch/ubuntu-22.04.3-live-server-amd64.iso \
  --graphics vnc \
  --os-variant ubuntu22.04 \
  --virt-type kvm \
  --autostart \
  --network network=hostbridge
```

(`virt-install --help` can help you figure out what the other options do.)

After the machine boots, your router should see it as a new machine and issue it an IP address over DHCP just as it would anything else on the network.

## Accessing the installer over VNC

The command above tells `virt-install` we want to use VNC to view the VM’s display and give it keyboard/mouse input. Note that you’ll connect to the host and not to the guest’s IP address with your VNC client.
First, figure out which port to use:
```sh
virsh vncdisplay altdns
# 127.0.0.1:1
```

This tells us that we can connect to the VNC server on the host’s port `5900 + 1`.

It also tells us that the VNC server on this host is listening on `localhost` only. You can use SSH port forwarding to create a tunnel from port `5999` on your desktop to port `5901` on the server:
```sh
ssh -L 5999:localhost:5901 my-vm-host.local
```

Once that’s up and running, you can open your favorite VNC client, connect to `localhost:5999` (no authentication required), and walk through the Ubuntu installer to get your VM up and running.

### Helpful Extra Commnads

---

When troubleshooting or trying to understand the network state on your host VM, the following commands are useful:
```sh
virsh net-list
virsh net-info hostbridge
virsh net-dhcp-leases hostbridge
sudo brctl show br0
```
