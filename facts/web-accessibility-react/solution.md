## Core Principles

**WCAG 2.1 Level AA targets:**
- Perceivable: Text alternatives, captions, readable contrast
- Operable: Keyboard navigation, no seizure triggers
- Understandable: Clear language, consistent navigation
- Robust: Valid HTML, assistive technology support

## 1. Semantic HTML

```typescript
// Bad: Non-semantic
export function Button() {
  return <div onClick={() => {}}>Click</div>;
}

// Good: Semantic
export function Button() {
  return <button onClick={() => {}}>Click</button>;
}
```

Use proper elements:
- `<button>` not `<div>` for actions
- `<a>` not `<button>` for navigation
- `<header>`, `<main>`, `<footer>` for structure
- `<h1>` through `<h6>` for headings (in order)
- `<label>` for form inputs

## 2. ARIA (Accessible Rich Internet Applications)

For custom components that can't use semantic HTML:

```typescript
// Custom dropdown
export function Dropdown() {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div className="dropdown">
      <button
        onClick={() => setIsOpen(!isOpen)}
        aria-haspopup="listbox"
        aria-expanded={isOpen}
        aria-controls="options-list"
      >
        Select option
      </button>

      {isOpen && (
        <ul
          id="options-list"
          role="listbox"
          aria-label="Options"
        >
          <li role="option" aria-selected={false}>Option 1</li>
          <li role="option" aria-selected={true}>Option 2</li>
        </ul>
      )}
    </div>
  );
}
```

**Common ARIA attributes:**
- `role`: Semantic meaning (button, listbox, dialog)
- `aria-label`: Label for icon buttons
- `aria-describedby`: Longer description
- `aria-expanded`: Show/hide state
- `aria-selected`: Selection state
- `aria-checked`: Checkbox/toggle state
- `aria-hidden="true"`: Hide from screen readers

## 3. Keyboard Navigation

```typescript
export function Modal({ isOpen, onClose }) {
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!isOpen) return;

    const handleEscape = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        onClose();
      }
    };

    document.addEventListener('keydown', handleEscape);
    return () => document.removeEventListener('keydown', handleEscape);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return (
    <div role="dialog" ref={ref} aria-modal="true">
      <h2>Confirm Action</h2>
      <button onClick={onClose}>Cancel (Esc)</button>
      <button onClick={() => { /* ... */ }}>Confirm (Enter)</button>
    </div>
  );
}
```

Use:
- `<button>` and `<a>` are keyboard accessible by default
- Tab order: Use `tabIndex` sparingly, tab should follow DOM order
- Trap focus in modals
- Escape closes modals
- Enter submits forms

## 4. Focus Management

```typescript
export function ComboBox() {
  const inputRef = useRef<HTMLInputElement>(null);
  const [suggestions, setSuggestions] = useState([]);

  const onInputChange = (value: string) => {
    setSuggestions(filterSuggestions(value));
    // Keep focus on input
    inputRef.current?.focus();
  };

  return (
    <div>
      <input
        ref={inputRef}
        type="text"
        onChange={(e) => onInputChange(e.target.value)}
        aria-autocomplete="list"
        aria-expanded={suggestions.length > 0}
        aria-controls="suggestions-list"
      />
      {suggestions.length > 0 && (
        <ul id="suggestions-list">
          {suggestions.map((s, i) => (
            <li key={i}>{s}</li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

## 5. Images & Media

```typescript
// Bad: Missing alt
export function ProfileImage() {
  return <img src="user.jpg" />;
}

// Good: Descriptive alt
export function ProfileImage({ userName }) {
  return (
    <img
      src="user.jpg"
      alt={`${userName}'s profile picture`}
    />
  );
}

// Decorative images
export function IconButton() {
  return (
    <button aria-label="Delete">
      <img src="trash.svg" alt="" aria-hidden="true" />
    </button>
  );
}
```

## 6. Color & Contrast

```typescript
// Bad: Insufficient contrast
<button className="text-gray-400 bg-gray-500">Submit</button>

// Good: WCAG AA compliant (4.5:1 text, 3:1 graphics)
<button className="text-white bg-blue-600">Submit</button>

// Check: Use WebAIM Contrast Checker
```

## 7. Form Labels

```typescript
// Bad: No label
export function Login() {
  return (
    <input type="email" placeholder="Email" />
  );
}

// Good: Associated label
export function Login() {
  return (
    <>
      <label htmlFor="email">Email</label>
      <input id="email" type="email" />
    </>
  );
}

// Error handling
export function PasswordInput() {
  const [error, setError] = useState('');

  return (
    <>
      <label htmlFor="password">Password</label>
      <input
        id="password"
        type="password"
        aria-invalid={!!error}
        aria-describedby={error ? 'password-error' : undefined}
      />
      {error && <span id="password-error">{error}</span>}
    </>
  );
}
```
