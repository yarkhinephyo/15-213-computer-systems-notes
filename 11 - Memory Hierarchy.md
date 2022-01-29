
**Volatile memories**

| .              | SRAM  | DRAM        |
| -------------- | ----- | ----------- |
| Trans. per-bit | 6     | 1           |
| Access time    | 1x    | 10x         |
| Needs refresh  | No    | Yes         |
| Cost           | 100x  | 1x          |
| Applications   | Cache | Main memory |
| Volatile       | Yes   | Yes         |

<ins>SRAM</ins>: Each bit is stored in a bistable memory cell that can stay in either of two voltage configurations as long as it is powered.

<ins>DRAM</ins>: Each bit is stored as charge on a capacitor. Memory system periodically refreshes every bit of memory by reading and rewriting it.

Each DRAM chip has _d_ supercells, each consisting of _w_ DRAM cells. In the example below, the chip has 16 supercells and 8 bits per supercell. When the memory controller sends row address strobe (RAS), the DRAM copies the supercells in row 2 onto an internal buffer. When the memory controller sends column address strobe (CAS), the DRAM copies the 8 bits inside the specific supercell (now in the internal buffer) and sends them to the memory controller.

![](images/Pasted%20image%2020220430212406.png)

**Nonvolatile memories**

<ins>ROM</ins>: Read-only memory.

<ins>PROM</ins>: Can be programmed exactly once.

<ins>EPROM</ins>: Can be bulk cleared to zero and rewritten. Requires a separate device to write _ones_.

<ins>EEPROM</ins>: Can be bulk cleared to zero and rewritten in-place without a separate device.

<ins>Flash memory</ins>: Based on EEPROMs. Partial erase capability which wears out over time.

**Firmware**: Programs stored in ROMs such as BIOS, disk controllers.

**Bus Interface**: Collection of parallel wires that carry address, data and control signals. The control wires carry signals that synchronize the transaction and identify the type being performed.

**CPU read transaction**

1. Command `movq A, %rax`.
2. CPU places address A on the memory bus.
3. Main memory reads address A from the memory bus and writes its data on the bus.
4. CPU reads the data and places it in `%rax`.

To be precise, the CPU's bus interface places the address A on the system bus and the I/O bridge forwards it to the memory bus.

![](images/Pasted%20image%2020220430223957.png)

**CPU write transaction**

1. Command `movq %rax, A`.
2. CPU places address A on the memory bus.
3. Main memory reads it and waits for data to arrive.
4. CPU places the data on the bus.
5. Main memory reads the data and stores it at address A.

**Bus is expensive**: Memory reads and writes are about 50-100 ns while operations between registers are < 1 ns.

**Rotating disks**: Consist of one or more platters, each with two surfaces. Each surface consists of concentric rings called tracks. Each track consists of sectors separated by gaps.

![](images/Pasted%20image%2020220430225112.png)

**Disk capacity**: Number of bits for storage.

![](images/Pasted%20image%2020220430225719.png)

**Disk access**: Seek -> Rotational Latency -> Data Transfer. Seek time dominates the disk access time. After the first byte, the remaining bytes are essentially free.

![](images/Pasted%20image%2020211220150721.png)

```
Given rotational rate = 7200 RPM
Average seek time = 9 ms
Average #sectors / track = 400

Tavg rotation = 1/2 x 60 /(7200 RPM) x 1000 ms/sec = 4 ms
Tavg transfer
= 60 /(7200 RPM) x 1/(400 sectors/track) x 1000 ms/sec = 0.02 ms
Taccess = 9ms + 4 ms + 0.02 ms
```

**Disk controller**: Keeps mapping of logical disk blocks (1 ... B-1) to physical sectors. Sets aside spare cylinders for each zone.

**Read a disk sector**

1. CPU writes a command + logical block number + destination memory address to a port associated with the disk controller.
2. Firmware on the disk controller performs a fast table lookup.
3. Hardware on the disk controller reads the sector and transfer into main memory.
4. Once data transfer completes, the disk controller notifies the CPU with an interrupt.

**I/O devices**: I/O bridge connects with system bus, memory bus and I/O bus. <ins>Unlike</ins> system bus and memory bus, I/O bus is designed to be CPU independent. It can accomodate a variety of third party I/O devices.

**Solid state disk (SSD)**: Storage technology based on flash memory. Flash translation layer serves the same purpose as the disk controller.

Data is written in units of pages but a page can be only written after its block has been erased. In other words, to modify a single page, other pages have to be copied onto a clean block. A block wears after about 100,000 erases. Consequently, reading is performed much faster than writing.

<ins>Performance</ins>: Average seq read time of 50 us and average seq write time of 60 us. Random writes are slower because erasing a block takes a long time (~1ms)

![](images/Pasted%20image%2020211220152933.png)

**Principle of locality**: Programs tend to use data and instructions with addresses near or equal to those used recently.

<ins>Temporal locality</ins>: Recently referenced items will be referenced again.

<ins>Spatial locality</ins>: Items with nearby addresses tend to be referenced close together in time.

**Memory hierachy**

1. Registers
2. L1 (SRAM)
3. L2 (SRAM)
4. L3 (SRAM)
5. Main memory (DRAM)
6. Local secondary storage (Disk)
7. Remote secondary storage (Servers)

**Types of cache misses**

<ins>Cold miss</ins>: Cache is initially empty.

<ins>Capacity miss</ins>: Occurs when active cache blocks (working set) is larger than the cache.

<ins>Conflict miss</ins>: Most caches limit blocks at level-(k+1) to a small subset of positions at level k. This is because randomly placed blocks are expensive to locate. However, this means that multiple data can map to the same level-k block and evicting one another.