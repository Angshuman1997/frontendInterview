# When would you use a Portal

This file covers specific use cases and scenarios where React Portals are the right solution.
Essential for understanding when to break out of normal React component hierarchy.

---

## What are React Portals?

React Portals provide a way to render children into a DOM node that exists outside the parent component's DOM hierarchy, while maintaining the React component tree relationship for context and event bubbling.

```jsx
import { createPortal } from 'react-dom';

function Portal({ children, container = document.body }) {
  return createPortal(children, container);
}
```

---

## Primary Use Cases

### 1. Modal Dialogs

**Why Portals**: Modals need to appear above all other content and often require fixed positioning relative to the viewport, not their parent container.

```jsx
function Modal({ isOpen, onClose, children }) {
  const [modalRoot, setModalRoot] = useState(null);

  useEffect(() => {
    // Create modal container if it doesn't exist
    let container = document.getElementById('modal-root');
    if (!container) {
      container = document.createElement('div');
      container.id = 'modal-root';
      document.body.appendChild(container);
    }
    setModalRoot(container);
  }, []);

  useEffect(() => {
    if (isOpen) {
      // Prevent background scroll
      document.body.style.overflow = 'hidden';
      
      // Focus management
      const previousActiveElement = document.activeElement;
      
      return () => {
        document.body.style.overflow = '';
        previousActiveElement?.focus();
      };
    }
  }, [isOpen]);

  if (!isOpen || !modalRoot) return null;

  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div 
        className="modal-content" 
        onClick={(e) => e.stopPropagation()}
        role="dialog"
        aria-modal="true"
      >
        <button 
          className="modal-close"
          onClick={onClose}
          aria-label="Close modal"
        >
          ×
        </button>
        {children}
      </div>
    </div>,
    modalRoot
  );
}

// Usage in deeply nested component
function ProductCard({ product }) {
  const [showDetails, setShowDetails] = useState(false);

  return (
    <div className="product-card">
      <img src={product.image} alt={product.name} />
      <h3>{product.name}</h3>
      <button onClick={() => setShowDetails(true)}>
        View Details
      </button>
      
      {/* Modal renders at document.body level, not inside product-card */}
      <Modal isOpen={showDetails} onClose={() => setShowDetails(false)}>
        <ProductDetails product={product} />
      </Modal>
    </div>
  );
}
```

### 2. Tooltips and Popovers

**Why Portals**: Tooltips must appear above other content and position correctly regardless of parent container's overflow or z-index properties.

