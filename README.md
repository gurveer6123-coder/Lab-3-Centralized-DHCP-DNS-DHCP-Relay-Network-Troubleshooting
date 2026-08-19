# Lab 3 — Centralized DHCP, DNS, DHCP Relay & Network Troubleshooting

## Project Overview

This project demonstrates the design, configuration, verification, and troubleshooting of a small enterprise-style network using **Cisco Packet Tracer**.

The network contains two separate IPv4 subnets connected through a Cisco router:

* A **User LAN** containing three client PCs
* A **Server LAN** containing a centralized server

The centralized server provides:

* DHCP services
* DNS services

Because the DHCP server is located on a different subnet from the client computers, **DHCP Relay** was configured on the router using the Cisco IOS `ip helper-address` command.

The lab also includes multiple intentionally created network failures to practice systematic troubleshooting.

---

# Skills Demonstrated

This lab demonstrates practical experience with:

* Cisco Packet Tracer
* Cisco IOS CLI
* IPv4 addressing
* Subnetting
* Static IP configuration
* Dynamic IP addressing
* DHCP
* Centralized DHCP services
* DHCP Relay
* `ip helper-address`
* DNS
* DNS A records
* Default gateways
* Inter-subnet routing
* Router interface configuration
* Switch port troubleshooting
* DHCP troubleshooting
* DNS troubleshooting
* Client connectivity troubleshooting
* Network verification
* Structured troubleshooting methodology

---

# Network Topology

The network contains:

* 1 Cisco 2911 Router
* 2 Cisco 2960 Switches
* 3 Client PCs
* 1 Server

The topology is:

```text
PC0 -----\
PC1 ------ SW1 -------- Router0 -------- SW2 -------- Server0
PC2 -----/
```

The user devices and server are intentionally placed on different networks.

```text
USER LAN
192.168.10.0/24

PC0
PC1
PC2
 |
SW1
 |
Router0 G0/0
192.168.10.1
 |
 | Routing
 |
Router0 G0/1
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

![Network Topology](https://raw.githubusercontent.com/gurveer6123-coder/lab3Img/main/01-topology.png)

---

# IP Addressing Plan

Two separate `/24` networks were created.

| Device  | Interface     | IP Address    | Subnet Mask   | Purpose            |
| ------- | ------------- | ------------- | ------------- | ------------------ |
| Router0 | G0/0          | 192.168.10.1  | 255.255.255.0 | User LAN Gateway   |
| Router0 | G0/1          | 192.168.50.1  | 255.255.255.0 | Server LAN Gateway |
| Server0 | FastEthernet0 | 192.168.50.10 | 255.255.255.0 | DHCP/DNS Server    |
| PC0     | FastEthernet0 | DHCP          | 255.255.255.0 | Client             |
| PC1     | FastEthernet0 | DHCP          | 255.255.255.0 | Client             |
| PC2     | FastEthernet0 | DHCP          | 255.255.255.0 | Client             |

---

# User Network

The user LAN uses:

```text
Network Address:   192.168.10.0
Subnet Mask:       255.255.255.0
CIDR:              /24
Default Gateway:   192.168.10.1
Broadcast Address: 192.168.10.255
```

The DHCP range begins at:

```text
192.168.10.20
```

This leaves lower addresses available for infrastructure devices.

Example addressing structure:

```text
192.168.10.0      Network Address
192.168.10.1      Router / Default Gateway
192.168.10.2-.19  Reserved for Static Devices
192.168.10.20+    DHCP Client Addresses
192.168.10.255    Broadcast Address
```

---

# Server Network

The server network uses:

```text
Network Address:   192.168.50.0
Subnet Mask:       255.255.255.0
CIDR:              /24
Default Gateway:   192.168.50.1
Server Address:    192.168.50.10
Broadcast Address: 192.168.50.255
```

---

# Router Configuration

Router0 connects both networks.

## Configure G0/0 — User LAN

```text
enable
configure terminal

