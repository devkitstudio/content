## State Management Trade-offs

When choosing a form management strategy, bundle size and rendering performance are the primary constraints.

| Feature                       | React Hook Form                     | Formik                 | Native `useReducer`           |
| :---------------------------- | :---------------------------------- | :--------------------- | :---------------------------- |
| **Architecture**              | Uncontrolled (Refs)                 | Controlled (State)     | Controlled (State)            |
| **Re-renders (on keystroke)** | **~0 (Isolated)**                   | High (Entire form)     | Custom / Manual               |
| **Bundle Size (Minified)**    | ~9KB                                | ~13KB                  | 0KB (Built-in)                |
| **3rd-Party UI Integration**  | Requires `<Controller>` wrapper     | Native / Easy          | Manual wiring                 |
| **Best Used For**             | Massive, performance-critical forms | Legacy enterprise apps | Learning, single simple forms |

### Decision Tree

```text
Are you integrating heavily with strict UI component libraries (Material UI, AntD)?
├─ YES → Use React Hook Form, but you must use the <Controller> API.
└─ NO → Are performance and re-renders a concern (Form has > 5 fields)?
    ├─ YES → Use React Hook Form (Standard ref registration).
    └─ NO → Native React useState is sufficient. Do not install a library.
```
