## Decision Framework

**Choose Tailwind if:**
- Team size < 20 (lower onboarding)
- Rapid prototyping priority
- Accept tight coupling (markup + utilities)
- Want single CSS file (~500KB gzipped)
- Happy with pre-defined design system

**Choose CSS Modules if:**
- Scoped styles required
- Multiple design systems per project
- Team with CSS expertise
- Legacy app migration
- No runtime JS needed

**Choose Styled Components if:**
- Heavy theming/dynamic styles
- Props-based styling requirement
- Component-first architecture
- Need CSS-in-JS for code organization
- Accept JS bundle size increase

**Choose Global CSS if:**
- Old project migration
- No component framework
- Simple sites (blogs, marketing)
- Multiple teams (hard to coordinate)

## Tailwind (Recommended for Most)

**Pros:**
- Single source of truth (design tokens)
- Small bundle (60KB gzipped with purge)
- Zero runtime overhead
- Great tooling (Intellisense, plugins)
- Team consistency forced

**Cons:**
- Markup bloat (className strings)
- Painful if design doesn't fit defaults
- Hard to share styles between components

```typescript
// Good: Component encapsulation
export function Button({ variant = 'primary' }) {
  const styles = {
    primary: 'bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700',
    secondary: 'bg-gray-200 text-gray-900 px-4 py-2 rounded hover:bg-gray-300'
  };

  return (
    <button className={styles[variant]}>
      Click me
    </button>
  );
}
```

**Better: With @apply for repeated patterns**

```css
@layer components {
  @apply text-base;

  .btn {
    @apply px-4 py-2 rounded font-medium transition;
  }

  .btn-primary {
    @apply btn bg-blue-600 text-white hover:bg-blue-700;
  }

  .btn-secondary {
    @apply btn bg-gray-200 text-gray-900 hover:bg-gray-300;
  }
}
```

```typescript
export function Button({ variant = 'primary' }) {
  return (
    <button className={`btn btn-${variant}`}>
      Click me
    </button>
  );
}
```

## CSS Modules

**Pros:**
- No naming conflicts
- Local scope by default
- Works with plain CSS
- Good for migrating legacy code

**Cons:**
- No dynamic styling (hard to add dark mode)
- Class name indirection
- Setup complexity

```typescript
// styles.module.css
.button {
  padding: 8px 16px;
  border-radius: 4px;
  font-weight: 500;
  transition: background-color 0.2s;
}

.primary {
  background-color: #2563eb;
  color: white;
}

.primary:hover {
  background-color: #1d4ed8;
}

// Button.tsx
import styles from './Button.module.css';

export function Button({ variant = 'primary' }) {
  return (
    <button className={`${styles.button} ${styles[variant]}`}>
      Click me
    </button>
  );
}
```

## Styled Components

**Pros:**
- Full CSS power + JS logic
- Dynamic theming
- Automatically vendor-prefixed
- Good for complex interactions

**Cons:**
- Bundle size (15KB min)
- Runtime performance cost
- Styling in JS = less familiar to designers

```typescript
import styled from 'styled-components';

const StyledButton = styled.button`
  padding: 8px 16px;
  border-radius: 4px;
  font-weight: 500;
  background-color: ${props => props.$variant === 'primary' ? '#2563eb' : '#e5e7eb'};
  color: ${props => props.$variant === 'primary' ? 'white' : '#111827'};
  transition: background-color 0.2s;

  &:hover {
    background-color: ${props => props.$variant === 'primary' ? '#1d4ed8' : '#d1d5db'};
  }
`;

export function Button({ variant = 'primary' }) {
  return <StyledButton $variant={variant}>Click me</StyledButton>;
}
```

## Recommended: Hybrid Approach

```
For most projects:
├─ Tailwind for component base utilities (80% of styles)
├─ CSS Modules for complex/scoped components (15%)
└─ Styled Components for heavily dynamic features (5%)
```

Example structure:

```typescript
// Simple components: Tailwind
export function Card({ children }) {
  return (
    <div className="bg-white rounded-lg shadow p-6 hover:shadow-lg transition">
      {children}
    </div>
  );
}

// Scoped complex styles: CSS Module
// DataTable.tsx + DataTable.module.css
export function DataTable({ data }) {
  return (
    <div className={styles.tableContainer}>
      <table className={styles.table}>
        {/* ... */}
      </table>
    </div>
  );
}

// Highly dynamic: Styled Components
// ThemeProvider
const themes = {
  light: { bg: '#fff', text: '#000' },
  dark: { bg: '#1a1a1a', text: '#fff' }
};

export function ThemedApp() {
  const [theme, setTheme] = useState('light');

  return (
    <ThemeProvider theme={themes[theme]}>
      <AppContent />
    </ThemeProvider>
  );
}
```
