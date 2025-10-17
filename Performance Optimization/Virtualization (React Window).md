# Virtualization (React Window)

This file covers virtualization techniques for rendering large lists efficiently in React.
Essential for handling thousands of items without performance degradation.

---

## What is virtualization

Virtualization renders only visible items in a list, dramatically improving performance for large datasets by:

* Rendering only items currently in the viewport
* Maintaining constant DOM node count regardless of list size
* Recycling DOM elements as user scrolls
* Reducing memory usage and CPU overhead
* Enabling smooth scrolling for thousands of items

Instead of rendering 10,000 DOM nodes, virtualization might render only 10-20 visible ones.

---

## React Window vs React Virtualized

### React Window (recommended)

* Lightweight rewrite of react-virtualized
* Smaller bundle size (~6KB vs ~27KB)
* Simpler API and better tree shaking
* Better TypeScript support
* Actively maintained

### React Virtualized (legacy)

* More features out of the box
* Larger community and ecosystem
* More complex API
* Larger bundle size

**Recommendation**: Use React Window for new projects.

---

## Basic implementation

### Installation

```bash
npm install react-window react-window-infinite-loader
# Optional: for auto-sizing
npm install react-virtualized-auto-sizer
```

### Fixed size list

```jsx
import { FixedSizeList as List } from 'react-window';

const Row = ({ index, style }) => (
  <div style={style}>
    Row {index}
  </div>
);

function VirtualizedList({ items }) {
  return (
    <List
      height={600}        // Container height
      itemCount={items.length}
      itemSize={35}       // Each row height
      width="100%"
    >
      {Row}
    </List>
  );
}
```

### Variable size list

```jsx
import { VariableSizeList as List } from 'react-window';

const getItemSize = (index) => {
  // Dynamic height based on content
  return index % 2 === 0 ? 50 : 35;
};

const Row = ({ index, style }) => (
  <div style={style}>
    <h4>Item {index}</h4>
    <p>Content of varying height...</p>
  </div>
);

function VariableList({ items }) {
  return (
    <List
      height={600}
      itemCount={items.length}
      itemSize={getItemSize}
      width="100%"
    >
      {Row}
    </List>
  );
}
```

---

## Grid virtualization

### Fixed size grid

```jsx
import { FixedSizeGrid as Grid } from 'react-window';

const Cell = ({ columnIndex, rowIndex, style }) => (
  <div style={style}>
    Cell ({rowIndex}, {columnIndex})
  </div>
);

function VirtualizedGrid() {
  return (
    <Grid
      columnCount={1000}
      columnWidth={100}
      height={600}
      rowCount={1000}
      rowHeight={35}
      width={800}
    >
      {Cell}
    </Grid>
  );
}
```

### Auto-sizing container

```jsx
import AutoSizer from 'react-virtualized-auto-sizer';
import { FixedSizeList as List } from 'react-window';

function ResponsiveList({ items }) {
  return (
    <div style={{ height: '100vh' }}>
      <AutoSizer>
        {({ height, width }) => (
          <List
            height={height}
            width={width}
            itemCount={items.length}
            itemSize={50}
          >
            {Row}
          </List>
        )}
      </AutoSizer>
    </div>
  );
}
```

---

## Advanced patterns

### Infinite loading

```jsx
import { FixedSizeList as List } from 'react-window';
import InfiniteLoader from 'react-window-infinite-loader';

function InfiniteList({ items, loadMore, hasNextPage, isLoading }) {
  const itemCount = hasNextPage ? items.length + 1 : items.length;
  
  const isItemLoaded = (index) => !!items[index];
  
  const Item = ({ index, style }) => {
    if (!isItemLoaded(index)) {
      return <div style={style}>Loading...</div>;
    }
    
    return (
      <div style={style}>
        {items[index].name}
      </div>
    );
  };

  return (
    <InfiniteLoader
      isItemLoaded={isItemLoaded}
      itemCount={itemCount}
      loadMoreItems={loadMore}
    >
      {({ onItemsRendered, ref }) => (
        <List
          ref={ref}
          height={600}
          itemCount={itemCount}
          itemSize={50}
          onItemsRendered={onItemsRendered}
          width="100%"
        >
          {Item}
        </List>
      )}
    </InfiniteLoader>
  );
}
```

### Dynamic content sizing

```jsx
import { VariableSizeList as List } from 'react-window';
import { useCallback, useRef } from 'react';

function DynamicList({ items }) {
  const listRef = useRef();
  const rowHeights = useRef({});

  const getItemSize = useCallback((index) => {
    return rowHeights.current[index] || 50;
  }, []);

  const setItemSize = (index, size) => {
    if (rowHeights.current[index] !== size) {
      rowHeights.current[index] = size;
      listRef.current?.resetAfterIndex(index);
    }
  };

  const Row = ({ index, style }) => {
    const rowRef = useRef();
    
    useEffect(() => {
      if (rowRef.current) {
        const height = rowRef.current.offsetHeight;
        setItemSize(index, height);
      }
    }, [index]);

    return (
      <div style={style}>
        <div ref={rowRef}>
          <h3>{items[index].title}</h3>
          <p>{items[index].content}</p>
        </div>
      </div>
    );
  };

  return (
    <List
      ref={listRef}
      height={600}
      itemCount={items.length}
      itemSize={getItemSize}
      width="100%"
    >
      {Row}
    </List>
  );
}
```

---

## Performance optimization

### Memoized components