interface GigabitEthernet0/0
ip address 192.168.10.1 255.255.255.0
no shutdown
exit
```

This interface becomes the default gateway for the client computers.

```text
Router0 G0/0
192.168.10.1/24
```

---

# Configure G0/1 — Server LAN

```text
interface GigabitEthernet0/1
ip address 192.168.50.1 255.255.255.0
no shutdown
exit

end
```

This interface becomes the default gateway for Server0.

```text
Router0 G0/1
192.168.50.1/24
```

---

# Router Interface Verification

The router interfaces were verified using:

```text
show ip interface brief
```

Expected operational interfaces:

```text
GigabitEthernet0/0   192.168.10.1   up   up
GigabitEthernet0/1   192.168.50.1   up   up
```

The `up/up` state confirms that:

* The physical interface is operational
* The Layer 2 protocol is operational

![Router Interface Verification](https://raw.githubusercontent.com/gurveer6123-coder/lab3Img/main/02-router-interfaces.png)

---

# Server Static IP Configuration

Server0 was assigned a static address.

Infrastructure devices such as servers normally use static addresses because clients and other network devices need to know where to find them consistently.

Server0 was configured with:

```text
IP Address:       192.168.50.10
Subnet Mask:      255.255.255.0
Default Gateway:  192.168.50.1
DNS Server:       192.168.50.10
```

The server uses:

```text
192.168.50.1
```

as its default gateway because Router0 G0/1 belongs to the same subnet.

![Server Static IP](https://raw.githubusercontent.com/gurveer6123-coder/lab3Img/main/04-server-static-ip.png)

---

# Server-to-Router Connectivity Test

Connectivity between Server0 and Router0 was verified with:

```text
ping 192.168.50.1
```

Successful replies confirmed that the following path was operational:

```text
Server0
   |
  SW2
   |
Router0 G0/1
```

---

# Centralized DHCP Server

Instead of configuring DHCP directly on Router0, Server0 was configured as a centralized DHCP server.

This demonstrates a common enterprise-style network design.

The DHCP pool was configured as follows:

```text
Pool Name:          Users
Default Gateway:    192.168.10.1
DNS Server:         192.168.50.10
Start IP Address:   192.168.10.20
Subnet Mask:        255.255.255.0
Maximum Users:      100
```

![DHCP Server Pool](https://raw.githubusercontent.com/gurveer6123-coder/lab3Img/main/05-dhcp-pool.png)

The first available DHCP address begins at:

```text
192.168.10.20
```

Possible assignments include:

```text
PC0 → 192.168.10.20
PC1 → 192.168.10.21
PC2 → 192.168.10.22
```

The exact addresses depend on the order in which clients request leases.

---

# Why DHCP Relay Was Required

The DHCP server is located on:

```text
192.168.50.0/24
```

while the client computers are located on:

```text
192.168.10.0/24
```

A DHCP client initially sends a broadcast request.

Example:

```text
DHCPDISCOVER
```

Routers do not normally forward broadcast packets between different IP networks.

Without DHCP Relay:

```text
PC
 |
SW1
 |
Router0
 X
 |
SW2
 |
Server0
```

The DHCP broadcast would stop at Router0.

---

# DHCP Relay Configuration

DHCP Relay was configured using:

```text
ip helper-address
```

The command was placed on Router0 G0/0 because this is the interface where DHCP requests from the client LAN arrive.

Configuration:

```text
enable
configure terminal

interface GigabitEthernet0/0
ip helper-address 192.168.50.10

end
```

The resulting interface configuration is:

```text
interface GigabitEthernet0/0
 ip address 192.168.10.1 255.255.255.0
 ip helper-address 192.168.50.10
```

![DHCP Relay Configuration](https://raw.githubusercontent.com/gurveer6123-coder/lab3Img/main/03-dhcp-relay.png)

The DHCP process becomes:

```text
PC
 |
 | DHCP Broadcast
 v
SW1
 |
 v
Router0 G0/0
 |
 | ip helper-address
 v
Server0
192.168.50.10
```

Router0 relays the DHCP request to the centralized server.

---

# DNS Configuration

Server0 was also configured as the DNS server.

The DNS service was enabled and an IPv4 A record was created.

```text
Name:     server.company.local
Type:     A Record
Address:  192.168.50.10
```

![DNS A Record](https://raw.githubusercontent.com/gurveer6123-coder/lab3Img/main/06-dns-record.png)

DNS allows clients to use:

```text
server.company.local
```

instead of remembering:

```text
192.168.50.10
```

The resolution process is:

```text
server.company.local
        |
        v
