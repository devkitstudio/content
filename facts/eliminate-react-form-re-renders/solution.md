# Controlled vs. Uncontrolled Architecture

Eliminate re-renders by leveraging uncontrolled components and refs.

Managing massive forms (20+ fields, dynamic arrays, deep validation) in React traditionally leads to severe performance degradation. Every keystroke triggers a state update, causing the entire form tree to re-render.

React Hook Form (RHF) solves this by abandoning the standard "Controlled Component" pattern in favor of an **Uncontrolled Architecture**.

### The Anti-Pattern: Controlled Forms (useState / Formik)

In traditional React forms, input values are bound to React state.

```text
Type "A" -> React State Updates -> Entire Form Re-renders -> Input displays "A"
```

Typing 10 characters in a 50-field Formik form triggers 500 field re-renders.

### The Standard: Uncontrolled Forms (React Hook Form)

RHF isolates the input using standard HTML `ref` attributes. The data lives inside the DOM node, not in React's render cycle.

```text
Type "A" -> DOM updates natively (No React state change) -> ZERO Re-renders
```

Validation and submission extract the values directly from the DOM refs on-demand. RHF only triggers re-renders when absolutely necessary (e.g., showing an error message).