```jsx
import { memo } from 'react';

const ListItem = memo(({ index, style, data }) => {
  const item = data[index];
  
  return (
    <div style={style} className="list-item">
      <img src={item.avatar} alt={item.name} />
      <div>
        <h4>{item.name}</h4>
        <p>{item.email}</p>
      </div>
    </div>
  );
});

function OptimizedList({ items }) {
  return (
    <List
      height={600}
      itemCount={items.length}
      itemSize={80}
      itemData={items} // Pass data to avoid prop drilling
      width="100%"
    >
      {ListItem}
    </List>
  );
}
```

### Custom scrolling behavior

```jsx
function ScrollToList({ items }) {
  const listRef = useRef();
  
  const scrollToItem = (index) => {
    listRef.current?.scrollToItem(index, 'center');
  };
  
  const scrollToTop = () => {
    listRef.current?.scrollTo(0);
  };

  return (
    <div>
      <button onClick={() => scrollToItem(100)}>Go to item 100</button>
      <button onClick={scrollToTop}>Scroll to top</button>
      
      <List
        ref={listRef}
        height={600}
        itemCount={items.length}
        itemSize={50}
        width="100%"
      >
        {Row}
      </List>
    </div>
  );
}
```

---

## Real-world examples

### Chat messages list

```jsx
import { VariableSizeList as List } from 'react-window';

const Message = ({ index, style, data }) => {
  const message = data.messages[index];
  const isOwn = message.userId === data.currentUserId;
  
  return (
    <div style={style} className={`message ${isOwn ? 'own' : 'other'}`}>
      <div className="message-content">
        <strong>{message.userName}</strong>
        <p>{message.text}</p>
        <small>{formatTime(message.timestamp)}</small>
      </div>
    </div>
  );
};

function ChatMessagesList({ messages, currentUserId }) {
  const getItemSize = useCallback((index) => {
    const message = messages[index];
    // Estimate height based on content length
    const baseHeight = 60;
    const textHeight = Math.ceil(message.text.length / 50) * 20;
    return baseHeight + textHeight;
  }, [messages]);

  const itemData = { messages, currentUserId };

  return (
    <List
      height={400}
      itemCount={messages.length}
      itemSize={getItemSize}
      itemData={itemData}
      width="100%"
      initialScrollOffset={0}
    >
      {Message}
    </List>
  );
}
```

### Data table with virtualization

```jsx
import { FixedSizeGrid as Grid } from 'react-window';

const Cell = ({ columnIndex, rowIndex, style, data }) => {
  const { rows, columns } = data;
  const row = rows[rowIndex];
  const column = columns[columnIndex];
  
  return (
    <div style={style} className="table-cell">
      {row[column.key]}
    </div>
  );
};

function VirtualizedTable({ rows, columns }) {
  const itemData = { rows, columns };
  
  return (
    <div className="virtual-table">
      {/* Header row */}
      <div className="table-header">
        {columns.map((col) => (
          <div key={col.key} className="header-cell">
            {col.title}
          </div>
        ))}
      </div>
      
      {/* Virtualized body */}
      <Grid
        columnCount={columns.length}
        columnWidth={150}
        height={400}
        rowCount={rows.length}
        rowHeight={40}
        width={columns.length * 150}
        itemData={itemData}
      >
        {Cell}
      </Grid>
    </div>
  );
}
```

---

## Alternative libraries

### react-virtual (TanStack)

```jsx
import { useVirtualizer } from '@tanstack/react-virtual';

function TanStackVirtualList({ items }) {
  const parentRef = useRef();
  
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
    overscan: 5, // Render extra items for smoother scrolling
  });

  return (
    <div ref={parentRef} style={{ height: 600, overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.index}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: virtualItem.size,
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            {items[virtualItem.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Native browser virtualization

```css
/* CSS-only virtualization for simple cases */
.scrollable-list {
  height: 400px;
  overflow-y: auto;
  contain: layout style paint;
}

.list-item {
  content-visibility: auto;
  contain-intrinsic-size: 50px;
}
```

---

## Performance considerations

### When to use virtualization

* Lists with 100+ items (especially 1000+)
* Each item has significant DOM complexity
* Mobile devices with limited memory
* Real-time data feeds (chat, logs, metrics)
* Large tables or grids

### When NOT to use virtualization

* Small lists (< 50 items)
* Simple text-only items
* Infrequent updates
* Complex item interactions (drag & drop)
* SEO-critical content (search engines can't scroll)

### Measuring impact

```jsx
// Before virtualization
function RegularList({ items }) {
  const startTime = performance.now();
  
  useEffect(() => {
    const endTime = performance.now();
    console.log(`Render time: ${endTime - startTime}ms`);
    console.log(`DOM nodes: ${document.querySelectorAll('.list-item').length}`);
  });

  return (
    <div>
      {items.map((item, index) => (
        <div key={index} className="list-item">
          {item.name}
        </div>
      ))}
    </div>
  );
}
```

---

## Best practices

* Use `memo()` for list item components to prevent unnecessary re-renders
* Implement proper key props for stable item identity
* Consider overscan for smoother scrolling experience
* Use `itemData` prop to avoid recreating props on each render
* Implement proper loading states for infinite lists
* Test on low-end devices and slow networks
* Consider accessibility implications (screen readers, keyboard navigation)
* Profile memory usage with large datasets
* Use AutoSizer for responsive layouts

---

## Interview-ready summary

Virtualization renders only visible items in large lists, dramatically improving performance by maintaining constant DOM nodes regardless of list size. React Window is the modern solution (smaller than react-virtualized) supporting fixed/variable size lists and grids. Key benefits include reduced memory usage, faster rendering, and smooth scrolling for thousands of items. Use for lists with 100+ complex items, but avoid for small lists or SEO-critical content. Essential patterns include infinite loading, auto-sizing, and memoized components.
