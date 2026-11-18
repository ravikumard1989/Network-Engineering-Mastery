# Address Resolution Protocol (ARP)

**NEM 2026 | Module 01 — Networking Fundamentals & Packet-Level Engineering**

| Attribute | Details |
|---|---|
| Topic | Address Resolution Protocol |
| OSI Layer | Layer 2 / Layer 3 interaction |
| Protocol | ARP |
| Ethernet EtherType | `0x0806` |
| RFC | RFC 826 |
| Learning status | Theory and packet structure |
| Practical status | Lab validation pending |

---

# Chapter 1 — Why ARP Exists

## 1.1 The Story Before ARP

Consider two computers connected to the same Ethernet switch.

```text
                  +-------------------+
                  |      SWITCH       |
                  |                   |
                  +-------------------+
                       |         |
                       |         |
                   +-------+  +-------+
                   | PC-A  |  | PC-B  |
                   +-------+  +-------+
```

**PC-A**

- IP Address: `192.168.1.10`
- MAC Address: `AA:AA:AA:AA:AA:AA`

**PC-B**

- IP Address: `192.168.1.20`
- MAC Address: `BB:BB:BB:BB:BB:BB`

## 1.2 Understanding the Problem

PC-A wants to send a file to PC-B.

PC-A knows:

```text
Destination IP = 192.168.1.20
```

However, PC-A does not initially know PC-B's MAC address.

Why does this matter?

An Ethernet switch forwards frames using destination MAC addresses. It maintains a MAC address table (CAM table) to identify which switch port leads to a particular MAC address.

Example:

| MAC Address | Switch Port |
|---|---|
| AA:AA:AA:AA:AA:AA | Port 1 |
| BB:BB:BB:BB:BB:BB | Port 2 |

The switch does not translate IP addresses into MAC addresses. That resolution happens at the endpoint using ARP.

## 1.3 The Fundamental Problem

There is a difference between the information available at Layer 3 and what is needed to construct an Ethernet frame at Layer 2.

PC-A knows:

```text
192.168.1.20
```

But it needs:

```text
BB:BB:BB:BB:BB:BB
```

The protocol that performs this IPv4-to-MAC address resolution is **Address Resolution Protocol (ARP)**.

```text
       IPv4 Address
       192.168.1.20
             |
             |
             v
       +-----------+
       |    ARP    |
       +-----------+
             |
             |
             v
        MAC Address
    BB:BB:BB:BB:BB:BB
```

## 1.4 Why Doesn't Ethernet Just Use IP Addresses?

Ethernet and IP have different responsibilities.

### Ethernet — Layer 2

Responsibilities:

- Local frame delivery across an Ethernet segment.
- MAC addressing.
- Frame forwarding within a Layer 2 broadcast domain.
- Hardware-based switching using MAC address tables.

### IP — Layer 3

Responsibilities:

- Logical addressing.
- Communication across multiple IP networks.
- Routing decisions.
- End-to-end packet delivery across routed networks.

This separation allows Layer 2 technologies and Layer 3 routing to work together across different network environments.

## 1.5 Real-World Analogy

Imagine an apartment complex.

- IP Address = Apartment Number (Apartment 1204).
- MAC Address = Identity of the local resident or endpoint.

A courier knows the apartment number but needs the local delivery system to identify the correct recipient.

ARP performs a similar resolution on an Ethernet LAN: it maps a known IPv4 address to the MAC address needed for local frame delivery.

### Key Takeaways

- An IP address identifies a logical network endpoint.
- A MAC address identifies a Layer 2 interface on an Ethernet network.
- Traditional Ethernet switches learn MAC addresses from incoming frames.
- ARP resolves IPv4 addresses into MAC addresses.
- Cloud providers may use centralized control planes and virtualized forwarding instead of conventional physical Layer 2 switching.
- Guest operating systems can still use Ethernet and ARP even when the underlying cloud infrastructure abstracts the physical network.

---

# Chapter 2 — ARP Packet-Level Deep Dive

## 2.1 Network Scenario

Consider PC-A communicating with PC-B.

| Parameter | PC-A | PC-B |
|---|---|---|
| IP Address | 10.1.1.10 | 10.1.1.20 |
| MAC Address | AA:AA:AA:AA:AA:AA | BB:BB:BB:BB:BB:BB |
| Subnet | 10.1.1.0/24 | 10.1.1.0/24 |

PC-A wants to communicate with `10.1.1.20`.

Both devices belong to the same subnet.

If PC-A does not already have a valid ARP cache entry for PC-B, it must resolve the destination MAC address before transmitting the Ethernet unicast data frame.

## 2.2 ARP Request

PC-A broadcasts an ARP request to discover the MAC address associated with `10.1.1.20`.

The request effectively asks:

**"Who has 10.1.1.20? Tell 10.1.1.10."**

### Ethernet Header

