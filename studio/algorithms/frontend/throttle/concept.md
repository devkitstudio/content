# Throttle

Throttling is a technique used in event coordination to guarantee that a function is executed **at most once** within a specified period of time. Unlike Debounce (which delays execution until events stop firing), Throttle ensures a steady, fixed rate of execution.

## Common Use Cases

1. **Scroll Events:** Checking scroll position (e.g., to trigger an infinite load or parallax effect) without overloading the browser's render pipeline.
2. **Resize Events:** Recalculating complex UI layouts when a user smoothly shrinks or expands a window.
3. **Continuous Actions:** Drag-and-drop operations, slider adjustments, or rapid mouse movement tracking.

## Anatomy of Throttle

- **Delay/Wait:** The fixed interval of time (e.g., 300ms).
- **Execution:** When the first event fires, it's executed immediately (leading edge). Subsequent events fired within the 300ms window are ignored. 
- **Trailing Edge (Optional):** Many standard libraries (like Lodash) allow executing the function on the trailing edge if the user continued firing events during the cooldown phase.

## Visual Difference: Throttle vs Debounce

- **Throttle:** Imagine a club bouncer letting exactly one person in every 10 seconds, regardless of how large the crowd pushing the door is.
- **Debounce:** Imagine elevator doors trying to close. Every time someone tries to step in, the timer to close the door resets. The doors ONLY close when no one has walked in for a few seconds.
