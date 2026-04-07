# Debounce

Debouncing is a programming practice used to enforce that a function does not execute too often. When an event fires, it starts a timer. If the event fires again before the timer expires, the timer resets. The function only executes once the timer completes.

## Why is it important?

In frontend engineering, we frequently deal with rapid consecutive events, such as:
- Scrolling (window scroll events)
- Resizing the browser window
- Typing in a search box (fetching autocomplete suggestions)

Without debouncing, handling every single keystroke could crash the browser thread or overwhelm the backend API with unnecessary queries.