```jsx
function Tooltip({ 
  children, 
  content, 
  position = 'top',
  delay = 200 
}) {
  const [isVisible, setIsVisible] = useState(false);
  const [tooltipPosition, setTooltipPosition] = useState({});
  const triggerRef = useRef(null);
  const tooltipRef = useRef(null);
  const timeoutRef = useRef(null);

  const showTooltip = useCallback(() => {
    if (timeoutRef.current) clearTimeout(timeoutRef.current);
    
    timeoutRef.current = setTimeout(() => {
      setIsVisible(true);
      // Calculate position after tooltip is rendered
      requestAnimationFrame(() => {
        if (triggerRef.current && tooltipRef.current) {
          const triggerRect = triggerRef.current.getBoundingClientRect();
          const tooltipRect = tooltipRef.current.getBoundingClientRect();
          
          let top, left;
          
          switch (position) {
            case 'top':
              top = triggerRect.top - tooltipRect.height - 8;
              left = triggerRect.left + (triggerRect.width - tooltipRect.width) / 2;
              break;
            case 'bottom':
              top = triggerRect.bottom + 8;
              left = triggerRect.left + (triggerRect.width - tooltipRect.width) / 2;
              break;
            case 'left':
              top = triggerRect.top + (triggerRect.height - tooltipRect.height) / 2;
              left = triggerRect.left - tooltipRect.width - 8;
              break;
            case 'right':
              top = triggerRect.top + (triggerRect.height - tooltipRect.height) / 2;
              left = triggerRect.right + 8;
              break;
          }
          
          // Adjust for viewport boundaries
          const viewport = {
            width: window.innerWidth,
            height: window.innerHeight
          };
          
          if (left < 8) left = 8;
          if (left + tooltipRect.width > viewport.width - 8) {
            left = viewport.width - tooltipRect.width - 8;
          }
          if (top < 8) top = 8;
          if (top + tooltipRect.height > viewport.height - 8) {
            top = triggerRect.top - tooltipRect.height - 8;
          }
          
          setTooltipPosition({
            position: 'fixed',
            top: `${top}px`,
            left: `${left}px`,
            zIndex: 9999
          });
        }
      });
    }, delay);
  }, [position, delay]);

  const hideTooltip = useCallback(() => {
    if (timeoutRef.current) clearTimeout(timeoutRef.current);
    setIsVisible(false);
  }, []);

  const tooltipElement = isVisible ? (
    <div
      ref={tooltipRef}
      className={`tooltip tooltip--${position}`}
      style={tooltipPosition}
      role="tooltip"
    >
      {content}
    </div>
  ) : null;

  return (
    <>
      <span
        ref={triggerRef}
        onMouseEnter={showTooltip}
        onMouseLeave={hideTooltip}
        onFocus={showTooltip}
        onBlur={hideTooltip}
      >
        {children}
      </span>
      {tooltipElement && createPortal(tooltipElement, document.body)}
    </>
  );
}

// Usage in complex layouts
function DataTable({ data }) {
  return (
    <div className="table-container" style={{ overflow: 'auto', height: '400px' }}>
      <table>
        <thead>
          <tr>
            <th>
              <Tooltip content="User's full name" position="top">
                Name
              </Tooltip>
            </th>
            <th>
              <Tooltip content="Email address for contact" position="top">
                Email
              </Tooltip>
            </th>
            <th>
              <Tooltip content="User role in organization" position="top">
                Role
              </Tooltip>
            </th>
          </tr>
        </thead>
        <tbody>
          {data.map(user => (
            <tr key={user.id}>
              <td>
                <Tooltip 
                  content={
                    <div>
                      <strong>{user.name}</strong>
                      <p>Joined: {user.joinDate}</p>
                      <p>Last active: {user.lastActive}</p>
                    </div>
                  }
                  position="right"
                >
                  {user.name}
                </Tooltip>
              </td>
              <td>{user.email}</td>
              <td>{user.role}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

### 3. Dropdown Menus and Select Components

**Why Portals**: Dropdowns must appear above other content and not be clipped by parent containers with `overflow: hidden`.

```jsx
function CustomSelect({ 
  options, 
  value, 
  onChange, 
  placeholder = "Select option...",
  className = ""
}) {
  const [isOpen, setIsOpen] = useState(false);
  const [selectedIndex, setSelectedIndex] = useState(-1);
  const [dropdownPosition, setDropdownPosition] = useState({});
  
  const selectRef = useRef(null);
  const dropdownRef = useRef(null);
  const optionRefs = useRef([]);

  // Position dropdown
  const updateDropdownPosition = useCallback(() => {
    if (!selectRef.current || !dropdownRef.current) return;

    const selectRect = selectRef.current.getBoundingClientRect();
    const dropdownRect = dropdownRef.current.getBoundingClientRect();
    const viewport = {
      width: window.innerWidth,
      height: window.innerHeight
    };

    let top = selectRect.bottom + 4;
    let left = selectRect.left;
    const width = selectRect.width;

    // Check if dropdown would go below viewport
    if (top + dropdownRect.height > viewport.height - 20) {
      top = selectRect.top - dropdownRect.height - 4;
    }

    // Check if dropdown would go outside viewport horizontally
    if (left + width > viewport.width - 20) {
      left = viewport.width - width - 20;
    }

    setDropdownPosition({
      position: 'fixed',
      top: `${top}px`,
      left: `${left}px`,
      width: `${width}px`,
      zIndex: 1000
    });
  }, []);

  // Click outside handler
  useEffect(() => {
    const handleClickOutside = (event) => {
      if (
        selectRef.current && !selectRef.current.contains(event.target) &&
        dropdownRef.current && !dropdownRef.current.contains(event.target)
      ) {
        setIsOpen(false);
        setSelectedIndex(-1);
      }
    };

    if (isOpen) {
      document.addEventListener('mousedown', handleClickOutside);
      updateDropdownPosition();
      
      // Update position on scroll/resize
      const handlePositionUpdate = () => updateDropdownPosition();
      window.addEventListener('scroll', handlePositionUpdate);
      window.addEventListener('resize', handlePositionUpdate);
      
      return () => {
        document.removeEventListener('mousedown', handleClickOutside);
        window.removeEventListener('scroll', handlePositionUpdate);
        window.removeEventListener('resize', handlePositionUpdate);
      };
    }
  }, [isOpen, updateDropdownPosition]);

  // Keyboard navigation
  const handleKeyDown = (event) => {
    switch (event.key) {
      case 'Enter':
      case ' ':
        if (!isOpen) {
          setIsOpen(true);
        } else if (selectedIndex >= 0) {
          onChange(options[selectedIndex]);
          setIsOpen(false);
          setSelectedIndex(-1);
        }
        event.preventDefault();
        break;
        
      case 'ArrowDown':
        if (!isOpen) {
          setIsOpen(true);
        } else {
          setSelectedIndex(prev => 
            prev < options.length - 1 ? prev + 1 : 0
          );
        }
        event.preventDefault();
        break;
        
      case 'ArrowUp':
        if (isOpen) {
          setSelectedIndex(prev => 
            prev > 0 ? prev - 1 : options.length - 1
          );
        }
        event.preventDefault();
        break;
        
      case 'Escape':
        setIsOpen(false);
        setSelectedIndex(-1);
        break;
    }
  };

  // Scroll to highlighted option
  useEffect(() => {
    if (selectedIndex >= 0 && optionRefs.current[selectedIndex]) {
      optionRefs.current[selectedIndex].scrollIntoView({
        behavior: 'smooth',
        block: 'nearest'
      });
    }
  }, [selectedIndex]);

  const selectedOption = options.find(opt => opt.value === value);

  const dropdownElement = isOpen ? (
    <div
      ref={dropdownRef}
      className="custom-select-dropdown"
      style={dropdownPosition}
      role="listbox"
    >
      {options.map((option, index) => (
        <div
          key={option.value}
          ref={el => optionRefs.current[index] = el}
          className={`custom-select-option ${
            index === selectedIndex ? 'highlighted' : ''
          } ${option.value === value ? 'selected' : ''}`}
          role="option"
          aria-selected={option.value === value}
          onClick={() => {
            onChange(option);
            setIsOpen(false);
            setSelectedIndex(-1);
          }}
          onMouseEnter={() => setSelectedIndex(index)}
        >
          {option.label}
        </div>
      ))}
    </div>
  ) : null;

  return (
    <>
      <div
        ref={selectRef}
        className={`custom-select ${isOpen ? 'open' : ''} ${className}`}
        onClick={() => setIsOpen(!isOpen)}
        onKeyDown={handleKeyDown}
        tabIndex={0}
        role="combobox"
        aria-expanded={isOpen}
        aria-haspopup="listbox"
      >
        <span className="custom-select-value">
          {selectedOption ? selectedOption.label : placeholder}
        </span>
        <span className="custom-select-arrow">▼</span>
      </div>
      
      {dropdownElement && createPortal(dropdownElement, document.body)}
    </>
  );
}

