# Subnet Masking: One Step at a Time

Each example below is isolated and introduces one new step. You can read them in order or jump to the one you need.

---

## Example 1: Read a mask

**New step: tell which part is network and which is host.**

```
Address:  192.168.1.50
Mask:     255.255.255.0
```

- 255 means the byte is network.
- 0 means the byte is host.

```
192.168.1  = network
50         = host
```

Result: the device is host 50 on the network 192.168.1.0.

---

## Example 2: Convert a mask to a slash number

**New step: count the 1 bits.**

```
Mask:    255.255.255.240
Binary:  11111111.11111111.11111111.11110000
```

Count the 1s: 8 + 8 + 8 + 4 = 28

Result: 255.255.255.240 is /28.

---

## Example 3: Convert a slash number to a mask

**New step: fill in 1s from the left, then turn each group of 8 into a number.**

```
Prefix: /26
```

```
26 ones:  11111111.11111111.11111111.11000000
```

The last byte is 11000000 = 128 + 64 = 192

Result: /26 is 255.255.255.192.

---

## Example 4: Count how many hosts fit

**New step: use the host bits.**

```
Prefix: /28
```

```
Host bits    = 32 - 28 = 4
Addresses    = 2^4 = 16
Usable hosts = 16 - 2 = 14
```

The 2 removed are the network address and the broadcast address.

Result: a /28 holds 14 usable hosts.

---

## Example 5: Find the block size from a mask

**New step: subtract the interesting byte from 256.**

```
Mask: 255.255.255.224
```

The last byte is the one that is neither 255 nor 0.

```
Block size = 256 - 224 = 32
```

Subnets in this size start at multiples of 32:

```
.0   .32   .64   .96   .128   .160   .192   .224
```

Result: each subnet has 32 addresses.

---

## Example 6: Find the network and broadcast address

**New step: place an address inside its block.**

```
Address: 10.0.5.77/27
```

Step 1: a /27 is mask 255.255.255.224, so the block size is 32.

Step 2: find which block 77 falls in.

```
Blocks:  0, 32, 64, 96 ...
64 <= 77 < 96   ->   the block starts at 64
```

Step 3: write the results.

```
Network address:    10.0.5.64
First usable host:  10.0.5.65
Last usable host:   10.0.5.94
Broadcast address:  10.0.5.95
```

---

## Example 7: Prove it with binary AND

**New step: see what the device actually does.**

Using the same address as Example 6, look at only the last byte.

```
Address last byte:  77   = 01001101
Mask last byte:     224  = 11100000
                           --------
AND result:                01000000  = 64
```

AND rule: the result is 1 only where both bits are 1.

Result: the network address ends in 64, which matches Example 6.

---

## Example 8: Check if two devices share a network

**New step: compare network addresses.**

```
Device A: 192.168.1.10/26
Device B: 192.168.1.50/26
```

Block size for /26 is 64.

```
A: 10 is in the block 0-63   ->  network 192.168.1.0
B: 50 is in the block 0-63   ->  network 192.168.1.0
```

Same network, so they talk directly.

Now compare with a third device:

```
Device C: 192.168.1.130/26   ->  130 is in the block 128-191  ->  network 192.168.1.128
```

A and C are on different networks, so a router is needed between them.

---

## Example 9: Work with a mask that changes the third byte

**New step: apply the block size to a different byte.**

```
Address: 10.1.37.9/20
```

A /20 mask is 255.255.240.0. The interesting byte is the third one.

```
Block size = 256 - 240 = 16
37 falls in the block starting at 32   (32, 48, 64 ...)
```

Results:

```
Network address:    10.1.32.0
Broadcast address:  10.1.47.255
Usable range:       10.1.32.1 to 10.1.47.254
```

The broadcast is 47.255 because every host bit is turned on: the end of the third-byte block (47) and all of the fourth byte (255).

---

## Example 10: Check two addresses across a wide block

**New step: compare using the block range, not just one byte.**

```
Device A: 172.16.4.20/22
Device B: 172.16.7.200/22
```

A /22 mask is 255.255.252.0, so the block size in the third byte is 4.

```
Third byte blocks: 0, 4, 8, 12 ...
A: 4 is in the block 4-7   ->  network 172.16.4.0
B: 7 is in the block 4-7   ->  network 172.16.4.0
```

Same network, even though the third bytes (4 and 7) look different.

Range of this network: 172.16.4.0 to 172.16.7.255.

---

## Example 11: Choose a prefix for a host requirement

**New step: work backwards from the number of devices.**

Requirement: a network that needs 50 hosts.

```
Try 5 host bits:  2^5 = 32  ->  30 usable   too small
Try 6 host bits:  2^6 = 64  ->  62 usable   fits
```

```
Prefix = 32 - 6 = /26
```

Result: use a /26 for 50 hosts.

Another case, needing 100 hosts:

```
7 host bits:  2^7 = 128  ->  126 usable   fits
Prefix = 32 - 7 = /25
```

---

## Example 12: Split one network into equal subnets

**New step: borrow host bits to make more networks.**

Start: 192.168.10.0/24. Goal: 4 equal subnets.

```
Need 4 subnets  ->  2^2 = 4  ->  borrow 2 bits
New prefix = 24 + 2 = /26
```

Block size for /26 is 64, so the four subnets are:

```
192.168.10.0/26     hosts .1   to .62      broadcast .63
192.168.10.64/26    hosts .65  to .126     broadcast .127
192.168.10.128/26   hosts .129 to .190     broadcast .191
192.168.10.192/26   hosts .193 to .254     broadcast .255
```

---

## Summary of the steps

| Step | What you learn                              |
|------|---------------------------------------------|
| 1    | Read network versus host from a mask        |
| 2    | Mask to slash number                        |
| 3    | Slash number to mask                        |
| 4    | Count usable hosts                          |
| 5    | Find the block size                         |
| 6    | Find network and broadcast addresses        |
| 7    | See the binary AND                          |
| 8    | Check if two devices share a network        |
| 9    | Handle masks that change the third byte     |
| 10   | Compare addresses across a wide block       |
| 11   | Pick a prefix for a host requirement        |
| 12   | Split a network into equal subnets          |