| Field | Value |
|---|---|
| Destination MAC | FF:FF:FF:FF:FF:FF |
| Source MAC | AA:AA:AA:AA:AA:AA |
| EtherType | 0x0806 |

### ARP Packet

| Field | Value |
|---|---|
| Hardware Type (HTYPE) | 1 |
| Protocol Type (PTYPE) | 0x0800 |
| Hardware Length (HLEN) | 6 |
| Protocol Length (PLEN) | 4 |
| Operation (OPER) | 1 — Request |
| Sender Hardware Address (SHA) | AA:AA:AA:AA:AA:AA |
| Sender Protocol Address (SPA) | 10.1.1.10 |
| Target Hardware Address (THA) | 00:00:00:00:00:00 |
| Target Protocol Address (TPA) | 10.1.1.20 |

### Why These Ethernet Header Values?

**Destination MAC: FF:FF:FF:FF:FF:FF**

PC-A does not know the destination MAC address. Therefore, it uses the Ethernet broadcast address.

The switch floods this frame within the relevant VLAN, excluding the ingress port.

**Source MAC: AA:AA:AA:AA:AA:AA**

This is PC-A's own MAC address.

**EtherType: 0x0806**

Identifies the Ethernet payload as an ARP packet.

### Common Ethernet EtherType Values

| EtherType | Protocol |
|---|---|
| 0x0800 | IPv4 |
| 0x0806 | ARP |
| 0x86DD | IPv6 |

Note: IPv6 uses Neighbor Discovery Protocol (NDP), not ARP.

## 2.3 Understanding ARP Header Fields

| Field | Size | Description |
|---|---:|---|
| HTYPE | 16 bits | Hardware type; 1 represents Ethernet |
| PTYPE | 16 bits | Protocol type; 0x0800 represents IPv4 |
| HLEN | 8 bits | Hardware address length; 6 bytes for Ethernet MAC |
| PLEN | 8 bits | Protocol address length; 4 bytes for IPv4 |
| OPER | 16 bits | Operation code; 1 = Request, 2 = Reply |
| SHA | 48 bits | Sender Hardware Address |
| SPA | 32 bits | Sender Protocol Address |
| THA | 48 bits | Target Hardware Address |
| TPA | 32 bits | Target Protocol Address |

### Important Field Explanations

**HTYPE — Hardware Type**

Identifies the hardware network technology.

```text
HTYPE = 1
```

Meaning: Ethernet.

**PTYPE — Protocol Type**

Identifies the protocol being mapped.

```text
PTYPE = 0x0800
```

Meaning: IPv4.

**HLEN — Hardware Length**

Specifies the length of the hardware address.

```text
HLEN = 6
```

An Ethernet MAC address contains 6 bytes.

**PLEN — Protocol Length**

Specifies the length of the protocol address.

```text
PLEN = 4
```

An IPv4 address contains 4 bytes.

**OPER — Operation**

Identifies the ARP message type.

```text
1 = ARP Request
2 = ARP Reply
```

**SHA — Sender Hardware Address**

The MAC address of the sender.

**SPA — Sender Protocol Address**

The IPv4 address of the sender.

**THA — Target Hardware Address**

The MAC address of the target.

**TPA — Target Protocol Address**

The IPv4 address whose MAC address is being resolved.

For a normal ARP request, THA is typically all zeros because the target MAC is not yet known.

---

# Chapter 3 — ARP Reply

## 3.1 How PC-B Responds

PC-B receives the broadcast ARP request.

It recognizes that the target protocol address matches its own IP address:

```text
Target IP = 10.1.1.20
```

PC-B responds with its MAC address.

The reply effectively communicates:

**"10.1.1.20 is at BB:BB:BB:BB:BB:BB."**

## 3.2 ARP Reply Packet Structure

### Ethernet Header

| Field | Value |
|---|---|
| Destination MAC | AA:AA:AA:AA:AA:AA |
| Source MAC | BB:BB:BB:BB:BB:BB |
| EtherType | 0x0806 |

### ARP Packet

| Field | Value |
|---|---|
| Hardware Type (HTYPE) | 1 |
| Protocol Type (PTYPE) | 0x0800 |
| Hardware Length (HLEN) | 6 |
| Protocol Length (PLEN) | 4 |
| Operation (OPER) | 2 — Reply |
| Sender Hardware Address (SHA) | BB:BB:BB:BB:BB:BB |
| Sender Protocol Address (SPA) | 10.1.1.20 |
| Target Hardware Address (THA) | AA:AA:AA:AA:AA:AA |
| Target Protocol Address (TPA) | 10.1.1.10 |

**Important:** The sender fields in the ARP reply contain PC-B's addresses, while the target fields contain PC-A's addresses.

## 3.3 Why Is the ARP Reply Unicast?

PC-B already knows PC-A's MAC address from the received ARP request.

Therefore, PC-B can send the reply directly to PC-A.

