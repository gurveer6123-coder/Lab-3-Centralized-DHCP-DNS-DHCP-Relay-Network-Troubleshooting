# Lab 3 — Centralized DHCP, DNS, DHCP Relay & Network Troubleshooting

## Project Overview

This lab demonstrates the configuration and troubleshooting of common network services in a small enterprise-style network using Cisco Packet Tracer.

The network contains two separate IPv4 networks connected through a Cisco router. User computers are located on one subnet while a centralized server providing DHCP and DNS services is located on another subnet.

Because DHCP broadcasts do not normally pass through routers, DHCP Relay was configured using the Cisco `ip helper-address` command.

The lab also included several intentionally created network failures to practice a structured troubleshooting process.

### Technologies and Concepts Practiced

* IPv4 addressing
* Subnetting
* Default gateways
* Static IP addressing
* Dynamic Host Configuration Protocol (DHCP)
* Centralized DHCP server
* DHCP Relay
* `ip helper-address`
* Domain Name System (DNS)
* DNS A records
* Inter-subnet routing
* Cisco router CLI
* Cisco switch CLI
* Interface troubleshooting
* Switch port troubleshooting
* DHCP failure diagnosis
* DNS troubleshooting
* Connectivity testing
* `ping`
* `ipconfig`
* `show ip interface brief`
* `show interfaces status`
* `show running-config`

---

# Network Topology

The topology consists of:

* 1 Cisco 2911 Router
* 2 Cisco 2960 Switches
* 3 Client PCs
* 1 Server
* 2 separate IPv4 networks

The PCs are located on the user LAN while the DHCP/DNS server is located on a separate server LAN.

```text
 PC0 -----\
 PC1 ------ SW1 -------- R1 -------- SW2 -------- Server0
 PC2 -----/
```

### Logical Network

```text
USER LAN
192.168.10.0/24

PC0
PC1
PC2
  |
 SW1
  |
R1 G0/0
192.168.10.1
  |
  | Routing
  |
R1 G0/1
192.168.50.1
  |
 SW2
  |
Server0
192.168.50.10

SERVER LAN
192.168.50.0/24
```

## Topology Screenshot

![Network Topology](images/01-topology\(1\).png)

---

# IP Addressing Plan

Two different `/24` networks were used.

| Device  | Interface     | IP Address    | Subnet Mask   | Purpose            |
| ------- | ------------- | ------------- | ------------- | ------------------ |
| Router0 | G0/0          | 192.168.10.1  | 255.255.255.0 | User LAN Gateway   |
| Router0 | G0/1          | 192.168.50.1  | 255.255.255.0 | Server LAN Gateway |
| Server0 | FastEthernet0 | 192.168.50.10 | 255.255.255.0 | DHCP/DNS Server    |
| PC0     | FastEthernet0 | DHCP          | 255.255.255.0 | User Client        |
| PC1     | FastEthernet0 | DHCP          | 255.255.255.0 | User Client        |
| PC2     | FastEthernet0 | DHCP          | 255.255.255.0 | User Client        |

### Network 1 — User LAN

```text
Network Address:  192.168.10.0/24
Default Gateway:  192.168.10.1
DHCP Start:       192.168.10.20
Broadcast:        192.168.10.255
```

Addresses below `.20` were kept outside of the DHCP client range so they could be used for infrastructure devices if needed.

Example:

```text
192.168.10.1     Router
192.168.10.2-.19 Reserved/Static Devices
192.168.10.20+   DHCP Clients
```

### Network 2 — Server LAN

```text
Network Address:  192.168.50.0/24
Default Gateway:  192.168.50.1
Server Address:   192.168.50.10
Broadcast:        192.168.50.255
```

---

# Router Configuration

The Cisco router connects the user network and server network.

## Configure User LAN Interface

```text
enable
configure terminal

interface GigabitEthernet0/0
ip address 192.168.10.1 255.255.255.0
no shutdown
exit
```

## Configure Server LAN Interface

```text
interface GigabitEthernet0/1
ip address 192.168.50.1 255.255.255.0
no shutdown
exit

end
```