DNS Server
        |
        v
192.168.50.10
```

---

# DHCP Client Verification

The PCs were configured to receive their IPv4 configuration automatically through DHCP.

Client configuration was checked using:

```text
ipconfig
```

or:

```text
ipconfig /all
```

A client successfully received:

```text
IPv4 Address:     192.168.10.21
Subnet Mask:      255.255.255.0
Default Gateway:  192.168.10.1
```

![PC DHCP Address](https://raw.githubusercontent.com/gurveer6123-coder/lab3Img/main/07-pc-dhcp-address.png)

This confirms that:

* Server0 DHCP is operational
* DHCP Relay is operational
* Router connectivity is operational
* The client received valid network configuration

---

# End-to-End Connectivity Test

The server was tested by hostname using:

```text
ping server.company.local
```

DNS successfully resolved:

```text
server.company.local
```

to:

```text
192.168.50.10
```

The server responded successfully.

![DNS Connectivity Test](https://raw.githubusercontent.com/gurveer6123-coder/lab3Img/main/08-connectivity-dns-test.png)

This single test helps verify:

```text
Client IP Address        ✓
Subnet Mask              ✓
Default Gateway          ✓
Switch Connectivity      ✓
Router Connectivity      ✓
Inter-Subnet Routing     ✓
DNS Resolution           ✓
Server Connectivity      ✓
```

---

# Network Troubleshooting

After confirming that the network worked correctly, several faults were intentionally introduced.

The purpose was to practice identifying problems using symptoms and troubleshooting commands rather than randomly changing configurations.

---

# Troubleshooting Scenario 1 — Incorrect Default Gateway

PC0 was intentionally configured with an incorrect default gateway.

Incorrect example:

```text
IP Address:       192.168.10.50
Subnet Mask:      255.255.255.0
Default Gateway:  192.168.10.100
DNS Server:       192.168.50.10
```

The correct gateway was:

```text
192.168.10.1
```

A host may still communicate with devices on its own subnet while being unable to communicate with remote networks.

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

The issue was resolved by restoring:

```text
Default Gateway: 192.168.10.1
```

The client was later returned to DHCP.

---

# Troubleshooting Scenario 2 — DHCP Relay Failure

The DHCP Relay configuration was intentionally removed from Router0.

Command:

```text
enable
configure terminal

interface GigabitEthernet0/0
no ip helper-address 192.168.50.10

end
```

The client was then forced to request a new DHCP lease.

Because the DHCP server was on another subnet and the relay configuration was missing, the client failed to receive a valid IPv4 address.

The result was:

```text
IPv4 Address:     0.0.0.0
Subnet Mask:      0.0.0.0
Default Gateway:  0.0.0.0
```

![DHCP Failure](https://raw.githubusercontent.com/gurveer6123-coder/lab3Img/main/09-dhcp-failure.png)

In real Windows environments, a DHCP failure may also cause a client to assign itself an APIPA address from:

```text
169.254.0.0/16
```

A `169.254.x.x` address is therefore an important troubleshooting clue for DHCP problems.

---

# Diagnosing the DHCP Failure

Client configuration can first be checked using:

```text
ipconfig /all
```

Router interfaces can then be checked with:

```text
show ip interface brief
```

If both router interfaces remain:

```text
up/up
```

the next step is checking the router configuration:

```text
show running-config
```

The missing command would be identified:

```text
ip helper-address 192.168.50.10
```

---

# Fixing DHCP Relay

The configuration was restored with:

```text
configure terminal

interface GigabitEthernet0/0
ip helper-address 192.168.50.10

end
```

The client was then configured for DHCP again and successfully received a valid:

```text
192.168.10.x
```

address.

---

# Troubleshooting Scenario 3 — DNS Failure

The DNS A record for:

```text
server.company.local
```

was intentionally removed from Server0.

The server could still be reached using:

```text
ping 192.168.50.10
```

but hostname resolution failed.

This demonstrated an important troubleshooting method.

```text
Ping by IP works
        |
