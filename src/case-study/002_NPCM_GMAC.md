# NPCM GMAC: Length Truncation Before Heap Use

## Summary

This sample is a narrowing bug in the transmit path. The guest can build a descriptor chain whose total packet size exceeds the width of the allocator-side length variable. The allocation length becomes truncated, but later copy operations still use the larger logical size. The result is a heap buffer overflow.

## Trigger Chain

1. The guest prepares many transmit descriptors.
2. The emulator accumulates the total payload size across the chain.
3. A narrowed length is used for allocation or resize.
4. The DMA copy path still uses the original larger amount of data.
5. The copied payload runs past the end of the heap buffer.

## Vulnerable Logic

```c
uint32_t total_len = 0;

for each descriptor {
    total_len += desc_len;
}

uint16_t alloc_len = total_len;
buf = realloc(buf, alloc_len);

for each descriptor {
    dma_read(desc_addr, buf + copied, desc_len);
    copied += desc_len;
}
```

## Fixed Logic

```c
uint64_t total_len = 0;

for each descriptor {
    total_len += desc_len;
    if (total_len > MAX_PACKET_SIZE) {
        fail_packet();
        return;
    }
}

buf = realloc(buf, total_len);

for each descriptor {
    dma_read(desc_addr, buf + copied, desc_len);
    copied += desc_len;
}
```

## Analysis

The root problem is inconsistent width handling. One path reasons about the packet with a wide integer, while another path silently narrows the same quantity before allocating memory. That mismatch is exactly why this bug is dangerous: the arithmetic error happens earlier, but the memory corruption appears later during DMA reads.