The router therefore has one interface in each network:

```text
G0/0 = 192.168.10.1/24
G0/1 = 192.168.50.1/24
```

---

# Router Interface Verification

The following command was used to verify interface addressing and operational status:

```text
show ip interface brief
```

Both configured interfaces showed:

```text
GigabitEthernet0/0   192.168.10.1   up   up
GigabitEthernet0/1   192.168.50.1   up   up
```

`up/up` confirms that both the physical interface and line protocol are operational.

![Router Interface Verification](images/02-router-interfaces.png)

---

# Server Static IP Configuration

Server0 was configured with a static IPv4 address because infrastructure servers should normally have predictable addresses.

The following configuration was used:

```text
IP Address:       192.168.50.10
Subnet Mask:      255.255.255.0
Default Gateway:  192.168.50.1
DNS Server:       192.168.50.10
```

The default gateway is `192.168.50.1` because Router0 G0/1 is the router interface on the server's local subnet.

![Server Static IP](images/04-server-static-ip.png)

Connectivity between the server and its gateway was verified using:

```text
ping 192.168.50.1
```

---

# Centralized DHCP Server Configuration

Instead of configuring DHCP directly on the router, Server0 was configured as the DHCP server.

This represents a more centralized enterprise-style design.

The DHCP pool was configured with:

```text
Pool Name:         Users
Default Gateway:   192.168.10.1
DNS Server:        192.168.50.10
Start IP Address:  192.168.10.20
Subnet Mask:       255.255.255.0
Maximum Users:     100
```

The DHCP server can therefore provide clients with addresses beginning from:

```text
192.168.10.20
```

For example:

```text
PC0 → 192.168.10.20
PC1 → 192.168.10.21
PC2 → 192.168.10.22
```

The exact assigned address depends on the order in which clients request DHCP leases.

![DHCP Server Pool](images/05-dhcp-pool.png)

---

# DHCP Relay Configuration

A major challenge in this topology is that the DHCP clients and DHCP server are located on different networks.

The clients are located on:

```text
192.168.10.0/24
```

while the DHCP server is located on:

```text
192.168.50.0/24
```

DHCP discovery initially uses broadcast traffic.

Routers do not normally forward Layer 3 broadcasts between subnets.

Therefore, without additional configuration:

```text
PC DHCPDISCOVER
       |
      SW1
       |
      R1
       X
DHCP broadcast does not reach Server0
```

To solve this problem, DHCP Relay was configured on Router0.

The following command was applied to the interface receiving DHCP broadcasts from the clients:

```text
configure terminal

interface GigabitEthernet0/0
ip helper-address 192.168.50.10

end
```

The important configuration is:

```text
interface GigabitEthernet0/0
 ip address 192.168.10.1 255.255.255.0
 ip helper-address 192.168.50.10
```

`ip helper-address` tells the router to relay supported UDP broadcast traffic received on G0/0 to Server0 at `192.168.50.10`.

The flow becomes:

```text
PC
 |
 | DHCP Broadcast
 v
SW1
 |
 v
Router G0/0
 |
 | ip helper-address
 v
192.168.50.10
DHCP Server
```

![DHCP Relay Configuration](images/03-dhcp-relay.png)

---

# DNS Server Configuration

Server0 was also configured as the DNS server.

A DNS A record was created:

```text
Name:     server.company.local
Type:     A Record
Address:  192.168.50.10
```

This allows clients to communicate with the server using:

```text
server.company.local
```

instead of remembering:

```text
192.168.50.10
```

DNS resolution works as follows:

```text
server.company.local
        |
        v
DNS Server
        |
        v
192.168.50.10
```

![DNS A Record](images/06-dns-record.png)

---

# DHCP Client Verification

The PCs were configured to obtain their network settings automatically using DHCP.

A client received:

```text
IPv4 Address:     192.168.10.21
Subnet Mask:      255.255.255.0
Default Gateway:  192.168.10.1
```

Other DHCP clients may receive `.20`, `.22`, or other available addresses from the pool.

The following command was used to inspect the client configuration:

