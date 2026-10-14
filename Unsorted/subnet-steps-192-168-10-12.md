# Steps to Find the Subnet Details of an Address

The steps below are written once, then applied to the address 192.168.10.12.

---

## The steps

1. Write the address and the prefix.
2. Convert the prefix to a mask.
3. Find the interesting byte (the one that is not 255 or 0).
4. Work out the block size: 256 minus the interesting byte.
5. Find which block the address falls in.
6. Read off the results:
   - Network address: the start of the block
   - Broadcast address: the end of the block
   - Usable hosts: everything in between
7. Count the usable hosts: 2^(host bits) minus 2.

---

## Worked example: 192.168.10.12/28

**Step 1: Write the address and prefix**

```
192.168.10.12/28
```

**Step 2: Convert the prefix to a mask**

```
/28 = 28 ones = 11111111.11111111.11111111.11110000
Last byte: 11110000 = 128 + 64 + 32 + 16 = 240
Mask: 255.255.255.240
```

**Step 3: Find the interesting byte**

```
255.255.255.240  ->  the last byte, 240
```

**Step 4: Block size**

```
256 - 240 = 16
```

Blocks start at multiples of 16:

```
.0   .16   .32   .48   .64 ...
```

**Step 5: Find the block**

```
The last byte of the address is 12.
0 <= 12 < 16  ->  the block starts at 0
```

**Step 6: Read off the results**

```
Network address:    192.168.10.0
First usable host:  192.168.10.1
Last usable host:   192.168.10.14
Broadcast address:  192.168.10.15
```

**Step 7: Count usable hosts**

```
Host bits    = 32 - 28 = 4
Addresses    = 2^4 = 16
Usable hosts = 16 - 2 = 14
```

---

## Same address, different prefixes

Only steps 2 to 5 change. Here is the same address with other masks.

### 192.168.10.12/24

```
Mask:        255.255.255.0
Block size:  the whole last byte (256)
Network:     192.168.10.0
Broadcast:   192.168.10.255
Usable:      .1 to .254   (254 hosts)
```

### 192.168.10.12/26

```
Mask:        255.255.255.192
Block size:  256 - 192 = 64
Block:       0 to 63   (12 is in here)
Network:     192.168.10.0
Broadcast:   192.168.10.63
Usable:      .1 to .62   (62 hosts)
```

### 192.168.10.12/29

```
Mask:        255.255.255.248
Block size:  256 - 248 = 8
Block:       8 to 15   (12 is in here)
Network:     192.168.10.8
Broadcast:   192.168.10.15
Usable:      .9 to .14   (6 hosts)
```

### 192.168.10.12/30

```
Mask:        255.255.255.252
Block size:  256 - 252 = 4
Block:       12 to 15   (12 is the start)
Network:     192.168.10.12
Broadcast:   192.168.10.15
Usable:      .13 to .14   (2 hosts)
```

Here 192.168.10.12 is the network address itself, so it cannot be given to a device.

---

## Summary table

| Prefix | Mask            | Block size | Network       | Broadcast      | Usable hosts   |
|--------|-----------------|------------|---------------|----------------|----------------|
| /24    | 255.255.255.0   | 256        | 192.168.10.0  | 192.168.10.255 | .1 to .254     |
| /26    | 255.255.255.192 | 64         | 192.168.10.0  | 192.168.10.63  | .1 to .62      |
| /28    | 255.255.255.240 | 16         | 192.168.10.0  | 192.168.10.15  | .1 to .14      |
| /29    | 255.255.255.248 | 8          | 192.168.10.8  | 192.168.10.15  | .9 to .14      |
| /30    | 255.255.255.252 | 4          | 192.168.10.12 | 192.168.10.15  | .13 to .14     |
