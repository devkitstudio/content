## The Traps of CSS Architecture

### 1. The `@apply` Anti-Pattern (Tailwind)

Many developers try to "clean up" their HTML by extracting Tailwind utilities into custom CSS classes using `@apply` (e.g., `.btn { @apply bg-blue-500 text-white; }`).

**The Flaw:** This fundamentally breaks the utility-first paradigm. It re-introduces naming fatigue, forces you to jump between CSS and JS files (context switching), and balloons your CSS bundle size because the compiled CSS is no longer deduplicated.
**The Fix:** Use component abstractions (React/Vue/Svelte components) to encapsulate styling logic, not CSS classes.

### 2. The Frankenstein Stack (The Hybrid Trap)

Attempting to use Tailwind for layout, CSS Modules for components, and Styled Components for dynamic states in the same repository.

**The Flaw:** This is an architectural failure. You are forcing the client to download multiple rendering engines and massive CSS bundles. You force developers to learn three different styling paradigms. Refactoring becomes nearly impossible.
**The Fix:** Choose **one** primary paradigm. If you choose Tailwind, use inline styles (`style={{ color: dynamicValue }}`) for the rare dynamic edges, rather than installing a full CSS-in-JS library as an "escape hatch".

### 3. Global Scope Bleeding

Using plain Global CSS (e.g., standard `.css` or `.scss` files without modules) in a modern Component-Based Architecture.

**The Flaw:** Without CSS Modules or Utility classes, class names will eventually collide (e.g., two developers styling `.card-header` differently in separate files). This leads to reliance on `!important` tags and unpredictable cascading UI bugs.