```text
ipconfig
```

or:

```text
ipconfig /all
```

![PC DHCP Address](images/07-pc-dhcp-address.png)

This confirmed that DHCP communication successfully crossed the router using DHCP Relay.

---

# DNS and End-to-End Connectivity Test

DNS resolution was tested from a client with:

```text
ping server.company.local
```

The hostname successfully resolved to:

```text
192.168.50.10
```

and the server returned ICMP replies.

![DNS Connectivity Test](images/08-connectivity-dns-test.png)

This test confirms several things simultaneously:

```text
PC IP Configuration     ✓
Switch Connectivity     ✓
Default Gateway         ✓
Router Connectivity     ✓
Inter-Subnet Routing    ✓
DNS Resolution          ✓
Server Connectivity     ✓
```

---

# Troubleshooting Scenarios

An important part of this lab was intentionally creating failures and using troubleshooting commands to identify the cause.

---

# Troubleshooting Scenario 1 — Wrong Default Gateway

A client was intentionally configured with an incorrect default gateway.

Example incorrect configuration:

```text
IP Address:       192.168.10.50
Subnet Mask:      255.255.255.0
Default Gateway:  192.168.10.100
```

The correct gateway should have been:

```text
192.168.10.1
```

A client may still communicate with devices on its own subnet while being unable to reach remote networks.

Troubleshooting command:

```text
ipconfig /all
```

Troubleshooting logic:

```text
Local communication works
        |
Remote communication fails
        |
        v
Check Default Gateway
```

The problem was corrected by restoring:

```text
Default Gateway: 192.168.10.1
```

The client was later returned to DHCP configuration.

---

# Troubleshooting Scenario 2 — DHCP Relay Failure

The DHCP relay configuration was intentionally removed from Router0.

Command:

```text
configure terminal

interface GigabitEthernet0/0
no ip helper-address 192.168.50.10

end
```

Without the helper address, the router stopped forwarding DHCP requests to the remote DHCP server.

After forcing the client to request another DHCP lease, the client failed to receive an IPv4 address.

In Packet Tracer this resulted in:

```text
IPv4 Address:    0.0.0.0
Subnet Mask:     0.0.0.0
Default Gateway: 0.0.0.0
```

![DHCP Failure](images/09-dhcp-failure.png)

This indicated that the client had failed to obtain valid network configuration from DHCP.

In real Windows environments, DHCP failure can also result in an APIPA address from:

```text
169.254.0.0/16
```

The troubleshooting process included checking:

```text
ipconfig /all
```

and then checking Router0:

```text
show ip interface brief
show running-config
```

The router interfaces remained operational, but the `ip helper-address` configuration was missing.

The issue was fixed with:

```text
configure terminal

interface GigabitEthernet0/0
ip helper-address 192.168.50.10

end
```

The client was then able to obtain a valid DHCP lease again.

---

# Troubleshooting Scenario 3 — DNS Failure

The DNS A record for:

```text
server.company.local
```

was intentionally removed.

The server could still be reached using:

```text
ping 192.168.50.10
```

but name-based communication failed.

This created an important troubleshooting distinction:

```text
Ping by IP works
Ping by hostname fails
        |
        v
Likely DNS Problem
```

The DNS configuration was checked on Server0 and the A record was restored:

```text
server.company.local → 192.168.50.10
```

After restoring the DNS record:

```text
ping server.company.local
```

worked successfully again.

---

# Troubleshooting Scenario 4 — Router Interface Failure

Router0 G0/1 was intentionally disabled:

```text
configure terminal

interface GigabitEthernet0/1
shutdown
```

The interface status was checked with:

```text
show ip interface brief
```

The output showed:

```text
GigabitEthernet0/1
192.168.50.1
administratively down
down
```

![Router Interface Failure](images/10-router-interface-failure.png)

The phrase:

```text
administratively down
```

indicates that the interface was manually disabled through configuration.

The problem was fixed using:

```text
configure terminal

interface GigabitEthernet0/1
no shutdown

end
```

Afterward:

```text
show ip interface brief
```

returned:

