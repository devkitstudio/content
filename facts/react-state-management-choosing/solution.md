# Decision Framework: Local vs Global vs Server State

## State Classification Framework

```
┌─────────────────────────────────────────────────────┐
│              WHERE DOES STATE LIVE?                 │
├──────────────────────────────────────────────────────┤
│                                                      │
│  LOCAL STATE              GLOBAL STATE   SERVER STATE
│  (Single component)       (Multi-component)  (API)
│  • Form inputs            • Auth user      • DB data
│  • Toggle menus           • App theme      • API cache
│  • Component UI           • Notifications  • Async data
│  ↓                        ↓                ↓
│  useState()               Redux/Zustand   TanStack Query
│  ├─ Simple                ├─ Predictable  ├─ Auto cache
│  ├─ Fast                  ├─ DevTools     ├─ Sync
│  └─ Sufficient            └─ Scalable     └─ Optimal
└────────────────────────────────────────────────────┘
```

## Decision Matrix

```javascript
// Question 1: Is this data used in MANY components?
if (usedInManyComponents) {
  // Question 2: Is this data from the server?
  if (fromServer) {
    return 'TanStack Query'; // ✓ Best
  } else {
    // Question 3: Need time-travel debugging?
    if (needsDevTools) {
      return 'Redux'; // Complex but powerful
    } else {
      return 'Zustand'; // Simple + effective
    }
  }
} else {
  // Single component or closely related
  return 'useState()'; // Fast and simple
}
```

## Local State (useState)

```javascript
// Use when: Single component or direct children
// Perfect for: Forms, toggles, dropdowns, local UI

function SearchBox() {
  const [query, setQuery] = useState('');
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        onFocus={() => setIsOpen(true)}
      />
      {isOpen && <Dropdown query={query} />}
    </div>
  );
}
```

## Global State (Zustand)

```javascript
// Use when: Multiple unrelated components need same state
// Perfect for: Authentication, theme, notifications

import { create } from 'zustand';

const useAuthStore = create((set) => ({
  user: null,
  isLoggedIn: false,
  login: (user) => set({ user, isLoggedIn: true }),
  logout: () => set({ user: null, isLoggedIn: false }),
}));

// Component 1
function Header() {
  const { user, logout } = useAuthStore();
  return <div>{user?.name} <button onClick={logout}>Logout</button></div>;
}

// Component 2
function Settings() {
  const { user } = useAuthStore();
  return <div>Email: {user?.email}</div>;
}
```

## Server State (TanStack Query)

```javascript
// Use when: Data comes from API/server
// Perfect for: User lists, posts, products, real-time data

import { useQuery, useMutation } from '@tanstack/react-query';

function UserProfile() {
  // Automatic caching, refetching, synchronization
  const { data: user, isLoading } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then(r => r.json()),
  });

  const { mutate: updateUser } = useMutation({
    mutationFn: (data) => fetch(`/api/users/${userId}`, {
      method: 'PUT',
      body: JSON.stringify(data),
    }).then(r => r.json()),
    onSuccess: () => {
      // Invalidate cache to refetch
      queryClient.invalidateQueries({ queryKey: ['user', userId] });
    },
  });

  if (isLoading) return <Skeleton />;
  return <UserEditor user={user} onSave={updateUser} />;
}
```

## Context API (Rare Case)

```javascript
// Use when: Need to pass props deeply WITHOUT a library
// NOT for performance-critical state (causes re-renders)

const ThemeContext = createContext();

function App() {
  const [theme, setTheme] = useState('light');

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Header />
      <Content />
      <Footer />
    </ThemeContext.Provider>
  );
}

function Header() {
  const { theme, setTheme } = useContext(ThemeContext);
  return <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>;
}
```

## Recommended Stack

```
For most React apps:

Local UI State (useState)
    ↓
Theme + Auth (Zustand)
    ↓
Server Data (TanStack Query)
```

**Benefits:**
- Simple and understandable
- Scales from 10 to 10,000 components
- Clear separation of concerns
- Minimal boilerplate
