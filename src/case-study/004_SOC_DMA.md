# SoC DMA: Fast-Path Length Overflow

## Summary

This sample is a fast-path size computation bug. The guest controls DMA counters that are later combined into a byte count for a bulk memory copy. When the arithmetic is performed in an insufficiently safe integer type, the computed length can overflow and become negative or otherwise invalid. The bad value is then passed into the fast `memcpy` path.

## Trigger Chain

1. The guest programs very large element and frame counts.
2. The DMA engine computes a bulk transfer length from those counters.
3. The arithmetic overflows in the intermediate length variable.
4. The fast path reuses the corrupted length as the `memcpy` size.
5. The host crashes or performs an invalid memory operation.

## Vulnerable Logic

```c
int elems = guest_elements;
int frames = guest_frames;
int bytes = elems * frames * element_size;

memcpy(dst, src, bytes);
```

## Fixed Logic

```c
uint64_t total = (uint64_t)guest_elements * guest_frames * element_size;

if (total == 0 || total > MAX_DMA_BYTES || total > SIZE_MAX) {
    report_guest_error();
    abort_fast_path();
    return;
}

memcpy(dst, src, (size_t)total);
```

## Analysis

The vulnerability is not in `memcpy()` itself. The real failure is earlier: the fast path assumes its computed byte count is trustworthy. Once the count overflows, a normal bulk copy API becomes the sink for a corrupted arithmetic result. This is exactly the sort of bug that is easy to reach but hard to detect with plain coverage guidance.