```text
GigabitEthernet0/1   192.168.50.1   up   up
```

---

# Troubleshooting Scenario 5 — Incorrect Client IP Configuration

A client was intentionally assigned an address from the wrong subnet.

Example:

```text
IP Address:       192.168.20.50
Subnet Mask:      255.255.255.0
Default Gateway:  192.168.10.1
```

The problem is that:

```text
PC Network:
192.168.20.0/24

Gateway Network:
192.168.10.0/24
```

The PC and its gateway were not in the same subnet.

The issue could be diagnosed using:

```text
ipconfig /all
```

and comparing the client's IP address, subnet mask, and default gateway.

The client was finally returned to DHCP so the correct configuration could be automatically restored.

---

# Troubleshooting Scenario 6 — Switch Port Failure

The switch port connected to a client was intentionally shut down.

On Switch0:

```text
enable
configure terminal

interface FastEthernet0/1
shutdown

end
```

The port status was then checked using:

```text
show interfaces status
```

The output showed:

```text
Fa0/1     disabled
```

while other connected interfaces remained operational.

![Switch Port Failure](images/11-switch-port-failure.png)

A more detailed interface check can also be performed with:

```text
show interfaces FastEthernet0/1
```

A manually disabled port may display:

```text
FastEthernet0/1 is administratively down, line protocol is down
```

The switch port was restored using:

```text
configure terminal

interface FastEthernet0/1
no shutdown

end
```

The port then returned to:

```text
connected
```

---

# Final Network Verification

After all troubleshooting exercises, every intentionally created problem was repaired.

The following tests were performed:

```text
ipconfig
```

to verify valid DHCP configuration.

Then:

```text
ping 192.168.10.1
```

to test the user's default gateway.

Then:

```text
ping 192.168.50.10
```

to test communication between the user and server networks.

Finally:

```text
ping server.company.local
```

was used to verify DNS resolution and complete end-to-end connectivity.

The hostname successfully resolved to:

```text
192.168.50.10
```

and all packets were successfully returned with:

```text
0% packet loss
```

![Final Verification](images/12-final-verification.png)

---

# Troubleshooting Commands Used

## Windows / Packet Tracer PC Commands

### View IP Configuration

```text
ipconfig
```

### View Detailed IP Configuration

```text
ipconfig /all
```

### Test Default Gateway

```text
ping 192.168.10.1
```

### Test Server by IP

```text
ping 192.168.50.10
```

### Test DNS Resolution

```text
ping server.company.local
```

---

# Router Troubleshooting Commands

### Display Interface Status

```text
show ip interface brief
```

Useful for identifying:

```text
up/up
down/down
administratively down/down
```

### Display Router Configuration

```text
show running-config
```

Useful for verifying:

* Interface IP addresses
* DHCP Relay
* Interface shutdown state
* Other configuration

---

# Switch Troubleshooting Commands

### Display Switch Port Status

```text
show interfaces status
```

Useful for identifying whether ports are:

```text
connected
notconnect
disabled
```

### Inspect a Specific Interface

```text
show interfaces FastEthernet0/1
```

### Display VLAN Information

```text
show vlan brief
```

---

# Troubleshooting Methodology Learned

This lab demonstrated that troubleshooting should be performed systematically instead of randomly changing configuration.

A basic troubleshooting process used during the lab was:

```text
1. Check physical connection
        |
        v
2. Check switch/interface status
        |
        v
3. Check client IP configuration
        |
        v
4. Check subnet mask
        |
        v
5. Check default gateway
        |
        v
6. Test local gateway
        |
        v
7. Test remote destination by IP
        |
        v
8. Test DNS/name resolution
        |
        v
9. Check DHCP/DNS server
        |
        v
10. Verify router configuration
```

Examples of symptoms and likely causes:

