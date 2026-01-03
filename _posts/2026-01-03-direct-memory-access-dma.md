---
layout: post
title: Direct Memory Access (DMA)
date: 2026-01-03 12:08 +0100
categories:
  - Emulation
tags:
  - technical
  - component
  - hardware
  - internals
---
## DMA In Computers
In a computer DMA is a feature that allows certain subsystems to directly access the system memory without the CPU.
In a DMA-less configuration, when memory is transferred from one location to another, the CPU handles it one word at a time. This fully occupies the CPU which cannot do some other meaningful calculations in the meantime.

Having Direct Memory Access (DMA) means that the CPU will trigger/initiate a memory transfer, say between the disk drive and the RAM to load an image. After triggering the transfer, the DMA starts loading the data into memory while the CPU can perform some calculations in parallel using its instruction cache.

The DMA Controller is the hardware chip that handles the the direct memory access transfers when receiving an interrupt or a signal to start a transfer.

There are 2 main types of DMA:
- [Third-party DMA](https://en.wikipedia.org/wiki/Direct_memory_access#:~:text=Standard%20DMA%2C%20also,in%20one%20burst.) (standard DMA)
- [First-party DMA](https://en.wikipedia.org/wiki/Direct_memory_access#:~:text=In%20a%20bus,does%20not%20occur.) (bus mastering DMA)

## DMA On The PS1
Since the Playstation 1 uses a third-party DMA, we will not cover the first-party DMA

The Playstation 1, uses the DMA to ensure transfers of data to/from the main system RAM to/from a bunch of subsystems:
- GPU (2 channels)
- SPU
- CDROM
- MDEC (2 channels)

The DMA has 3 different modes of operation :
- Burst mode (transfer data all at once after a Data Request signal is received)
- Slice mode (split data into blocks, transfer next block whenever the Data Request signal is received)
- Linked List mode

### DMA Registers
The DMA Controller (DMAC) has 7 channels for hardware peripherals and IO (GPU, SPU, CDROM, etc.)
A [DMA channel](https://psx-spx.consoledev.net/dmachannels/) contains the necessary data for a transfer to eventually take place. It also contains the information about the mode of transfer and destination/direction of transfer.

There are 2 main control registers on the DMAC:
- Control Register -> Each nibble (4bits) represents a channel (enabled and priority)
- Interrupt Register

Each channel has 3 registers:
- DMA base address
- DMA block control
- DMA channel control

### Linked List Mode
In [linked list mode](https://psx-spx.consoledev.net/dmachannels/#linked-list-dma), the data is transferred as packets/nodes from a linked list. Each packet contains a header-word:

```
0-23 Address of the next node (or end marker)
24-31 Number of extra words to transfer for this node
```

Example diagram of a linked list:
```
   Adrress
  0xABCD1234        0x12340000          0xnnnnnnnn
+------------+    +------------+    +----------------+
| 0x12340003 |    | 0xABABFFF2 |    |  More packets  |
| 0xDEADBEEF |--->| 0xABABABAB |--->| until end tag  |
| 0x12345678 |    | 0x1A2B3C4D |    |    0xFFFFFF    |
| 0xFFFFBBBB |    +------------+    |   is reached   |
+------------+                      +----------------+
Next packet is
at 0x12340000
and contains 3
data words to
transfer
```

The linked list mode is mostly used to transfer draw-calls and configuration data to the GPU. The linked list acts as a "z-buffer", first drawing the farthest polygons.

## References
1. [https://psx-spx.consoledev.net/dmachannels/](https://en.wikipedia.org/wiki/Direct_memory_access)
2. [https://en.wikipedia.org/wiki/Direct_memory_access](https://en.wikipedia.org/wiki/Direct_memory_access)