# Configuration setup:

## Create folder structure for containers if non-default is wanted:
Examples:
* `mkdir /home/[user]/containers`
* `mkdir /home/[user]/containers/[container-app]`

## Install baseline software:
* `sudo dnf -y install podman container-tools`
* If also running Cockpit: `sudo dnf -y install podman container-tools cockpit-podman`

   
## Setup Podman:
Start AutoUpdate timer to check for latest images - `sudo systemctl enable --now podman-auto-update.timer`
Test the AutoUpdate `sudo podman auto-update`

1. `sudo systemctl enable --now podman.socket`
2. `systemctl --user enable --now podman.socket`

## Allow users NET_ADMIN capability if needed for direct network permissions:
1. `sudo setcap cap_net_admin+eip /usr/libexec/podman/netavark`
2. `sudo setcap cap_net_admin+eip /usr/bin/podman`


### Setup Containers:
[Container_Setup](https://github.com/Duckmanjbr/Podman-setup-on-RHEL9-Rocky9/blob/main/Quadlet_Setup.md#podman-container-setup-on-rhel9rocky9-with-security-implications)