```text
Destination MAC = AA:AA:AA:AA:AA:AA
Source MAC      = BB:BB:BB:BB:BB:BB
```

The reply is normally sent as a unicast Ethernet frame.

## 3.4 Complete ARP Request and Reply Flow

```text
          PC-A                         SWITCH                       PC-B
     10.1.1.10                       Ethernet                    10.1.1.20
     MAC: AA:AA                                                   MAC: BB:BB
         |                               |                            |
         |                               |                            |
         |------ ARP REQUEST ----------->|--------------------------->|
         |       Broadcast               |                            |
         |                               |                            |
         |                               |                            |
         |<----- ARP REPLY --------------|<---------------------------|
         |       Unicast                 |                            |
         |                               |                            |
         |                               |                            |
         |------ DATA FRAME ------------>|--------------------------->|
         |       Unicast                 |                            |
         |                               |                            |
```

After receiving the ARP reply, PC-A can update its ARP cache:

| IPv4 Address | MAC Address |
|---|---|
| 10.1.1.20 | BB:BB:BB:BB:BB:BB |

PC-A can now send Ethernet frames directly to PC-B using the resolved MAC address.

---

# Chapter 4 — ARP Packet Size and Encapsulation

For Ethernet and IPv4, the ARP message has the following size:

| Field | Size |
|---|---:|
| HTYPE | 2 bytes |
| PTYPE | 2 bytes |
| HLEN | 1 byte |
| PLEN | 1 byte |
| OPER | 2 bytes |
| SHA | 6 bytes |
| SPA | 4 bytes |
| THA | 6 bytes |
| TPA | 4 bytes |
| **Total** | **28 bytes** |

Ethernet II header:

| Field | Size |
|---|---:|
| Destination MAC | 6 bytes |
| Source MAC | 6 bytes |
| EtherType | 2 bytes |
| **Total** | **14 bytes** |

Minimum Ethernet frame calculation:

```text
Ethernet Header       = 14 bytes
ARP Payload           = 28 bytes
Ethernet Padding      = 18 bytes
FCS                   =  4 bytes
--------------------------------
Total                 = 64 bytes
```

The minimum Ethernet frame size is 64 bytes, including the FCS.

Some packet captures do not include the FCS, so the captured frame may appear as 60 bytes.

---

# Chapter 5 — Important ARP Concepts to Explore

The following topics will be covered in the next stages of ARP learning:

- ARP cache and neighbor table.
- Gratuitous ARP.
- Proxy ARP.
- ARP aging and refresh behavior.
- Duplicate IP detection.
- ARP spoofing and ARP poisoning.
- Dynamic ARP Inspection (DAI).
- DHCP snooping.
- ARP behavior in virtualized and cloud networks.
- ARP suppression in EVPN/VXLAN.

---

# Chapter 6 — Hands-on Lab and Packet Analysis

**Status: Pending**

Lab objectives:

1. Inspect the Linux interface and IP configuration.
2. Inspect the neighbor table.
3. Generate an ARP request.
4. Capture the ARP request and reply.
5. Decode Ethernet and ARP headers.
6. Verify the sender and target address fields.
7. Compare the actual packet capture with the documented packet structure.

Planned Linux commands:

```bash
ip addr show
ip route show
ip neigh show

sudo tcpdump -ni eth0 -e -vv arp
```

Replace `eth0` with the actual interface name.

### Wireshark Display Filters

All ARP packets:

```text
arp
```

ARP requests:

```text
arp.opcode == 1
```

ARP replies:

```text
arp.opcode == 2
```

### Lab Evidence to Add

- [ ] Linux interface output.
- [ ] ARP neighbor table output.
- [ ] ARP request PCAP.
- [ ] ARP reply PCAP.
- [ ] Wireshark packet decode screenshot.
- [ ] Explanation of observed packet fields.

---

# Chapter 7 — References

1. RFC 826 — An Ethernet Address Resolution Protocol.
2. RFC 5227 — IPv4 Address Conflict Detection.
3. RFC 1122 — Requirements for Internet Hosts.
4. IEEE 802.3 — Ethernet Standard.
5. Linux `ip-neighbour(8)` manual.

---

# Learning Completion Checklist

- [ ] Explain why ARP exists.
- [ ] Explain the difference between Layer 2 and Layer 3 addressing.
- [ ] Explain same-subnet ARP resolution.
- [ ] Understand Ethernet headers and EtherType values.
- [ ] Understand every ARP header field.
- [ ] Explain ARP request and reply behavior.
- [ ] Decode ARP packet fields.
- [ ] Understand Ethernet frame size and padding.
- [ ] Complete the hands-on Linux lab.
- [ ] Analyze ARP packets in Wireshark.
- [ ] Understand advanced ARP concepts.
- [ ] Complete troubleshooting and interview validation.

**NEM 2026 — Learn deeply. Engineer practically. Design at scale.**
