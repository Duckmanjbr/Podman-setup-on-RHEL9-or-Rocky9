# Podman container setup on RHEL9/Rocky9 with security implications

1. [Container_Auto_Update](#enable-automatic-updates-for-containers-if-needed)

2. [Systemd_Reload](#after-any-quadlet-file-change-reload-systemd-daemon-to-sync-with-systemd)

3. [Quadlet_File_location](#save-quadlet-files-in-one-of-two-locations)

4. [DryRun_Testing](#once-created-test-quadlet-files-using-quadlet-dryrun--any-errors-found-will-be-listed-at-the-end-of-the-output-explaining-the-issue)

5. [Quadlet_Generation_Error_Checking](#check-for-quadlet-generation-errors-after-reloading-systemd)

6. [Running_Quadlet_Files](#running-quadlet-processes-individually)

7. [Enable_Rootless_Auto_Start](#enable-systemd-lingering-for-the-user)

8. [Auto_Start_Container](#enable-the-service-to-start-automatically)

9. [Volume_Files_Setup](#volume-files)

10. [Network_Files_Setup](#network-files)

## Assumptions:
1. CCNA level of networking knowledge
2. RHEL OS understanding
3. Basic Podman understanding
4. Basic Quadlets understanding

## Podman 
Rootless Podman is the prefered method of container execution.  For Quadlets the .container file is the heart of the application so I'll cover that last.  Before we get to each application there are some other basics of understanding required. Each type of Quadlet file creates a systemd entry and therfore can be run as a seperate process.  This is good for testing out the configuration but there are other testing methods to consider using for testing as well.

### Enable automatic updates for containers if needed.
* Run as root: `sudo systemctl enable --now podman-auto-update.service`
* Run as a user `systemctl --user enable --now podman-auto-update.service`

#### After any quadlet file change reload systemd daemon to sync with systemd.
* Run as root: `sudo systemctl daemon-reload`
* Run as a user `systemctl --user daemon-reload`

#### Save quadlet files in one of two locations:
* Root access: `/etc/containers/systemd/`
* Rootless access: `~/.config/containers/systemd/`

#### Once created, test quadlet files using quadlet dryrun.  Any errors found will be listed at the end of the output explaining the issue.

* Testing rootfull container files stored in /etc/containers/systemd/: `sudo /usr/libexec/podman/quadlet -dryrun`
* Testing rootless container files stored in $HOME/.config/containers/systemd/: `/usr/libexec/podman/quadlet -dryrun --user`

#### Check for quadlet generation errors after reloading systemd:
* `sudo journalctl -xe | grep -i quadlet`

#### Running Quadlet processes individually:
* Run the container and any sub-process (.volume, .network, etc) as root: `sudo systemctl start [app-name].service`
* Run the .volume quadlet as root: `sudo systemctl start [volume-file-name]-volume.service`
* Run the .network quadlet as root: `sudo systemctl start [network-file-name]-network.service`
* Run the container and any sub-process (.volume, .network, etc) as user: `systemctl --user start [app-name].service`
* Run the .volume quadlet as user: `systemctl --user start [volume-file-name]-volume.service`
* Run the .network quadlet as user: `systemctl --user start [network-file-name]-network.service`

#### Enable Systemd Lingering for the User:
Systemd lingering allows a user’s systemd session to remain active even when the user is not logged in, enabling user-specific services (like rootless Podman containers) to start at boot. This ensures the user’s systemd session starts at boot and remains active without a login.
* `sudo loginctl enable-linger [username]`

#### Enable the service to start automatically:
* Enable rootfull: `sudo systemctl enable [container-name].service`
  - add the `WantedBy=multi-user.target` to the .container file in `/etc/containers/systemd/`.
* Enable as rootless: `systemctl --user enable [container-name].service`
  - add the `WantedBy=default.target` to the .container file in `~/.config/containers/systemd/`.




## Volume files.
Volume files ([name].volume) are the preferred Quadlet method of calling volumes in Podman. A .volume file is a systemd unit file with a .volume extension that tells Podman to create and manage a Podman volume. These files are processed by the podman-systemd-generator when the `systemctl daemon-reload` command is executed to create systemd services that manage the volume lifecycle. Their purpose:
  * Define persistent storage volumes for Podman containers or pods
  * Automate volume creation and ensure they are available when needed
  * Integrate volume management with systemd, allowing volumes to be created, started, or removed as part of the system’s service management
  * Key Benefit: Simplifies volume configuration by replacing manual podman volume create commands with declarative configuration files that systemd manages, enabling automatic startup and dependency handling.

An extreamly simplified example could be something like this db_data.volume file:
```
[Unit]
Description=Podman volume for database data

[Volume]
VolumeName=db_data
Driver=local
Label=app=database

[Install]
#To auto run as user at login:
WantedBy=default.target
#To auto run as root:
WantedBy=multi-user.target
```

A more specific configuration wanting to specify a file path would look something like this homebridge.volume file:
```
[Unit]
Description=Homebridge volume for container data

[Volume]
VolumeName=homebridge
Device=/mnt/NAS-Vol2/Containers/homebridge/
Driver=local
Options=bind

[Install]
#To auto run as user at login:
WantedBy=default.target
#To auto run as root:
WantedBy=multi-user.target
```



## Network files.
Network files ([name].network) are the preferred Quadlet method of calling/creating networks for container use.  These can be named to match the container application or named to cover a larger scope depending on network use case. There are a few major types of container networks and both have advantages and disadvantages. Once a network is created you can inspect the network with the following:

`sudo podman network ls`

`sudo podman network inspect [network-name]`

#### Type 1 - Macvlan
Networks created with "macvlan" allow direct network connectivity bypassing the host IP stack.  Macvlan is riskier in environments where network segmentation is weak or where containers run untrusted workloads, as it exposes containers directly to the network. It’s better suited for scenarios where containers need to act like physical devices (e.g., IoT or network appliances) and where strong network-level controls (e.g., VLANs, firewall rules) are in place. It has the following disadvantages:

  1. Increased Network Exposure:
Containers using macvlan are directly exposed to the physical network, as they receive their own IP and MAC address. This makes them more discoverable and potentially vulnerable to attacks from other devices on the same network, as they bypass the host’s network stack isolation. If the network lacks proper segmentation (e.g., VLANs or firewall rules), containers could be targeted directly by external devices.

  2. Limited Host Firewall Control:
Since macvlan traffic bypasses the host’s network stack (e.g., iptables or nftables), the host’s firewall rules cannot easily filter or monitor container traffic. You’d need to rely on network-level firewalls or container-specific rules, which can complicate security management.

  3. Potential for MAC Address Conflicts:
If not configured carefully, macvlan can lead to MAC address conflicts on the network, especially in dynamic environments. While not a direct security issue, misconfiguration can cause network instability, potentially enabling denial-of-service (DoS) scenarios.

  4. No Network Address Translation (NAT):
Unlike bridge networking, macvlan doesn’t use NAT, so containers are directly accessible from the network. This can be a security risk if containers run services that aren’t properly secured, as there’s no NAT layer to obscure them.

  5. Promiscuous Mode Requirement:
Some macvlan setups require the host’s network interface to be in promiscuous mode, which could allow the host to capture all network traffic, potentially exposing sensitive data if the host is compromised.

A good example of connecting a container directly to an interface (homebridge.network). 

  ```
  [Network]
  NetworkName=homebridge
  Driver=macvlan
  Subnet=192.168.2.0/24
  Gateway=192.168.2.1
  PodmanArgs=--interface-name=br-IOT
  ```

This example creates a network called "homebridge", connects to the linux bridge named "br-IOT" and uses the 192.168.x.x subnet.  In this example a Linux bridge is used as the connection interface but this could be a normal interface connection as well (eno0, enp2s0, etc) depending on the desired use case.  This could also be a stacked interface if the required outcome is utilizing a vlan in a trunked architecture. For example:
  
  enp2s0 -> br-vlan0(bridge) -> vlan.4(vlan) -> br-vlan4(bridge)

Any of the interfaces used in the example above (enp2s0, br-vlan0, vlan.4, br-vlan4) will work for the Quadlet configuration depending on what type of connectivity and network design you need for your enviornment. This direct layer two connection type has advantages but be careful to mitigate high risk areas.
  
  * Use VLANs to isolate container traffic from the rest of the network
  * Implement strict network firewall rules to control container access
  * Avoid running untrusted workloads with macvlan
  * Monitor network traffic for anomalies, as containers are more exposed


#### Type 2 - Bridge
Networks created as a "bridge" allow connectivity from behind the host network IP stack.  Bridge networking creates a virtual network bridge on the host, and containers connect to it. Traffic is typically NAT’d to the outside network. This is generally more secure for most use cases because it provides better isolation through NAT and allows the host’s firewall to filter traffic but does have the following disadvantages:

  1. Single Point of Failure:
The bridge network is managed by the host, so a misconfiguration or vulnerability in the host’s network stack (e.g., iptables) could impact all containers. A compromised host could expose or manipulate container traffic.

  2. NAT Overhead and Complexity:
Bridge networking relies on NAT for external communication, which can introduce complexity in firewall rules and port forwarding. Misconfigured NAT rules might inadvertently expose container ports to the external network.

  3. Shared Network Namespace:
Containers on the same bridge share a virtual network, so a compromised container could potentially interfere with others on the same bridge (e.g., via ARP spoofing or IP conflicts), though this is mitigated by Podman’s default network isolation.

  4. Limited Network Visibility:
Containers behind a bridge are less visible to the external network due to NAT, which is generally good for security. However, this can make it harder to detect malicious activity within a container, as traffic is masked by the host’s IP.

A good example of a basic bridge .network file configuration (test_range.network):
```
[Network]
NetworkName=test_range
Subnet=10.0.0.0/16
DisableDNS=true

[Install]
WantedBy=default.target
```

This example creates a network named "test_range" and utilizes the 10.0.x.x subnet.  This will be NAT'd behind the host Firewall so additional FW settings may be needed.

For bridge connections, though generally more secure, try to mitigate where possible.

  * Regularly audit iptables/nftables rules to ensure proper NAT and port forwarding
  * Use Podman’s network namespaces to isolate containers further
  * Limit exposed ports to only those necessary for the application
  * Keep the host’s kernel and network stack updated to avoid vulnerabilities.

## [Continue on to .container files and their setup](/Quadlet_Containers.md)
