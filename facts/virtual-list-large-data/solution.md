# Virtual Scrolling with react-window & TanStack Virtual

## The Problem

```javascript
// SLOW: Renders all 10,000 items
function BadLargeList({ items }) {
  return (
    <div>
      {items.map(item => (
        <div key={item.id}>{item.name}</div>
      ))}
    </div>
  );
}

// Result: Browser freezes, 10,000 DOM nodes!
```

## The Solution: Windowing

```
Viewport (500px height)
┌─────────────────────┐
│ Item 1 (visible)    │ <- Only render these 5-10 items
│ Item 2 (visible)    │
│ Item 3 (visible)    │
│ Item 4 (visible)    │
│ Item 5 (visible)    │
├─────────────────────┤
│ Item 6-9995 (hidden)│ <- Skip these, save space with spacers
├─────────────────────┤
│ Item 9996 (visible) │ <- Render next visible
│ Item 9997 (visible) │
│ Item 9998 (visible) │
│ Item 9999 (visible) │
│ Item 10000 (visible)│
└─────────────────────┘
```

## react-window Installation

```bash
npm install react-window
```

## Basic Fixed-Height List

```javascript
import { FixedSizeList as List } from 'react-window';

const items = Array.from({ length: 10000 }, (_, i) => ({
  id: i,
  name: `Item ${i}`,
}));

const Row = ({ index, style }) => (
  <div style={style} className="list-item">
    {items[index].name}
  </div>
);

function VirtualList() {
  return (
    <List
      height={600}        // Viewport height
      itemCount={10000}   // Total items
      itemSize={35}       // Each item height
      width="100%"
    >
      {Row}
    </List>
  );
}
```

## Variable-Height List

```javascript
import { VariableSizeList as List } from 'react-window';

function VariableList() {
  // Memoize height calculations
  const getItemSize = (index) => {
    const item = items[index];
    return item.type === 'large' ? 100 : 35;
  };

  const listRef = useRef();

  // Clear cache when data changes
  const handleItemChange = (index) => {
    listRef.current?.resetAfterIndex(index);
  };

  return (
    <List
      height={600}
      itemCount={10000}
      itemSize={getItemSize}
      width="100%"
      ref={listRef}
    >
      {({ index, style }) => (
        <div style={style} className="list-item">
          {items[index].name}
        </div>
      )}
    </List>
  );
}
```

## Grid (Table-like) Virtualization

```javascript
import { FixedSizeGrid as Grid } from 'react-window';

const items = Array.from({ length: 10000 }, (_, i) => ({
  id: i,
  col1: `Data ${i}-1`,
  col2: `Data ${i}-2`,
  col3: `Data ${i}-3`,
}));

const Cell = ({ columnIndex, rowIndex, style }) => {
  const item = items[rowIndex];
  const columns = ['col1', 'col2', 'col3'];
  const value = item[columns[columnIndex]];

  return (
    <div style={style} className="grid-cell">
      {value}
    </div>
  );
};

function VirtualTable() {
  return (
    <Grid
      columnCount={3}
      columnWidth={100}
      height={600}
      rowCount={10000}
      rowHeight={35}
      width={300}
    >
      {Cell}
    </Grid>
  );
}
```

## TanStack Virtual (Alternative)

```bash
npm install @tanstack/react-virtual
```

```javascript
import { useVirtualizer } from '@tanstack/react-virtual';
import { useRef } from 'react';

function TanstackVirtualList() {
  const parentRef = useRef();

  const virtualizer = useVirtualizer({
    count: 10000,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 35,
    overscan: 10, // Render 10 extra items for smoother scrolling
  });

  return (
    <div
      ref={parentRef}
      style={{
        height: '600px',
        overflow: 'auto',
      }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          width: '100%',
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualItem.size}px`,
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            Item {virtualItem.index}
          </div>
        ))}
      </div>
    </div>
  );
}
```

## With Search Filter

```javascript
function SearchableVirtualList() {
  const [searchTerm, setSearchTerm] = useState('');
  const listRef = useRef();

  const filteredItems = items.filter(item =>
    item.name.toLowerCase().includes(searchTerm.toLowerCase())
  );

  useEffect(() => {
    // Reset to top when filtering
    listRef.current?.scrollToItem(0);
  }, [searchTerm]);

  return (
    <>
      <input
        type="text"
        placeholder="Search..."
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
      />

      <FixedSizeList
        ref={listRef}
        height={600}
        itemCount={filteredItems.length}
        itemSize={35}
        width="100%"
      >
        {({ index, style }) => (
          <div style={style}>{filteredItems[index].name}</div>
        )}
      </FixedSizeList>
    </>
  );
}
```

## Performance Metrics

```
Without virtualization (10,000 items):
- Initial render: 3000ms
- Scroll frame rate: 15 fps (choppy)
- DOM nodes: 10,000+
- Memory: 50-100MB

With virtualization:
- Initial render: 100ms
- Scroll frame rate: 55+ fps (smooth)
- DOM nodes: 20-30 (visible + overscan)
- Memory: 5-10MB

Improvement: 30x faster, smoother, less memory
```

## Key Points

- Only render visible items + small overscan buffer
- Cache row height calculations for variable-height lists
- Support search filtering with dynamic item count
- 10,000+ items becomes seamless
