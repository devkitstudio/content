# Preventing Unnecessary Re-renders with React.memo, useMemo, useCallback

## React.memo for Props Comparison

```javascript
// Before: Re-renders on every parent change
function SearchInput({ value, onChange }) {
  console.log('SearchInput rendered');
  return <input value={value} onChange={onChange} />;
}

// After: Only re-renders when props change
const SearchInput = React.memo(({ value, onChange }) => {
  console.log('SearchInput rendered');
  return <input value={value} onChange={onChange} />;
});
```

## useCallback to Prevent New Function References

```javascript
// Problem: onChange is new every render
function Parent() {
  const [input, setInput] = useState('');

  const handleChange = (e) => setInput(e.target.value);

  return <SearchInput value={input} onChange={handleChange} />;
  // handleChange is different on every render!
}

// Solution: useCallback memoizes function
function Parent() {
  const [input, setInput] = useState('');

  const handleChange = useCallback((e) => {
    setInput(e.target.value);
  }, []); // Dependencies: none needed here

  return <SearchInput value={input} onChange={handleChange} />;
}

const SearchInput = React.memo(({ value, onChange }) => {
  return <input value={value} onChange={onChange} />;
});
```

## useMemo for Expensive Computations

```javascript
// Problem: filterResults runs on every render
function UserList({ users, searchTerm }) {
  const filteredUsers = users.filter(user =>
    user.name.includes(searchTerm)
  );

  return (
    <div>
      {filteredUsers.map(user => (
        <UserCard key={user.id} user={user} />
      ))}
    </div>
  );
}

// Solution: useMemo caches result
function UserList({ users, searchTerm }) {
  const filteredUsers = useMemo(
    () => users.filter(user => user.name.includes(searchTerm)),
    [users, searchTerm] // Recalculate only when these change
  );

  return (
    <div>
      {filteredUsers.map(user => (
        <UserCard key={user.id} user={user} />
      ))}
    </div>
  );
}
```

## Combined Example: Complete Input Form

```javascript
const UserSearchForm = React.memo(({ onSearch }) => {
  const [input, setInput] = useState('');

  const handleChange = useCallback((e) => {
    setInput(e.target.value);
  }, []);

  const handleSubmit = useCallback((e) => {
    e.preventDefault();
    onSearch(input);
  }, [input, onSearch]);

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={input}
        onChange={handleChange}
        placeholder="Search..."
      />
      <button type="submit">Search</button>
    </form>
  );
});

function App() {
  const [results, setResults] = useState([]);

  const handleSearch = useCallback((query) => {
    setResults(api.search(query));
  }, []);

  return (
    <div>
      <UserSearchForm onSearch={handleSearch} />
      <ResultsList results={results} />
    </div>
  );
}
```

## Key Points

- **React.memo**: Prevents re-render if props are identical
- **useCallback**: Memoizes function reference across renders
- **useMemo**: Memoizes computed values
- Only use when profiling shows actual performance issues