| Symptom                                      | Likely Area to Investigate   |
| -------------------------------------------- | ---------------------------- |
| `0.0.0.0` or `169.254.x.x`                   | DHCP                         |
| Cannot reach default gateway                 | Local LAN / IP / switch      |
| Local network works but remote network fails | Gateway / Routing            |
| Ping by IP works but hostname fails          | DNS                          |
| `administratively down`                      | Interface manually shut down |
| Switch port shows `disabled`                 | Switch port shutdown         |
| Wrong subnet/IP combination                  | Client configuration         |
| DHCP server on another subnet not responding | DHCP Relay                   |

---

# Key Concepts Learned

## DHCP

DHCP automatically provides clients with:

```text
IP Address
Subnet Mask
Default Gateway
DNS Server
```

This reduces the need for manually configuring every client.

---

## DHCP Relay

DHCP broadcasts normally cannot cross routers.

The Cisco command:

```text
ip helper-address 192.168.50.10
```

allows Router0 to forward DHCP requests from the user LAN to the centralized DHCP server.

---

## DNS

DNS translates human-readable names into IP addresses.

Example:

```text
server.company.local
        ↓
192.168.50.10
```

A successful IP ping combined with a failed hostname ping is an important indication of a possible DNS problem.

---

## Default Gateway

The default gateway allows a device to communicate with destinations outside its local subnet.

For the user network:

```text
192.168.10.1
```

is the gateway.

For the server network:

```text
192.168.50.1
```

is the gateway.

---

## Static vs Dynamic IP Addresses

Infrastructure devices such as servers and routers commonly use static addresses.

Example:

```text
Router G0/0 = 192.168.10.1
Router G0/1 = 192.168.50.1
Server0     = 192.168.50.10
```

End-user devices can receive addresses dynamically through DHCP.

---

# Final Working Configuration

## Router0

```text
interface GigabitEthernet0/0
 ip address 192.168.10.1 255.255.255.0
 ip helper-address 192.168.50.10
 no shutdown
!
interface GigabitEthernet0/1
 ip address 192.168.50.1 255.255.255.0
 no shutdown
```

## Server0

```text
IP Address:       192.168.50.10
Subnet Mask:      255.255.255.0
Default Gateway:  192.168.50.1
DNS Server:       192.168.50.10
```

## DHCP Pool

```text
Pool Name:         Users
Network:           192.168.10.0/24
Default Gateway:   192.168.10.1
DNS Server:        192.168.50.10
Start IP Address:  192.168.10.20
Maximum Users:     100
```

## DNS Record

```text
server.company.local
A Record
192.168.50.10
```

---

# Skills Demonstrated

This project demonstrates practical experience with:

* Cisco Packet Tracer
* Cisco IOS CLI
* IPv4 network design
* IP addressing
* Subnetting
* Router configuration
* Switch troubleshooting
* Centralized DHCP
* DHCP Relay
* DNS configuration
* Static and dynamic addressing
* Default gateways
* Inter-subnet communication
* Client network troubleshooting
* Router interface troubleshooting
* Switch port troubleshooting
* DNS troubleshooting
* DHCP troubleshooting
* Network verification
* Structured troubleshooting methodology

---

# Project Outcome

The final network successfully provides centralized DHCP and DNS services to clients located on a different subnet.

Router0 successfully routes traffic between:

```text
192.168.10.0/24
```

and:

```text
192.168.50.0/24
```

DHCP Relay allows client DHCP requests to reach the centralized Server0 at:

```text
192.168.50.10
```

DNS allows clients to resolve:

```text
server.company.local
```

to:

```text
192.168.50.10
```

Multiple failures were intentionally introduced, diagnosed, and corrected, including:

* Incorrect default gateway
* DHCP Relay failure
* DHCP client configuration failure
* DNS record failure
* Router interface shutdown
* Incorrect client IP configuration
* Switch port shutdown

The final connectivity test confirmed successful DHCP addressing, routing, DNS resolution, and communication between the user and server networks.

---

## Final Result

```text
DHCP                    ✓
DHCP Relay              ✓
DNS                     ✓
Inter-Subnet Routing    ✓
Default Gateway         ✓
Router Interfaces       ✓
Switch Connectivity     ✓
Hostname Resolution     ✓
End-to-End Connectivity ✓
Troubleshooting         ✓
```

**Lab 3 successfully completed.**
