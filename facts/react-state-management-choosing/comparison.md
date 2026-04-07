# Redux vs Zustand vs Jotai vs Context vs TanStack Query

## Comprehensive Comparison Table

| Feature | Redux | Zustand | Jotai | Context | TanStack Query |
|---------|-------|---------|-------|---------|---|
| Bundle Size | 15KB | 2.5KB | 3.8KB | 0KB | 8KB |
| Learning Curve | Steep | Gentle | Medium | Gentle | Medium |
| DevTools | Excellent | Good | Basic | None | Good |
| Time-Travel Debug | Yes | No | No | No | No |
| TypeScript | Excellent | Excellent | Excellent | Good | Excellent |
| Boilerplate | High | Low | Low | Low | Low |
| Async Support | Middleware | Native | Native | Not ideal | Native |
| Caching | Manual | Manual | Manual | Manual | Automatic |
| Concurrent Safe | Yes | Yes | Yes | No | Yes |
| Best For | Large apps | Most apps | Atoms | Prop drilling | Server data |

## Redux Example

```javascript
// store.js
import { createSlice, configureStore } from '@reduxjs/toolkit';

const userSlice = createSlice({
  name: 'user',
  initialState: { user: null, isLoading: false },
  reducers: {
    setUser: (state, action) => {
      state.user = action.payload;
    },
    setLoading: (state, action) => {
      state.isLoading = action.payload;
    },
  },
});

export const store = configureStore({
  reducer: { user: userSlice.reducer },
});

// Component.jsx
import { useSelector, useDispatch } from 'react-redux';

function Profile() {
  const user = useSelector(state => state.user.user);
  const dispatch = useDispatch();

  useEffect(() => {
    dispatch(setLoading(true));
    fetch('/api/user')
      .then(r => r.json())
      .then(data => {
        dispatch(setUser(data));
        dispatch(setLoading(false));
      });
  }, [dispatch]);

  return <div>{user?.name}</div>;
}
```

## Zustand Example

```javascript
// store.js
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

const useUserStore = create(
  devtools(
    persist(
      (set) => ({
        user: null,
        isLoading: false,
        setUser: (user) => set({ user }),
        setLoading: (loading) => set({ isLoading: loading }),
        fetchUser: async () => {
          set({ isLoading: true });
          const res = await fetch('/api/user');
          const data = await res.json();
          set({ user: data, isLoading: false });
        },
      }),
      { name: 'user-storage' }
    )
  )
);

// Component.jsx
function Profile() {
  const user = useUserStore(state => state.user);
  const fetchUser = useUserStore(state => state.fetchUser);

  useEffect(() => {
    fetchUser();
  }, [fetchUser]);

  return <div>{user?.name}</div>;
}
```

## Jotai Example

```javascript
// atoms.js
import { atom } from 'jotai';

const userAtom = atom(null);
const isLoadingAtom = atom(false);

export const userWithLoading = atom(async (get) => {
  const user = get(userAtom);
  const isLoading = get(isLoadingAtom);
  return { user, isLoading };
});

// Component.jsx
import { useAtom, useSetAtom } from 'jotai';

function Profile() {
  const [user] = useAtom(userAtom);
  const setLoading = useSetAtom(isLoadingAtom);

  useEffect(() => {
    setLoading(true);
    fetch('/api/user')
      .then(r => r.json())
      .then(data => {
        setUser(data);
        setLoading(false);
      });
  }, []);

  return <div>{user?.name}</div>;
}
```

## Context Example

```javascript
// UserContext.jsx
const UserContext = createContext();

export function UserProvider({ children }) {
  const [user, setUser] = useState(null);
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    setIsLoading(true);
    fetch('/api/user')
      .then(r => r.json())
      .then(data => {
        setUser(data);
        setIsLoading(false);
      });
  }, []);

  return (
    <UserContext.Provider value={{ user, isLoading, setUser }}>
      {children}
    </UserContext.Provider>
  );
}

// Component.jsx
function Profile() {
  const { user } = useContext(UserContext);
  return <div>{user?.name}</div>;
}
```

## TanStack Query Example

```javascript
// hooks/useUser.js
import { useQuery } from '@tanstack/react-query';

export function useUser() {
  return useQuery({
    queryKey: ['user'],
    queryFn: () => fetch('/api/user').then(r => r.json()),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
}

// Component.jsx
function Profile() {
  const { data: user, isLoading } = useUser();
  return <div>{isLoading ? 'Loading...' : user?.name}</div>;
}
```

## When to Use Each

**Redux:** Large enterprise apps with complex state, multiple teams, time-travel debugging requirement

**Zustand:** Most React applications, startups, teams favoring simplicity

**Jotai:** Atomic architecture preference, functional composition patterns

**Context API:** Small projects, avoiding external dependencies, simple prop drilling fixes

**TanStack Query:** Any app with server data (90% of apps), critical for caching and sync
