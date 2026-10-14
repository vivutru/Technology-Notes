# Class A, B and C Subnet Examples

Each class has a default mask. Each example below finds the default network first, then subnets it further.

---

## Class overview

| Class | First byte range | Default mask  | Prefix | Usable hosts per network |
|-------|------------------|---------------|--------|--------------------------|
| A     | 1 to 126         | 255.0.0.0     | /8     | 16,777,214               |
| B     | 128 to 191       | 255.255.0.0   | /16    | 65,534                   |
| C     | 192 to 223       | 255.255.255.0 | /24    | 254                      |

The first byte of the address tells you the class. 127 is reserved for loopback.

Classes are the old system. Modern networks use CIDR (any prefix length), but the defaults are still a useful starting point.

---

## Class A example: 10.20.30.40

**Default mask: 255.0.0.0 (/8)**

```
Only the first byte is network.
Network address:    10.0.0.0
First usable host:  10.0.0.1
Last usable host:   10.255.255.254
Broadcast address:  10.255.255.255
Usable hosts:       2^24 - 2 = 16,777,214
```

### Subnetted to /16 (255.255.0.0)

```
Block size in the second byte: 256 (whole byte)
Network address:    10.20.0.0
Broadcast address:  10.20.255.255
Usable range:       10.20.0.1 to 10.20.255.254
Usable hosts:       2^16 - 2 = 65,534
```

### Subnetted to /20 (255.255.240.0)

```
Interesting byte:   the third byte, 240
Block size:         256 - 240 = 16
Third byte is 30:   30 falls in the block 16 to 31
Network address:    10.20.16.0
Broadcast address:  10.20.31.255
Usable range:       10.20.16.1 to 10.20.31.254
Usable hosts:       2^12 - 2 = 4,094
```

---

## Class B example: 172.16.50.100

**Default mask: 255.255.0.0 (/16)**

```
The first two bytes are network.
Network address:    172.16.0.0
First usable host:  172.16.0.1
Last usable host:   172.16.255.254
Broadcast address:  172.16.255.255
Usable hosts:       2^16 - 2 = 65,534
```

### Subnetted to /24 (255.255.255.0)

```
Network address:    172.16.50.0
Broadcast address:  172.16.50.255
Usable range:       172.16.50.1 to 172.16.50.254
Usable hosts:       254
```

### Subnetted to /20 (255.255.240.0)

```
Interesting byte:   the third byte, 240
Block size:         256 - 240 = 16
Third byte is 50:   50 falls in the block 48 to 63
Network address:    172.16.48.0
Broadcast address:  172.16.63.255
Usable range:       172.16.48.1 to 172.16.63.254
Usable hosts:       2^12 - 2 = 4,094
```

---

## Class C example: 192.168.1.77

**Default mask: 255.255.255.0 (/24)**

```
The first three bytes are network.
Network address:    192.168.1.0
First usable host:  192.168.1.1
Last usable host:   192.168.1.254
Broadcast address:  192.168.1.255
Usable hosts:       254
```

### Subnetted to /26 (255.255.255.192)

```
Block size:         256 - 192 = 64
Last byte is 77:    77 falls in the block 64 to 127
Network address:    192.168.1.64
Broadcast address:  192.168.1.127
Usable range:       192.168.1.65 to 192.168.1.126
Usable hosts:       62
```

### Subnetted to /28 (255.255.255.240)

```
Block size:         256 - 240 = 16
Last byte is 77:    77 falls in the block 64 to 79
Network address:    192.168.1.64
Broadcast address:  192.168.1.79
Usable range:       192.168.1.65 to 192.168.1.78
Usable hosts:       14
```

---

## Side by side

| Address       | Class | Prefix | Network       | Broadcast        | Usable hosts |
|---------------|-------|--------|---------------|------------------|--------------|
| 10.20.30.40   | A     | /8     | 10.0.0.0      | 10.255.255.255   | 16,777,214   |
| 10.20.30.40   | A     | /16    | 10.20.0.0     | 10.20.255.255    | 65,534       |
| 10.20.30.40   | A     | /20    | 10.20.16.0    | 10.20.31.255     | 4,094        |
| 172.16.50.100 | B     | /16    | 172.16.0.0    | 172.16.255.255   | 65,534       |
| 172.16.50.100 | B     | /24    | 172.16.50.0   | 172.16.50.255    | 254          |
| 172.16.50.100 | B     | /20    | 172.16.48.0   | 172.16.63.255    | 4,094        |
| 192.168.1.77  | C     | /24    | 192.168.1.0   | 192.168.1.255    | 254          |
| 192.168.1.77  | C     | /26    | 192.168.1.64  | 192.168.1.127    | 62           |
| 192.168.1.77  | C     | /28    | 192.168.1.64  | 192.168.1.79     | 14           |

---

## Pattern to remember

- The class gives the default mask.
- A longer prefix than the default means you are subnetting.
- A shorter prefix than the default means you are combining networks (supernetting).
- The block-size method works the same in every class. Only the byte you apply it to changes.