Ping by hostname fails
        |
        v
Investigate DNS
```

The DNS record was restored:

```text
server.company.local
A Record
192.168.50.10
```

After restoring the record:

```text
ping server.company.local
```

worked successfully again.

---

# Troubleshooting Scenario 4 — Router Interface Failure

Router0 G0/1 was intentionally shut down.

Configuration:

```text
enable
configure terminal

interface GigabitEthernet0/1
shutdown
```

The interface status was checked using:

```text
show ip interface brief
```

The result showed:

```text
GigabitEthernet0/1
192.168.50.1
administratively down
down
```

![Router Interface Failure](https://raw.githubusercontent.com/gurveer6123-coder/lab3Img/main/10-router-interface-failure.png)

The phrase:

```text
administratively down
```

indicates that the interface has been manually disabled using the Cisco `shutdown` command.

---

# Fixing the Router Interface

The interface was restored using:

```text
configure terminal

interface GigabitEthernet0/1
no shutdown

end
```

Verification:

```text
show ip interface brief
```

The interface returned to:

```text
GigabitEthernet0/1   192.168.50.1   up   up
```

---

# Understanding Router Interface States

The `show ip interface brief` command can provide useful troubleshooting information.

### up / up

```text
up   up
```

Usually indicates a healthy interface.

### administratively down / down

```text
administratively down   down
```

Usually indicates the interface has been manually shut down.

Solution:

```text
no shutdown
```

### down / down

```text
down   down
```

Can indicate:

* Cable problem
* Remote device powered off
* Remote interface down
* Physical connection issue

---

# Troubleshooting Scenario 5 — Incorrect Client IP Address

A client was intentionally configured with an address from the wrong subnet.

Incorrect configuration:

```text
IP Address:       192.168.20.50
Subnet Mask:      255.255.255.0
Default Gateway:  192.168.10.1
```

The PC belongs to:

```text
192.168.20.0/24
```

while the configured gateway belongs to:

```text
192.168.10.0/24
```

Therefore, they are not in the same subnet.

The client configuration was checked using:

```text
ipconfig /all
```

The IP address and gateway were compared to identify the problem.

The client was eventually returned to:

```text
DHCP
```

so Server0 could automatically restore the correct settings.

---

# Troubleshooting Scenario 6 — Switch Port Failure

The switch port connected to PC0 was intentionally disabled.

Switch configuration:

```text
enable
configure terminal

interface FastEthernet0/1
shutdown

end
```

The switch port status was checked using:

```text
show interfaces status
```

The result showed:

```text
Fa0/1     disabled
```

while the other switch interfaces remained connected.

![Switch Port Failure](https://raw.githubusercontent.com/gurveer6123-coder/lab3Img/main/11-switch-port-failure.png)

This isolated the problem specifically to the switch port connected to PC0.

---

# Detailed Switch Interface Troubleshooting

A specific switch port can be examined using:

```text
show interfaces FastEthernet0/1
```

A manually disabled interface may show:

```text
FastEthernet0/1 is administratively down,
line protocol is down
```

The port was restored using:

```text
configure terminal

interface FastEthernet0/1
no shutdown

end
```

The status was checked again using:

```text
show interfaces status
```

The port returned to:

```text
connected
```

---

# Final Network Verification

After all troubleshooting exercises were completed, all intentionally created failures were repaired.

Client configuration was checked using:

```text
ipconfig
```

The client received:

```text
IPv4 Address:     192.168.10.21
Subnet Mask:      255.255.255.0
Default Gateway:  192.168.10.1
```

The default gateway was tested using:

```text
ping 192.168.10.1
```

The server was tested directly using:

```text
ping 192.168.50.10
```

DNS and end-to-end connectivity were verified using:

```text
ping server.company.local
```

The DNS server successfully resolved:

```text
server.company.local
```

to:

```text
192.168.50.10
```

and all packets were returned successfully.

![Final Network Verification](https://raw.githubusercontent.com/gurveer6123-coder/lab3Img/main/12-final-verification.png)

The final test confirmed:

```text
Packets Sent     = 4
Packets Received = 4
Packets Lost     = 0

