## Execution: The Uncontrolled Pattern

Stop using `onChange` and `value` props. Use the `register` function to inject the `ref` directly into the native HTML input.

```tsx
import { useForm } from 'react-hook-form';

type FormData = {
  email: string;
  age: number;
};

export function PerformanceForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<FormData>();

  // Executes ONLY on submit, extracting data directly from DOM refs
  const onSubmit = async (data: FormData) => {
    await fetch('/api/users', { method: 'POST', body: JSON.stringify(data) });
  };

  return (
    {/* Form submission is intercepted by RHF */}
    <form onSubmit={handleSubmit(onSubmit)}>

      {/* ZERO re-renders on keystroke. Validation happens natively. */}
      <input
        {...register('email', {
          required: 'Email is required',
          pattern: { value: /^\S+@\S+$/i, message: 'Invalid email' }
        })}
        placeholder="Email"
      />
      {errors.email && <span className="error">{errors.email.message}</span>}

      <input
        {...register('age', { min: 18, max: 99 })}
        type="number"
      />

      <button type="submit">Submit Data</button>
    </form>
  );
}
```
