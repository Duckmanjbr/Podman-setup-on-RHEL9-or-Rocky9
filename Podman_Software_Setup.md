# Configuration setup:

## Create folder structure for containers if non-default is wanted:
Examples:
* `mkdir /home/[user]/containers`
* `mkdir /home/[user]/containers/[container-app]`

## Create users for services if needed:
1. Homebridge:  `sudo adduser -M -u 1012 homebridge & sudo usermod -s /sbin/nologin homebridge`
2. Pihole:  `sudo adduser -M -u 1010 pihole & sudo usermod -s /sbin/nologin pihole`
3. Unifi:  `sudo adduser -M -u 1011 unifi & sudo usermod -s /sbin/nologin unifi`
   
## Install baseline software:
1. Install software: `sudo dnf -y install podman container-tools`
2. If also running Cockpit: `sudo dnf -y install podman container-tools cockpit-podman`

   
## Setup Podman:
1. Start AutoUpdate timer to check for latest images:  `sudo systemctl enable --now podman-auto-update.timer`
2. Test the AutoUpdate:  `sudo podman auto-update`
3. Enable podman:  `sudo systemctl enable --now podman.socket`
4. Enable user podman:  `systemctl --user enable --now podman.socket`

## Allow users NET_ADMIN capability if needed for direct network permissions:
1. `sudo setcap cap_net_admin+eip /usr/libexec/podman/netavark`
2. `sudo setcap cap_net_admin+eip /usr/bin/podman`


## Allow lower unprivileged port use with Podman:
1. Lower the kernel's unprivileged port threshold so rootless users can bind to ports like 80 and 443:
  `echo 'net.ipv4.ip_unprivileged_port_start=80' | sudo tee /etc/sysctl.d/99-podman-privileged-ports.conf`
2. Apply:
   `sudo sysctl -p /etc/sysctl.d/99-podman-privileged-ports.conf`




### Setup Containers:
[Container Quadlet_Setup](https://github.com/Duckmanjbr/Podman-setup-on-RHEL9-Rocky9/blob/main/Quadlet_Setup.md#podman-container-setup-on-rhel9rocky9-with-security-implications)