// Usage in constrained layout
function FormInSidebar() {
  const [country, setCountry] = useState('');
  
  const countries = [
    { value: 'us', label: 'United States' },
    { value: 'ca', label: 'Canada' },
    { value: 'uk', label: 'United Kingdom' },
    { value: 'de', label: 'Germany' },
    { value: 'fr', label: 'France' }
  ];

  return (
    <aside className="sidebar" style={{ overflow: 'hidden', width: '250px' }}>
      <form>
        <label>
          Country:
          <CustomSelect
            options={countries}
            value={country}
            onChange={(option) => setCountry(option.value)}
            placeholder="Select country..."
          />
        </label>
        {/* More form fields... */}
      </form>
    </aside>
  );
}
```

### 4. Toast Notifications

**Why Portals**: Notifications should appear at a consistent location (usually top or bottom of screen) regardless of where they're triggered from in the component tree.

```jsx
// Toast notification system with portal
const ToastContext = createContext();

function ToastProvider({ children }) {
  const [toasts, setToasts] = useState([]);

  const addToast = useCallback((toast) => {
    const id = Date.now();
    const newToast = {
      id,
      type: 'info',
      duration: 5000,
      ...toast
    };
    
    setToasts(prev => [...prev, newToast]);
    
    // Auto remove after duration
    if (newToast.duration > 0) {
      setTimeout(() => {
        removeToast(id);
      }, newToast.duration);
    }
    
    return id;
  }, []);

  const removeToast = useCallback((id) => {
    setToasts(prev => prev.filter(toast => toast.id !== id));
  }, []);

  const clearAllToasts = useCallback(() => {
    setToasts([]);
  }, []);

  return (
    <ToastContext.Provider value={{ 
      toasts, 
      addToast, 
      removeToast, 
      clearAllToasts 
    }}>
      {children}
      <ToastContainer toasts={toasts} onRemove={removeToast} />
    </ToastContext.Provider>
  );
}

