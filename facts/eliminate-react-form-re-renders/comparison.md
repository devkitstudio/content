## State Management Trade-offs

When choosing a form management strategy, runtime performance (re-renders) and architectural overhead are the primary constraints.

| Feature                       | React Hook Form                             | Formik                 | Native `useReducer`               |
| :---------------------------- | :------------------------------------------ | :--------------------- | :-------------------------------- |
| **Architecture**              | Uncontrolled (Refs)                         | Controlled (State)     | Controlled (State)                |
| **Re-renders (on keystroke)** | **~0 (Isolated)**                           | High (Entire form)     | Manual / Custom optimization      |
| **Footprint Overhead**        | Minimal (Highly optimized for tree-shaking) | Moderate               | Zero (Native React API)           |
| **3rd-Party UI Integration**  | Requires `<Controller>` wrapper             | Native / Easy          | Manual wiring                     |
| **Best Used For**             | Massive, performance-critical forms         | Legacy enterprise apps | Learning, trivial localized forms |

### Decision Tree

```text
Are you integrating heavily with strict UI component libraries (Material UI, AntD)?
├─ YES → Use React Hook Form, but you MUST use the <Controller> API.
└─ NO → Is the form structurally complex (dynamic arrays, deep validation, high-frequency inputs)?
    ├─ YES → Use React Hook Form (Standard ref registration).
    └─ NO → Native React useState is sufficient. Do not add 3rd-party library overhead.
```
