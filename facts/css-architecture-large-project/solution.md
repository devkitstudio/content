## Architectural Trade-offs: The CSS Paradigm Shift

Choosing a CSS architecture fundamentally dictates your application's rendering performance (Compile-time vs. Runtime) and your team's context-switching overhead.

| Paradigm                          | Scoping Mechanism                  | Performance Profile                                                                  | Footprint / Bundle Size                                                   | Optimal Use Case                                       |
| :-------------------------------- | :--------------------------------- | :----------------------------------------------------------------------------------- | :------------------------------------------------------------------------ | :----------------------------------------------------- |
| **Utility-First (Tailwind)**      | Implicit (Inline classes)          | **Zero Runtime Overhead.** Styles are compiled ahead-of-time (AOT).                  | Minimal (Aggressive tree-shaking purges unused CSS).                      | Large teams, component-heavy apps, rapid iteration.    |
| **CSS Modules**                   | Explicit (Unique generated hashes) | **Zero Runtime Overhead.** Standard CSS evaluation by the browser engine.            | Moderate (Scales linearly with custom CSS written).                       | Legacy migrations, teams with deep CSS expertise.      |
| **CSS-in-JS (Styled Components)** | Explicit (Unique generated hashes) | **Runtime Penalty.** JavaScript must execute to inject `<style>` tags during render. | Heavy (Requires shipping a styling engine parsing library to the client). | Highly dynamic theming controlled by complex JS state. |

## The Decision Matrix

Do not choose an architecture based on preference; choose it based on engineering constraints.

```text
Does your application require highly dynamic, JS-driven state to calculate styles?
├─ YES → Use CSS-in-JS (Styled Components / Emotion). Accept the runtime performance hit.
└─ NO → Do you have an established, strict design system and want zero runtime overhead?
    ├─ YES → Use Tailwind CSS. (Enforces consistency, minimizes CSS bundle).
    └─ NO → Use CSS Modules. (Retains standard CSS syntax but guarantees local scoping).
```