function ToastContainer({ toasts, onRemove }) {
  const containerRef = useRef(null);

  useEffect(() => {
    // Create toast container if it doesn't exist
    let container = document.getElementById('toast-root');
    if (!container) {
      container = document.createElement('div');
      container.id = 'toast-root';
      document.body.appendChild(container);
    }
    containerRef.current = container;
  }, []);

  if (!containerRef.current) return null;

  return createPortal(
    <div className="toast-container">
      <AnimatePresence>
        {toasts.map(toast => (
          <Toast
            key={toast.id}
            toast={toast}
            onRemove={() => onRemove(toast.id)}
          />
        ))}
      </AnimatePresence>
    </div>,
    containerRef.current
  );
}

function Toast({ toast, onRemove }) {
  const [isRemoving, setIsRemoving] = useState(false);

  const handleRemove = () => {
    setIsRemoving(true);
    setTimeout(onRemove, 300); // Wait for exit animation
  };

  useEffect(() => {
    // Auto-close functionality
    if (toast.duration > 0) {
      const timer = setTimeout(handleRemove, toast.duration);
      return () => clearTimeout(timer);
    }
  }, [toast.duration]);

  return (
    <div
      className={`toast toast--${toast.type} ${isRemoving ? 'removing' : ''}`}
      role="alert"
      aria-live="polite"
    >
      <div className="toast-content">
        {toast.title && (
          <div className="toast-title">{toast.title}</div>
        )}
        <div className="toast-message">{toast.message}</div>
      </div>
      
      <button
        className="toast-close"
        onClick={handleRemove}
        aria-label="Close notification"
      >
        ×
      </button>
    </div>
  );
}

// Custom hook for using toasts
function useToast() {
  const context = useContext(ToastContext);
  if (!context) {
    throw new Error('useToast must be used within ToastProvider');
  }
  return context;
}

// Usage anywhere in the app
function SaveButton({ onSave }) {
  const { addToast } = useToast();

  const handleSave = async () => {
    try {
      await onSave();
      addToast({
        type: 'success',
        title: 'Success!',
        message: 'Your changes have been saved.',
        duration: 3000
      });
    } catch (error) {
      addToast({
        type: 'error',
        title: 'Error',
        message: 'Failed to save changes. Please try again.',
        duration: 5000
      });
    }
  };

  return <button onClick={handleSave}>Save</button>;
}
```

### 5. Context Menus

**Why Portals**: Context menus appear at cursor position and must not be clipped by parent containers.

```jsx
function useContextMenu() {
  const [isVisible, setIsVisible] = useState(false);
  const [position, setPosition] = useState({ x: 0, y: 0 });
  const [menuContent, setMenuContent] = useState(null);

  const showContextMenu = useCallback((event, content) => {
    event.preventDefault();
    
    const { clientX, clientY } = event;
    setPosition({ x: clientX, y: clientY });
    setMenuContent(content);
    setIsVisible(true);
  }, []);

  const hideContextMenu = useCallback(() => {
    setIsVisible(false);
    setMenuContent(null);
  }, []);

  // Click outside or escape key handling
  useEffect(() => {
    if (isVisible) {
      const handleClick = () => hideContextMenu();
      const handleKeyDown = (event) => {
        if (event.key === 'Escape') {
          hideContextMenu();
        }
      };

      document.addEventListener('click', handleClick);
      document.addEventListener('keydown', handleKeyDown);
      
      return () => {
        document.removeEventListener('click', handleClick);
        document.removeEventListener('keydown', handleKeyDown);
      };
    }
  }, [isVisible, hideContextMenu]);

  return {
    isVisible,
    position,
    menuContent,
    showContextMenu,
    hideContextMenu
  };
}

function ContextMenu({ isVisible, position, children }) {
  const menuRef = useRef(null);
  const [adjustedPosition, setAdjustedPosition] = useState(position);

  useEffect(() => {
    if (isVisible && menuRef.current) {
      const menu = menuRef.current;
      const menuRect = menu.getBoundingClientRect();
      const viewport = {
        width: window.innerWidth,
        height: window.innerHeight
      };

      let { x, y } = position;

      // Adjust position if menu would go off-screen
      if (x + menuRect.width > viewport.width - 10) {
        x = viewport.width - menuRect.width - 10;
      }
      if (y + menuRect.height > viewport.height - 10) {
        y = viewport.height - menuRect.height - 10;
      }

      setAdjustedPosition({ x, y });
    }
  }, [isVisible, position]);

  if (!isVisible) return null;

  return createPortal(
    <div
      ref={menuRef}
      className="context-menu"
      style={{
        position: 'fixed',
        top: adjustedPosition.y,
        left: adjustedPosition.x,
        zIndex: 10000
      }}
      onClick={(e) => e.stopPropagation()}
    >
      {children}
    </div>,
    document.body
  );
}

