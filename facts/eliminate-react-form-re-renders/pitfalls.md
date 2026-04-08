## The Traps of React Hook Form

If you are migrating from Formik or Redux-Form, your muscle memory will cause you to break RHF's performance benefits.

### 1. The Global `watch()` Trap

If you need to conditionally render a field based on another field's value, do NOT use the root-level `watch()` function carelessly.

```tsx
// ❌ FATAL ANTI-PATTERN: Destroys performance
const { watch } = useForm();
const emailValue = watch("email"); // Re-renders the ENTIRE form on every keystroke in 'email'
```

**Fix:** Use the `useWatch` hook to isolate the re-render strictly to the component that needs the value, keeping the rest of the form untouched.

### 2. The 3rd-Party UI Library Trap

You cannot use standard `register()` on complex UI library components (like Material-UI `<Select>` or Ant Design `<DatePicker>`) because they often do not expose the native `ref` correctly.

**Fix:** You must wrap them in the RHF `<Controller />` component, which acts as an adapter between RHF's uncontrolled architecture and the UI library's controlled requirements.

```tsx
// ✅ CORRECT: Integrating a Material-UI Switch
<Controller
  name="marketingEmails"
  control={control}
  render={({ field }) => (
    <Switch
      onChange={field.onChange}
      checked={field.value}
      inputRef={field.ref}
    />
  )}
/>
```