Packet Loss      = 0%
```

---

# Important Client Troubleshooting Commands

## Check Basic IP Configuration

```text
ipconfig
```

Used to check:

* IPv4 address
* Subnet mask
* Default gateway

---

## Check Detailed IP Configuration

```text
ipconfig /all
```

Used to check:

* IPv4 address
* Subnet mask
* Default gateway
* DNS server
* DHCP information

---

# Ping the Default Gateway

```text
ping 192.168.10.1
```

This tests:

```text
PC → SW1 → Router0
```

If this fails, investigate:

* Client IP address
* Subnet mask
* Default gateway
* Ethernet connection
* Switch port
* Router G0/0

---

# Ping Server by IP

```text
ping 192.168.50.10
```

This tests:

```text
PC
 |
SW1
 |
Router0
 |
SW2
 |
Server0
```

If the gateway works but this fails, investigate:

* Router server-side interface
* Server network
* Server configuration
* Routing

---

# Test DNS

```text
ping server.company.local
```

If:

```text
ping 192.168.50.10
```

works but:

```text
ping server.company.local
```

fails, investigate DNS.

---

# Important Router Troubleshooting Commands

## Show IP Interface Status

```text
show ip interface brief
```

Useful for identifying:

```text
up/up

down/down

administratively down/down
```

---

# Show Router Configuration

```text
show running-config
```

Useful for verifying:

* Interface IP addresses
* Subnet masks
* DHCP Relay
* Shutdown configuration
* Other router settings

---

# Important Switch Troubleshooting Commands

## Show Switch Port Status

```text
show interfaces status
```

Useful for identifying:

```text
connected
notconnect
disabled
```

---

## Show Specific Interface Information

```text
show interfaces FastEthernet0/1
```

Useful for checking:

* Interface state
* Line protocol
* Physical connectivity
* Administrative shutdown state

---

## Show VLAN Configuration

```text
show vlan brief
```

Useful for checking:

* VLAN membership
* Active VLANs
* Port assignments

---

# Troubleshooting Methodology

A major goal of this lab was learning to troubleshoot systematically rather than randomly changing network configuration.

The following process was used:

```text
1. Check physical connection
        |
        v
2. Check switch port
        |
        v
3. Check client IP address
        |
        v
4. Check subnet mask
        |
        v
5. Check default gateway
        |
        v
6. Ping local gateway
        |
        v
7. Ping remote destination by IP
        |
        v
8. Test DNS
        |
        v
9. Check DHCP/DNS services
        |
        v
10. Check router configuration
```

---

# Common Symptoms and Possible Causes

| Symptom                                      | Possible Cause                            |
| -------------------------------------------- | ----------------------------------------- |
| `0.0.0.0` address                            | DHCP failure                              |
| `169.254.x.x` address                        | DHCP/APIPA                                |
| Cannot reach gateway                         | Local LAN, IP, switch, or gateway problem |
| Local network works but remote network fails | Gateway or routing problem                |
| Server works by IP but not hostname          | DNS problem                               |
| `administratively down`                      | Interface manually shut down              |
| Switch port shows `disabled`                 | Port shutdown                             |
| Wrong subnet address                         | Client IP configuration                   |
| Remote DHCP server unreachable               | DHCP Relay problem                        |
| `ip helper-address` missing                  | DHCP cannot cross router                  |

---

# DHCP vs Router DHCP

A router can act as a DHCP server.

For example:

```text
ip dhcp pool USERS
network 192.168.10.0 255.255.255.0
default-router 192.168.10.1
```

However, this lab used a dedicated centralized server.

```text
Server0 = DHCP Server
Router0 = DHCP Relay
```

A centralized DHCP server is commonly used in larger environments where administrators want DHCP services centrally managed.

---

# DHCP Information Provided to Clients

DHCP can automatically provide:

```text
IP Address
Subnet Mask
Default Gateway
DNS Server
```

For example:

```text
IP Address:       192.168.10.21
Subnet Mask:      255.255.255.0
Default Gateway:  192.168.10.1
DNS Server:       192.168.50.10
```

---

# DNS Concept

DNS translates names into IP addresses.

Example:

```text
server.company.local
        |
        v
