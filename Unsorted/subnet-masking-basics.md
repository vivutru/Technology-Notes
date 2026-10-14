# Subnet Masking Basics

A plain-language guide to what subnet masks are and how to read them.

## 1. The big idea

An IP address has two parts:

- The network part: which "street" the device lives on.
- The host part: which "house" on that street.

The subnet mask is the rule that says where the street name ends and the house number begins.

```
IP address:    192.168.1.130
                \_____/ \_/
              network   host
               part     part

Subnet mask:   255.255.255.0
                \_____/ \_/
              "network"  "host"
               bytes     byte
```

A 255 in the mask means "this byte is network." A 0 means "this byte is host."

## 2. The binary view

Computers read the mask in binary. Every 1 marks a network bit. Every 0 marks a host bit.

```
255.255.255.0 in binary:

  11111111 . 11111111 . 11111111 . 00000000
  \_____________________________/   \______/
          24 network bits           8 host bits
```

The 1s are always on the left, and the 0s are always on the right. They never mix.

## 3. CIDR notation (the slash)

Instead of writing the whole mask, you can just count the 1s and write that number after a slash.

```
255.255.255.0    =  /24    (24 ones)
255.255.0.0      =  /16    (16 ones)
255.0.0.0        =  /8     (8 ones)
255.255.255.192  =  /26    (26 ones)
```

So 192.168.1.0/24 means "the network 192.168.1.0 with a 24-bit mask."

## 4. How many devices fit?

The host bits decide the size of the network.

```
Host bits = 8   ->   2^8 = 256 addresses
                     minus 2 reserved = 254 usable
```

Two addresses in every subnet are reserved:

- The first address is the network address (the name of the subnet itself).
- The last address is the broadcast address (used to talk to everyone on the subnet).

```
192.168.1.0/24

  192.168.1.0      192.168.1.1  ...  192.168.1.254      192.168.1.255
  +-----------+    +-------------------------------+    +-------------+
  |  Network  |    |     Usable host addresses     |    |  Broadcast  |
  |  address  |    |                               |    |   address   |
  +-----------+    +-------------------------------+    +-------------+
```

Formula: usable hosts = 2^(host bits) - 2

## 5. Splitting a network (subnetting)

If you "borrow" bits from the host side and turn them into network bits, you cut one big network into smaller ones.

Starting with 192.168.1.0/24 and moving to /26 (borrowing 2 bits):

```
/24:  11111111.11111111.11111111.00000000     1 network, 256 addresses
/26:  11111111.11111111.11111111.11000000     4 networks, 64 addresses each
                                  ^^
                            2 borrowed bits
                            2^2 = 4 subnets
```

The four resulting subnets:

```
192.168.1.0/26      range .0   to .63
192.168.1.64/26     range .64  to .127
192.168.1.128/26    range .128 to .191
192.168.1.192/26    range .192 to .255
```

Visual layout of the original /24 block:

```
.0                                                         .255
|-------------|-------------|--------------|-------------|
|   Subnet 1  |   Subnet 2  |   Subnet 3   |   Subnet 4  |
|   .0-.63    |  .64-.127   |  .128-.191   |  .192-.255  |
|-------------|-------------|--------------|-------------|
     64             64             64             64
```

Each subnet has 64 addresses and 62 usable hosts.

## 6. How a device uses the mask

To find out which network an address belongs to, the device does a bitwise AND of the address and the mask. Where both bits are 1, the result is 1. Otherwise it is 0.

Example: 192.168.1.130 with mask 255.255.255.192 (/26)

```
IP address:   11000000.10101000.00000001.10000010   (192.168.1.130)
Mask:         11111111.11111111.11111111.11000000   (255.255.255.192)
              ----------------------------------
AND result:   11000000.10101000.00000001.10000000   (192.168.1.128)
```

So 192.168.1.130 lives on the network 192.168.1.128/26.

- Network address: 192.168.1.128
- Usable hosts: 192.168.1.129 to 192.168.1.190
- Broadcast address: 192.168.1.191

## 7. Same network or not?

Two devices can talk directly only if they land on the same network after the AND step. Otherwise traffic must go through a router.

```
Device A: 192.168.1.10/26   -> network 192.168.1.0
Device B: 192.168.1.50/26   -> network 192.168.1.0     same network, direct

Device A: 192.168.1.10/26   -> network 192.168.1.0
Device C: 192.168.1.130/26  -> network 192.168.1.128   different, needs a router

   [A]---[B]                [C]
     Subnet 1      [Router]    Subnet 3
```

## 8. Cheat sheet

| CIDR | Subnet mask     | Total addresses | Usable hosts |
|------|-----------------|-----------------|--------------|
| /8   | 255.0.0.0       | 16,777,216      | 16,777,214   |
| /16  | 255.255.0.0     | 65,536          | 65,534       |
| /24  | 255.255.255.0   | 256             | 254          |
| /25  | 255.255.255.128 | 128             | 126          |
| /26  | 255.255.255.192 | 64              | 62           |
| /27  | 255.255.255.224 | 32              | 30           |
| /28  | 255.255.255.240 | 16              | 14           |
| /29  | 255.255.255.248 | 8               | 6            |
| /30  | 255.255.255.252 | 4               | 2            |

## 9. Quick method for any subnet

1. Write the prefix, for example /27.
2. Host bits = 32 minus the prefix, so 32 - 27 = 5.
3. Block size = 2 to the power of host bits, so 2^5 = 32.
4. Subnets start at multiples of the block size: .0, .32, .64, .96, and so on.
5. Usable hosts = block size minus 2, so 30.

## 10. Key takeaways

- The mask separates the network part from the host part.
- 1s mean network, 0s mean host.
- The slash number is just the count of 1s.
- More host bits means a bigger subnet. More network bits means more, smaller subnets.
- Every subnet loses two addresses to the network and broadcast roles.
- Devices compare addresses with the mask (AND) to decide if they share a network.
