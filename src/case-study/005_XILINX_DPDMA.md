# Xilinx DPDMA: Descriptor-Driven Heap Overflow

## Summary

This sample is driven by inconsistent descriptor geometry. The guest can provide descriptor fields whose transfer layout does not match the size of the destination buffer. The DMA engine keeps advancing a write pointer according to descriptor-derived chunk sizes, but it does not stop when the logical transfer exceeds the allocated output region. The result is a heap overflow.

## Trigger Chain

1. The guest prepares a descriptor with an inconsistent transfer layout.
2. The DMA engine derives chunk sizes from line size, stride, or fragment geometry.
3. The internal write pointer keeps moving forward after each DMA read.
4. The implementation trusts descriptor-derived chunk sizes more than the actual destination capacity.
5. A later chunk lands just past the end of the heap buffer.

## Vulnerable Logic

```c
size_t ptr = 0;

while (transfer_len > 0) {
    size_t chunk = next_chunk_from_descriptor(desc);
    dma_read(src_addr, dst_buf + ptr, chunk);
    ptr += chunk;
    transfer_len -= chunk;
}
```

## Suggested Fix Logic

```c
size_t ptr = 0;

while (transfer_len > 0) {
    size_t chunk = next_chunk_from_descriptor(desc);
    size_t remaining = dst_buf_size - ptr;

    if (chunk == 0 || chunk > transfer_len || chunk > remaining) {
        report_descriptor_error();
        stop_channel();
        return;
    }

    dma_read(src_addr, dst_buf + ptr, chunk);
    ptr += chunk;
    transfer_len -= chunk;
}
```

## Analysis

This is a descriptor-validation failure. The DMA engine has enough information to compute how much data it plans to import, but it does not enforce that plan against the real heap buffer boundary. Once the descriptor geometry and the destination capacity diverge, pointer arithmetic walks off the end of the buffer. This sample still needs a strict bounds check before every chunk transfer.
