# Lima Templates

## Prerequisites (Run on Mac)
Goal: Running lima VMs within a bridged network as this allows bypassing port-forwarding for Ingress, and Pods in the cluster can just hairpin instead of having to rewrite DNS to use the in-cluster Ingress hostname.

Install `lima` and `socket_vmnet`:
```bash
brew install lima socket_vmnet
```

Update the file called `~/.lima/_config/networks.yaml` with the following content to enable bridge network connectivity to the VM:
```bash
paths:
  # Update `socketVMNet` to your actually installed `socket_vmnet` binary version, if it doesn't match yet:
  socketVMNet: "/opt/homebrew/Cellar/socket_vmnet/1.1.4/bin/socket_vmnet"

# Change the existing `shared` network `dhcpEnd` to `192.168.105.100` so we have space for static addressing:
networks:
  shared:
    mode: shared
    gateway: 192.168.105.1
    dhcpEnd: 192.168.105.100
    netmask: 255.255.255.0

networks:
- lima: shared
  # Set a static MAC so your VM will always get the same IP (in combination with the /etc/bootptab config below)
  macAddress: "de:ad:be:ef:00:01"
```

Create/edit `/etc/bootptab` and add the following lines:
```bash
# bootptab
%%
# hostname      hwtype  hwaddr              ipaddr          bootfile
docker          1       de:ad:be:ef:00:01   192.168.105.101
go-dev          1       de:ad:be:ef:00:02   192.168.105.102
go-dev-vz       1       de:ad:be:ef:00:03   192.168.105.103
```

Start/reload the the DHCP daemon:
```bash
sudo /bin/launchctl load -w /System/Library/LaunchDaemons/bootps.plist
# or the reload:
#sudo /bin/launchctl kickstart -kp system/com.apple.bootpd

# to stop the service:
# sudo /bin/launchctl unload -w /System/Library/LaunchDaemons/bootps.plist
```

Load `sudo` dependency for Lima's support for `socket_vmnet`:
```bash
limactl sudoers | sudo tee /etc/sudoers.d/lima
```

## Virtual Machines
### Go Dev Environment
(Used for [Cilium development](https://docs.cilium.io/en/stable/contributing/development/dev_setup).)

```bash
limactl start --name=go-dev go-dev.yaml --tty=false
```

Temporarily hack: Initially restart the `go-dev` once to fully apply everything:
```bash
limactl stop go-dev
limactl start go-dev
```

### Running Docker
```bash
limactl start --name=docker docker.yaml --tty=false
```

## Access Virtual Machine
```bash
# List all VMs:
limactl list

# Open shell:
limactl shell <VM-name>
```

## Clean Up Environment
```bash
# List all VMs:
limactl list

# Stop and delete a VM:
limactl stop <VM-name>
limactl delete <VM-name>
```

## Sources
- https://lima-vm.io/docs/config/network/#socket_vmnet
- https://github.com/lima-vm/socket_vmnet?tab=readme-ov-file#how-to-reserve-dhcp-addresses
- https://medium.com/@maninder.bindra/remote-debugging-go-code-within-a-nested-lima-linux-vm-on-a-mac-using-vscode-acfab96b7b3a