// Usage example
function FileExplorer({ files }) {
  const { 
    isVisible, 
    position, 
    menuContent, 
    showContextMenu, 
    hideContextMenu 
  } = useContextMenu();

  const handleFileContextMenu = (event, file) => {
    const menuItems = (
      <div className="context-menu-items">
        <button onClick={() => console.log('Open', file.name)}>
          Open
        </button>
        <button onClick={() => console.log('Rename', file.name)}>
          Rename
        </button>
        <button onClick={() => console.log('Delete', file.name)}>
          Delete
        </button>
        <hr />
        <button onClick={() => console.log('Properties', file.name)}>
          Properties
        </button>
      </div>
    );
    
    showContextMenu(event, menuItems);
  };

  return (
    <div className="file-explorer">
      {files.map(file => (
        <div
          key={file.id}
          className="file-item"
          onContextMenu={(e) => handleFileContextMenu(e, file)}
        >
          <span>{file.name}</span>
        </div>
      ))}
      
      <ContextMenu 
        isVisible={isVisible} 
        position={position}
      >
        {menuContent}
      </ContextMenu>
    </div>
  );
}
```

---

## When NOT to Use Portals

### 1. Simple Component Communication
Don't use portals when regular props or context would suffice:

```jsx
// ❌ Unnecessary portal usage
function Parent() {
  return (
    <div>
      <Child />
      {createPortal(<Notification />, document.body)}
    </div>
  );
}

// ✅ Use props or context instead
function Parent() {
  const [notification, setNotification] = useState(null);
  
  return (
    <div>
      <Child onNotify={setNotification} />
      {notification && <Notification {...notification} />}
    </div>
  );
}
```

### 2. Regular Layout Components
Don't use portals for components that should be part of normal document flow:

```jsx
// ❌ Unnecessary portal for regular content
function Article({ children }) {
  return createPortal(
    <article>{children}</article>,
    document.body
  );
}

// ✅ Regular component
function Article({ children }) {
  return <article>{children}</article>;
}
```

### 3. Components That Don't Need Special Positioning
Only use portals when you specifically need to escape parent containers' styling constraints.

---

## Best Practices

### 1. Clean Up Portal Containers
```jsx
function usePortalContainer(id) {
  const [container, setContainer] = useState(null);

  useEffect(() => {
    let element = document.getElementById(id);
    
    if (!element) {
      element = document.createElement('div');
      element.id = id;
      document.body.appendChild(element);
    }
    
    setContainer(element);
    
    return () => {
      // Clean up if no children
      if (element && element.childNodes.length === 0) {
        document.body.removeChild(element);
      }
    };
  }, [id]);

  return container;
}
```

### 2. Handle Focus Management
```jsx
function Modal({ isOpen, onClose, children }) {
  const modalRef = useRef(null);
  const previousActiveElement = useRef(null);

  useEffect(() => {
    if (isOpen) {
      previousActiveElement.current = document.activeElement;
      
      // Focus the modal after render
      requestAnimationFrame(() => {
        if (modalRef.current) {
          modalRef.current.focus();
        }
      });
    }
    
    return () => {
      if (previousActiveElement.current) {
        previousActiveElement.current.focus();
      }
    };
  }, [isOpen]);

  if (!isOpen) return null;

  return createPortal(
    <div 
      ref={modalRef}
      className="modal"
      tabIndex={-1}
      role="dialog"
      aria-modal="true"
    >
      {children}
    </div>,
    document.body
  );
}
```

### 3. Event Handling Considerations
Remember that React events bubble through the React tree, not the DOM tree:

```jsx
function App() {
  const handleClick = () => {
    console.log('App clicked'); // This will fire even for portal content
  };

  return (
    <div onClick={handleClick}>
      <button>Regular button</button>
      
      {createPortal(
        <button>Portal button</button>, // Clicking this will also trigger handleClick
        document.body
      )}
    </div>
  );
}
```

---

## Interview-ready summary

**React Portals** are essential when you need to render components outside their parent's DOM hierarchy while maintaining React's component tree relationships. Primary use cases include **modals** (escape z-index stacking), **tooltips** (avoid clipping by overflow:hidden), **dropdowns** (position above other content), **toast notifications** (consistent global positioning), and **context menus** (cursor-based positioning). Portals solve CSS containment issues, allow proper layering with z-index, and enable components to render at optimal DOM locations. Key considerations include cleanup of portal containers, focus management for accessibility, understanding event bubbling behavior, and performance implications. Avoid portals for regular layout components or simple communication - use them specifically when you need to escape parent container constraints.

