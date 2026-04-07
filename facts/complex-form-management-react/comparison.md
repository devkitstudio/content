# React Hook Form vs Formik vs useReducer

## Comprehensive Comparison

| Feature | React Hook Form | Formik | useReducer |
|---------|-----------------|--------|-----------|
| Bundle Size | 7KB | 17KB | 0KB (built-in) |
| Learning Curve | Gentle | Steeper | Steep |
| Re-renders | Minimal | Many | Custom |
| Form Validation | Native + custom | Built-in | Manual |
| Async Validation | Yes | Yes | Manual |
| Dynamic Fields | Yes | Yes | Manual |
| File Upload | Yes | Limited | Manual |
| Integration | Easy | Medium | Hard |
| TypeScript | Excellent | Good | Good |
| DevTools | Good | Good | Reducers DevTools |

## React Hook Form

```javascript
import { useForm } from 'react-hook-form';

function Form() {
  const { register, handleSubmit, formState: { errors } } = useForm();

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name', { required: true })} />
      {errors.name && <span>Required</span>}
      <button>Submit</button>
    </form>
  );
}
```

**Pros:**
- Minimal re-renders
- Simple API
- Great performance
- Excellent TypeScript support

**Cons:**
- Requires understanding uncontrolled components
- Validation happens at field level

**Best for:** Most React applications, performance-critical forms

## Formik

```javascript
import { Formik, Form, Field, ErrorMessage } from 'formik';
import * as Yup from 'yup';

const validationSchema = Yup.object().shape({
  name: Yup.string().required('Name required'),
  email: Yup.string().email().required(),
});

function FormikForm() {
  return (
    <Formik
      initialValues={{ name: '', email: '' }}
      validationSchema={validationSchema}
      onSubmit={(values) => console.log(values)}
    >
      <Form>
        <Field name="name" />
        <ErrorMessage name="name" />
        <button type="submit">Submit</button>
      </Form>
    </Formik>
  );
}
```

**Pros:**
- Declarative validation with Yup
- Built-in form state management
- Good documentation
- Great for simple to medium forms

**Cons:**
- More bundle size
- More re-renders by default
- Steeper learning curve
- Yup schema can get complex

**Best for:** Enterprise forms with complex validation rules

## useReducer (Custom)

```javascript
import { useReducer } from 'react';

const initialState = {
  values: { name: '', email: '' },
  errors: {},
  touched: {},
  isSubmitting: false,
};

function formReducer(state, action) {
  switch (action.type) {
    case 'SET_FIELD':
      return {
        ...state,
        values: { ...state.values, [action.name]: action.value },
      };
    case 'SET_ERROR':
      return {
        ...state,
        errors: { ...state.errors, [action.name]: action.error },
      };
    case 'SET_TOUCHED':
      return {
        ...state,
        touched: { ...state.touched, [action.name]: true },
      };
    case 'SUBMIT_START':
      return { ...state, isSubmitting: true };
    case 'SUBMIT_END':
      return { ...state, isSubmitting: false };
    case 'RESET':
      return initialState;
    default:
      return state;
  }
}

function CustomForm() {
  const [state, dispatch] = useReducer(formReducer, initialState);

  const handleChange = (e) => {
    const { name, value } = e.target;
    dispatch({ type: 'SET_FIELD', name, value });

    // Validate
    const error = validateField(name, value);
    dispatch({ type: 'SET_ERROR', name, error });
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    dispatch({ type: 'SUBMIT_START' });
    await submitForm(state.values);
    dispatch({ type: 'SUBMIT_END' });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        name="name"
        value={state.values.name}
        onChange={handleChange}
      />
      {state.errors.name && <span>{state.errors.name}</span>}
      <button disabled={state.isSubmitting}>Submit</button>
    </form>
  );
}
```

**Pros:**
- Full control
- No dependencies
- Predictable state updates
- Good for learning

**Cons:**
- Lots of boilerplate
- Manual validation
- Manual async handling
- Complex for multi-field forms

**Best for:** Learning, understanding form flow, lightweight single-form apps

## Real-World Decision Tree

```
Is it a simple form (< 5 fields)?
├─ YES: useState + basic validation
└─ NO: Go to next question

Does it need complex async validation?
├─ YES: React Hook Form or Formik
└─ NO: Go to next question

Do you already use Formik in your app?
├─ YES: Use Formik for consistency
└─ NO: Use React Hook Form (lighter, easier)
```

## Performance Comparison

For 20-field form with real-time validation:

- **React Hook Form**: ~2-3 re-renders
- **Formik**: ~8-12 re-renders
- **useReducer**: ~1 re-render (if optimized)

React Hook Form wins for most use cases due to minimal re-renders.
