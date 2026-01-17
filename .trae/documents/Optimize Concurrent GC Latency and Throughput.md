I will optimize the concurrent GC from three key dimensions to resolve the latency issues:

1.  **Eliminate Main Thread Waiting (Latency)**:
    *   **Current Behavior**: When the heap is full, the main thread waits for the background GC to finish before allocating, causing frame drops.
    *   **Optimization**: Allow the allocator to "borrow" more memory from the OS if a concurrent GC is already running. This bypasses the wait loop entirely.
    *   **Implementation**: Modify `GlobalAllocator::allowMoreBlocks` to return `true` when `gConcurrentState` is active, even in Generational mode. I will also move the `ConcurrentGCState` definition to the top of the file so it's visible to the allocator.

2.  **Relax Fallback Threshold (Stability)**:
    *   **Current Behavior**: If memory usage exceeds 85%, the GC panic-switches to a blocking Full GC.
    *   **Optimization**: Increase this threshold to **92%**. This gives the concurrent GC more time to finish its job without triggering a stop-the-world fallback.

3.  **Early Trigger (Throughput)**:
    *   By allowing borrowing, we effectively let the application run faster while GC cleans up in the background, improving overall throughput.

**Actionable Steps**:
1.  Move `ConcurrentGCState` and related globals to the top of `Immix.cpp`.
2.  Update `allowMoreBlocks` to check `gConcurrentState`.
3.  Update `Collect` to use `0.92` as the fallback threshold.