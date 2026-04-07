# React Hook Form: Production-Ready Form Management

## Installation & Setup

```bash
npm install react-hook-form
```

## Basic Form with 20 Fields

```javascript
import { useForm } from 'react-hook-form';

function ApplicationForm() {
  const {
    register,
    handleSubmit,
    watch,
    formState: { errors },
    control,
  } = useForm({
    defaultValues: {
      firstName: '',
      lastName: '',
      email: '',
      phone: '',
      // ... 16 more fields
    },
  });

  const onSubmit = (data) => {
    console.log('Form data:', data);
    // Submit to API
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* Field 1 */}
      <input
        {...register('firstName', {
          required: 'First name is required',
          minLength: { value: 2, message: 'Min 2 characters' },
        })}
      />
      {errors.firstName && <span>{errors.firstName.message}</span>}

      {/* Field 2 */}
      <input
        {...register('email', {
          required: 'Email is required',
          pattern: {
            value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
            message: 'Invalid email',
          },
        })}
      />
      {errors.email && <span>{errors.email.message}</span>}

      <button type="submit">Submit</button>
    </form>
  );
}
```

## Conditional Fields

```javascript
function ConditionalForm() {
  const { register, watch, formState: { errors } } = useForm();

  // Watch specific field to show/hide other fields
  const accountType = watch('accountType');

  return (
    <form>
      <select {...register('accountType', { required: true })}>
        <option value="personal">Personal</option>
        <option value="business">Business</option>
      </select>

      {/* Show only if business */}
      {accountType === 'business' && (
        <>
          <input
            {...register('companyName', {
              required: 'Company name required',
            })}
            placeholder="Company Name"
          />
          <input
            {...register('taxId', {
              required: 'Tax ID required',
            })}
            placeholder="Tax ID"
          />
        </>
      )}

      {/* Show only if personal */}
      {accountType === 'personal' && (
        <input
          {...register('ssn', {
            required: 'SSN required',
          })}
          placeholder="Social Security Number"
        />
      )}
    </form>
  );
}
```

## Complex Validation

```javascript
function RegistrationForm() {
  const {
    register,
    watch,
    formState: { errors },
  } = useForm();

  const password = watch('password');

  return (
    <form>
      <input
        {...register('password', {
          required: 'Password required',
          minLength: { value: 8, message: 'Min 8 characters' },
          validate: {
            hasUpper: (value) =>
              /[A-Z]/.test(value) || 'Must have uppercase',
            hasLower: (value) =>
              /[a-z]/.test(value) || 'Must have lowercase',
            hasNumber: (value) =>
              /[0-9]/.test(value) || 'Must have number',
          },
        })}
        type="password"
      />
      {errors.password && <span>{errors.password.message}</span>}

      {/* Confirm password must match */}
      <input
        {...register('confirmPassword', {
          required: 'Confirm password',
          validate: (value) => value === password || 'Passwords do not match',
        })}
        type="password"
      />
      {errors.confirmPassword && (
        <span>{errors.confirmPassword.message}</span>
      )}
    </form>
  );
}
```

## Dynamic Arrays (Repeating Fields)

```javascript
import { useFieldArray, Controller } from 'react-hook-form';

function MultiContactForm() {
  const { register, control, watch } = useForm({
    defaultValues: {
      contacts: [{ name: '', email: '' }],
    },
  });

  const { fields, append, remove } = useFieldArray({
    control,
    name: 'contacts',
  });

  return (
    <form>
      {fields.map((field, index) => (
        <div key={field.id}>
          <input
            {...register(`contacts.${index}.name`, {
              required: 'Name required',
            })}
            placeholder="Name"
          />
          <input
            {...register(`contacts.${index}.email`, {
              required: 'Email required',
            })}
            placeholder="Email"
          />
          <button type="button" onClick={() => remove(index)}>
            Remove Contact
          </button>
        </div>
      ))}

      <button
        type="button"
        onClick={() => append({ name: '', email: '' })}
      >
        Add Another Contact
      </button>
    </form>
  );
}
```

## Async Validation

```javascript
function EmailCheckForm() {
  const {
    register,
    formState: { errors },
  } = useForm();

  return (
    <form>
      <input
        {...register('email', {
          required: 'Email required',
          validate: async (email) => {
            const response = await fetch(`/api/check-email?email=${email}`);
            const { available } = await response.json();
            return available || 'Email already in use';
          },
        })}
        placeholder="Email"
      />
      {errors.email && <span>{errors.email.message}</span>}
    </form>
  );
}
```

## Form State Management

```javascript
function CompleteForm() {
  const {
    register,
    handleSubmit,
    reset,
    formState: { isDirty, isValid, isSubmitting, errors },
  } = useForm({ mode: 'onChange' });

  const onSubmit = async (data) => {
    await submitForm(data);
    reset(); // Clear form after submit
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* Fields */}
      <input {...register('name', { required: true })} />

      {/* Status indicators */}
      {isDirty && <p>You have unsaved changes</p>}
      {isSubmitting && <p>Submitting...</p>}

      <button type="submit" disabled={!isValid || isSubmitting}>
        Submit
      </button>
    </form>
  );
}
```

## Key Benefits

- Zero re-renders for unchanged fields
- Only ~7KB minified
- TypeScript support
- Integrates with UI libraries (Material-UI, Chakra, etc.)
- Async validation out of box
- Cross-field validation
- No dependencies needed for basic forms