192.168.50.10
```

Without DNS, users would need to remember the IP address of every server.

---

# Default Gateway Concept

A default gateway is used when a device needs to communicate with another network.

For the client network:

```text
Default Gateway = 192.168.10.1
```

For the server network:

```text
Default Gateway = 192.168.50.1
```

Router0 connects the two networks.

```text
192.168.10.0/24
       |
192.168.10.1
   Router0
192.168.50.1
       |
192.168.50.0/24
```

---

# Static vs Dynamic IP Addresses

Infrastructure devices were configured using static addresses.

```text
Router0 G0/0 = 192.168.10.1

Router0 G0/1 = 192.168.50.1

Server0 = 192.168.50.10
```

End-user PCs were configured dynamically using DHCP.

This helps keep infrastructure addresses predictable while simplifying client management.

---

# Final Router Configuration

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

---

# Final Server Configuration

```text
IP Address:       192.168.50.10
Subnet Mask:      255.255.255.0
Default Gateway:  192.168.50.1
DNS Server:       192.168.50.10
```

---

# Final DHCP Pool

```text
Pool Name:          Users
Network:            192.168.10.0/24
Default Gateway:    192.168.10.1
DNS Server:         192.168.50.10
Start IP Address:   192.168.10.20
Subnet Mask:        255.255.255.0
Maximum Users:      100
```

---

# Final DNS Record

```text
Name:     server.company.local
Type:     A Record
Address:  192.168.50.10
```

---

# Final Working Network

```text
PC0/PC1/PC2
DHCP Clients
192.168.10.0/24

       |
       v

Switch0

       |
       v

Router0 G0/0
192.168.10.1

       |
       | Routing
       |

Router0 G0/1
192.168.50.1

       |
       v

Switch1

       |
       v

Server0
192.168.50.10
DHCP + DNS
```

---

# Lab Results

The final network successfully demonstrated:

```text
IPv4 Addressing            ✓
Subnetting                 ✓
Static Addressing          ✓
Dynamic Addressing         ✓
Centralized DHCP           ✓
DHCP Relay                 ✓
DNS                        ✓
Default Gateways           ✓
Inter-Subnet Routing       ✓
Router Configuration       ✓
Switch Troubleshooting     ✓
DHCP Troubleshooting       ✓
DNS Troubleshooting        ✓
Interface Troubleshooting  ✓
End-to-End Connectivity    ✓
```

---

# Troubleshooting Scenarios Completed

The following failures were intentionally introduced, diagnosed, and repaired:

1. Incorrect default gateway
2. DHCP Relay failure
3. DHCP client addressing failure
4. DNS record failure
5. Router interface shutdown
6. Incorrect client IP configuration
7. Switch port shutdown

---

# Project Outcome

This lab successfully demonstrated how centralized network services can support clients located on different subnets.

Server0 provided DHCP and DNS services from:

```text
192.168.50.10
```

while the client PCs were located on:

```text
192.168.10.0/24
```

Router0 provided connectivity between:

```text
192.168.10.0/24
```

and:

```text
192.168.50.0/24
```

The command:

```text
ip helper-address 192.168.50.10
```

allowed DHCP requests from the user LAN to reach the DHCP server located on another subnet.

DNS successfully resolved:

```text
server.company.local
```

to:

```text
192.168.50.10
```

The network was tested under multiple failure conditions, and each problem was identified and corrected using Cisco IOS and client troubleshooting commands.

---

# Final Verification

The final network successfully achieved:

```text
DHCP                     ✓
DHCP Relay               ✓
DNS                      ✓
Inter-Subnet Routing     ✓
Default Gateway          ✓
Router Interfaces        ✓
Switch Connectivity      ✓
Hostname Resolution      ✓
Client Connectivity      ✓
Server Connectivity      ✓
End-to-End Connectivity  ✓
Troubleshooting          ✓
```

## Lab 3 Successfully Completed
