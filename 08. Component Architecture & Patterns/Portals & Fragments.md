# Portals & Fragments

This file covers React Portals for rendering outside the component tree and Fragments for grouping elements without extra DOM nodes.
Essential for advanced component composition and DOM manipulation.

---

## React Fragments

### What are Fragments?
Fragments let you group multiple elements without adding extra nodes to the DOM. They solve the problem of React components needing to return a single parent element.

### Basic Fragment usage

```jsx
// ❌ Without Fragments - creates unnecessary div
function UserInfo({ user }) {
  return (
    <div> {/* Extra wrapper div */}
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <p>{user.role}</p>
    </div>
  );
}

// ✅ With Fragments - no extra DOM node
function UserInfo({ user }) {
  return (
    <React.Fragment>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <p>{user.role}</p>
    </React.Fragment>
  );
}

// ✅ Short syntax
function UserInfo({ user }) {
  return (
    <>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <p>{user.role}</p>
    </>
  );
}
```

### Fragments with keys

```jsx
// When you need to provide a key (e.g., in lists)
function DefinitionList({ items }) {
  return (
    <dl>
      {items.map(item => (
        <React.Fragment key={item.id}>
          <dt>{item.term}</dt>
          <dd>{item.definition}</dd>
        </React.Fragment>
      ))}
    </dl>
  );
}

// Real-world example: Table rows with multiple cells
function UserTable({ users }) {
  return (
    <table>
      <tbody>
        {users.map(user => (
          <React.Fragment key={user.id}>
            <tr>
              <td>{user.name}</td>
              <td>{user.email}</td>
              <td>{user.role}</td>
            </tr>
            {user.isExpanded && (
              <tr>
                <td colSpan="3">
                  <UserDetails user={user} />
                </td>
              </tr>
            )}
          </React.Fragment>
        ))}
      </tbody>
    </table>
  );
}
```

### Complex Fragment scenarios

```jsx
// Conditional fragments
function ConditionalContent({ showHeader, showFooter, children }) {
  return (
    <div className="content">
      {showHeader && (
        <>
          <header>Content Header</header>
          <hr />
        </>
      )}
      
      <main>{children}</main>
      
      {showFooter && (
        <>
          <hr />
          <footer>Content Footer</footer>
        </>
      )}
    </div>
  );
}

// Fragments in component arrays
function ComponentList({ components }) {
  return (
    <div>
      {components.map((Component, index) => (
        <React.Fragment key={index}>
          <Component />
          {index < components.length - 1 && <hr />}
        </React.Fragment>
      ))}
    </div>
  );
}

// Advanced list rendering with separators
function TagList({ tags, separator = ', ' }) {
  return (
    <span className="tag-list">
      {tags.map((tag, index) => (
        <React.Fragment key={tag.id}>
          <span className="tag">{tag.name}</span>
          {index < tags.length - 1 && (
            <span className="separator">{separator}</span>
          )}
        </React.Fragment>
      ))}
    </span>
  );
}
```

---

## React Portals

### What are Portals?
Portals provide a way to render children into a DOM node that exists outside the parent component's DOM hierarchy while maintaining React's event bubbling and context.

### Basic Portal usage

```jsx
import { createPortal } from 'react-dom';

// Basic portal to document.body
function Portal({ children }) {
  return createPortal(children, document.body);
}

// Portal to specific DOM element
function PortalToElement({ children, elementId = 'portal-root' }) {
  const portalElement = document.getElementById(elementId);
  
  if (!portalElement) {
    console.warn(`Portal element with id '${elementId}' not found`);
    return null;
  }
  
  return createPortal(children, portalElement);
}

// Usage
function App() {
  return (
    <div>
      <h1>Main App</h1>
      <Portal>
        <div>This renders in document.body</div>
      </Portal>
    </div>
  );
}
```

### Modal implementation with Portals

```jsx
// Modal component using Portal
function Modal({ isOpen, onClose, children, className = '' }) {
  const [modalRoot, setModalRoot] = useState(null);

  useEffect(() => {
    // Create or find modal root
    let root = document.getElementById('modal-root');
    if (!root) {
      root = document.createElement('div');
      root.id = 'modal-root';
      document.body.appendChild(root);
    }
    setModalRoot(root);

    // Cleanup function
    return () => {
      if (root && root.childNodes.length === 0) {
        document.body.removeChild(root);
      }
    };
  }, []);

  useEffect(() => {
    if (isOpen) {
      // Prevent body scroll when modal is open
      document.body.style.overflow = 'hidden';
    } else {
      document.body.style.overflow = 'unset';
    }

    // Cleanup
    return () => {
      document.body.style.overflow = 'unset';
    };
  }, [isOpen]);

  // Handle ESC key
  useEffect(() => {
    const handleEscape = (e) => {
      if (e.key === 'Escape') {
        onClose();
      }
    };

    if (isOpen) {
      document.addEventListener('keydown', handleEscape);
    }

    return () => {
      document.removeEventListener('keydown', handleEscape);
    };
  }, [isOpen, onClose]);

  if (!isOpen || !modalRoot) {
    return null;
  }

  return createPortal(
    <div className={`modal-overlay ${className}`}>
      <div 
        className="modal-backdrop" 
        onClick={onClose}
        aria-hidden="true"
      />
      <div 
        className="modal-content"
        role="dialog"
        aria-modal="true"
        onClick={(e) => e.stopPropagation()}
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

// Usage
function App() {
  const [showModal, setShowModal] = useState(false);

  return (
    <div>
      <button onClick={() => setShowModal(true)}>
        Open Modal
      </button>
      
      <Modal 
        isOpen={showModal} 
        onClose={() => setShowModal(false)}
      >
        <h2>Modal Title</h2>
        <p>Modal content goes here...</p>
        <button onClick={() => setShowModal(false)}>
          Close
        </button>
      </Modal>
    </div>
  );
}
```

### Tooltip implementation with Portals

```jsx
// Advanced tooltip with portal and positioning
function Tooltip({ 
  children, 
  content, 
  position = 'top', 
  delay = 200,
  className = '' 
}) {
  const [isVisible, setIsVisible] = useState(false);
  const [tooltipStyle, setTooltipStyle] = useState({});
  const triggerRef = useRef(null);
  const tooltipRef = useRef(null);
  const timeoutRef = useRef(null);

  const showTooltip = () => {
    if (timeoutRef.current) {
      clearTimeout(timeoutRef.current);
    }
    
    timeoutRef.current = setTimeout(() => {
      setIsVisible(true);
      updateTooltipPosition();
    }, delay);
  };

  const hideTooltip = () => {
    if (timeoutRef.current) {
      clearTimeout(timeoutRef.current);
    }
    setIsVisible(false);
  };

  const updateTooltipPosition = () => {
    if (!triggerRef.current || !tooltipRef.current) return;

    const triggerRect = triggerRef.current.getBoundingClientRect();
    const tooltipRect = tooltipRef.current.getBoundingClientRect();
    const scrollTop = window.pageYOffset;
    const scrollLeft = window.pageXOffset;

    let top, left;

    switch (position) {
      case 'top':
        top = triggerRect.top + scrollTop - tooltipRect.height - 8;
        left = triggerRect.left + scrollLeft + (triggerRect.width - tooltipRect.width) / 2;
        break;
      case 'bottom':
        top = triggerRect.bottom + scrollTop + 8;
        left = triggerRect.left + scrollLeft + (triggerRect.width - tooltipRect.width) / 2;
        break;
      case 'left':
        top = triggerRect.top + scrollTop + (triggerRect.height - tooltipRect.height) / 2;
        left = triggerRect.left + scrollLeft - tooltipRect.width - 8;
        break;
      case 'right':
        top = triggerRect.top + scrollTop + (triggerRect.height - tooltipRect.height) / 2;
        left = triggerRect.right + scrollLeft + 8;
        break;
      default:
        top = triggerRect.top + scrollTop - tooltipRect.height - 8;
        left = triggerRect.left + scrollLeft + (triggerRect.width - tooltipRect.width) / 2;
    }

    setTooltipStyle({
      position: 'absolute',
      top: `${top}px`,
      left: `${left}px`,
      zIndex: 9999
    });
  };

  useEffect(() => {
    if (isVisible) {
      updateTooltipPosition();
      
      const handleResize = () => updateTooltipPosition();
      const handleScroll = () => updateTooltipPosition();
      
      window.addEventListener('resize', handleResize);
      window.addEventListener('scroll', handleScroll);
      
      return () => {
        window.removeEventListener('resize', handleResize);
        window.removeEventListener('scroll', handleScroll);
      };
    }
  }, [isVisible]);

  const tooltipElement = isVisible && content ? (
    <div
      ref={tooltipRef}
      className={`tooltip tooltip--${position} ${className}`}
      style={tooltipStyle}
      role="tooltip"
    >
      {content}
      <div className={`tooltip-arrow tooltip-arrow--${position}`} />
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

// Usage
function App() {
  return (
    <div>
      <Tooltip content="This is a helpful tooltip" position="top">
        <button>Hover me</button>
      </Tooltip>
      
      <Tooltip 
        content={
          <div>
            <strong>Rich tooltip content</strong>
            <p>Can contain any JSX!</p>
          </div>
        } 
        position="bottom"
      >
        <span>Rich tooltip trigger</span>
      </Tooltip>
    </div>
  );
}
```

### Dropdown/Popover with Portals

```jsx
// Advanced dropdown with portal and click outside detection
function Dropdown({ 
  trigger, 
  children, 
  placement = 'bottom-start',
  offset = 8,
  className = ''
}) {
  const [isOpen, setIsOpen] = useState(false);
  const [dropdownStyle, setDropdownStyle] = useState({});
  const triggerRef = useRef(null);
  const dropdownRef = useRef(null);

  // Click outside detection
  useEffect(() => {
    const handleClickOutside = (event) => {
      if (
        dropdownRef.current && 
        !dropdownRef.current.contains(event.target) &&
        triggerRef.current && 
        !triggerRef.current.contains(event.target)
      ) {
        setIsOpen(false);
      }
    };

    if (isOpen) {
      document.addEventListener('mousedown', handleClickOutside);
    }

    return () => {
      document.removeEventListener('mousedown', handleClickOutside);
    };
  }, [isOpen]);

  // Position calculation
  const updatePosition = useCallback(() => {
    if (!triggerRef.current || !dropdownRef.current) return;

    const triggerRect = triggerRef.current.getBoundingClientRect();
    const dropdownRect = dropdownRef.current.getBoundingClientRect();
    const viewport = {
      width: window.innerWidth,
      height: window.innerHeight
    };

    let top, left;

    // Calculate position based on placement
    switch (placement) {
      case 'bottom-start':
        top = triggerRect.bottom + offset;
        left = triggerRect.left;
        break;
      case 'bottom-end':
        top = triggerRect.bottom + offset;
        left = triggerRect.right - dropdownRect.width;
        break;
      case 'top-start':
        top = triggerRect.top - dropdownRect.height - offset;
        left = triggerRect.left;
        break;
      case 'top-end':
        top = triggerRect.top - dropdownRect.height - offset;
        left = triggerRect.right - dropdownRect.width;
        break;
      case 'right-start':
        top = triggerRect.top;
        left = triggerRect.right + offset;
        break;
      case 'left-start':
        top = triggerRect.top;
        left = triggerRect.left - dropdownRect.width - offset;
        break;
      default:
        top = triggerRect.bottom + offset;
        left = triggerRect.left;
    }

    // Viewport collision detection and adjustment
    if (left < 0) left = 8;
    if (left + dropdownRect.width > viewport.width) {
      left = viewport.width - dropdownRect.width - 8;
    }
    if (top < 0) top = 8;
    if (top + dropdownRect.height > viewport.height) {
      top = triggerRect.top - dropdownRect.height - offset;
    }

    setDropdownStyle({
      position: 'fixed',
      top: `${top}px`,
      left: `${left}px`,
      zIndex: 1000
    });
  }, [placement, offset]);

  useEffect(() => {
    if (isOpen) {
      updatePosition();
      
      const handleResize = () => updatePosition();
      const handleScroll = () => updatePosition();
      
      window.addEventListener('resize', handleResize);
      window.addEventListener('scroll', handleScroll);
      
      return () => {
        window.removeEventListener('resize', handleResize);
        window.removeEventListener('scroll', handleScroll);
      };
    }
  }, [isOpen, updatePosition]);

  const handleTriggerClick = () => {
    setIsOpen(!isOpen);
  };

  const dropdownElement = isOpen ? (
    <div
      ref={dropdownRef}
      className={`dropdown ${className}`}
      style={dropdownStyle}
    >
      {children}
    </div>
  ) : null;

  return (
    <>
      <div ref={triggerRef} onClick={handleTriggerClick}>
        {trigger}
      </div>
      {dropdownElement && createPortal(dropdownElement, document.body)}
    </>
  );
}

// Usage
function NavigationMenu() {
  return (
    <nav>
      <Dropdown
        trigger={
          <button className="nav-button">
            User Menu ▼
          </button>
        }
        placement="bottom-start"
      >
        <div className="dropdown-menu">
          <a href="/profile" className="dropdown-item">Profile</a>
          <a href="/settings" className="dropdown-item">Settings</a>
          <hr className="dropdown-divider" />
          <a href="/logout" className="dropdown-item">Logout</a>
        </div>
      </Dropdown>
    </nav>
  );
}
```

---

## Advanced Portal Patterns

### Portal Manager for Multiple Portals

```jsx
// Portal manager context
const PortalContext = createContext();

function PortalProvider({ children }) {
  const [portals, setPortals] = useState(new Map());

  const createPortal = useCallback((id, element) => {
    setPortals(prev => new Map(prev).set(id, element));
  }, []);

  const removePortal = useCallback((id) => {
    setPortals(prev => {
      const newPortals = new Map(prev);
      newPortals.delete(id);
      return newPortals;
    });
  }, []);

  const getPortal = useCallback((id) => {
    return portals.get(id);
  }, [portals]);

  return (
    <PortalContext.Provider value={{ 
      createPortal, 
      removePortal, 
      getPortal,
      portals 
    }}>
      {children}
      {/* Render all portals */}
      <div id="portal-container">
        {Array.from(portals.entries()).map(([id, element]) => (
          <div key={id} data-portal-id={id}>
            {element}
          </div>
        ))}
      </div>
    </PortalContext.Provider>
  );
}

// Custom hook for using portals
function usePortal(id) {
  const context = useContext(PortalContext);
  if (!context) {
    throw new Error('usePortal must be used within PortalProvider');
  }
  
  const { createPortal, removePortal, getPortal } = context;
  
  const addToPortal = useCallback((element) => {
    createPortal(id, element);
  }, [id, createPortal]);
  
  const removeFromPortal = useCallback(() => {
    removePortal(id);
  }, [id, removePortal]);
  
  useEffect(() => {
    return () => {
      removeFromPortal();
    };
  }, [removeFromPortal]);
  
  return { addToPortal, removeFromPortal, portal: getPortal(id) };
}
```

### Dynamic Portal Creation

```jsx
// Hook for creating dynamic portals
function useDynamicPortal(containerId) {
  const [portalElement, setPortalElement] = useState(null);

  useEffect(() => {
    // Create or find container
    let container = document.getElementById(containerId);
    
    if (!container) {
      container = document.createElement('div');
      container.id = containerId;
      document.body.appendChild(container);
    }

    setPortalElement(container);

    // Cleanup
    return () => {
      if (container && container.childNodes.length === 0) {
        document.body.removeChild(container);
      }
    };
  }, [containerId]);

  const Portal = useCallback(({ children }) => {
    if (!portalElement) return null;
    return createPortal(children, portalElement);
  }, [portalElement]);

  return Portal;
}

// Usage
function DynamicModalExample() {
  const [modals, setModals] = useState([]);
  const Portal = useDynamicPortal('dynamic-modals');

  const addModal = () => {
    const newModal = {
      id: Date.now(),
      title: `Modal ${modals.length + 1}`
    };
    setModals(prev => [...prev, newModal]);
  };

  const removeModal = (id) => {
    setModals(prev => prev.filter(modal => modal.id !== id));
  };

  return (
    <div>
      <button onClick={addModal}>Add Modal</button>
      
      <Portal>
        {modals.map(modal => (
          <div key={modal.id} className="dynamic-modal">
            <h3>{modal.title}</h3>
            <button onClick={() => removeModal(modal.id)}>
              Close
            </button>
          </div>
        ))}
      </Portal>
    </div>
  );
}
```

---

## Best Practices

### Fragments
* Use short syntax `<>` when you don't need keys
* Use `<React.Fragment key={...}>` when rendering lists
* Prefer fragments over unnecessary wrapper divs
* Be mindful of CSS flexbox/grid layouts that might need wrapper elements

### Portals
* Always clean up portal containers when unmounting
* Handle escape key and click-outside for modals
* Consider accessibility (focus management, ARIA attributes)
* Be careful with event bubbling - events bubble through React tree, not DOM tree
* Use portals sparingly - they can make debugging harder

### Performance Considerations
```jsx
// ✅ Good: Memoize portal content
const ModalContent = memo(({ children, onClose }) => (
  <div className="modal">
    <button onClick={onClose}>×</button>
    {children}
  </div>
));

// ✅ Good: Lazy create portal containers
function usePortalContainer(id) {
  return useMemo(() => {
    let container = document.getElementById(id);
    if (!container) {
      container = document.createElement('div');
      container.id = id;
      document.body.appendChild(container);
    }
    return container;
  }, [id]);
}
```

---

## Interview-ready summary

**Fragments** group multiple elements without adding extra DOM nodes, using `<>` shorthand or `<React.Fragment>` with keys. They solve React's requirement for single parent elements and improve DOM structure cleanliness. **Portals** render children into DOM nodes outside the parent component hierarchy using `createPortal(child, container)`. Common use cases include modals, tooltips, dropdowns, and notifications. Portals maintain React's event bubbling and context while rendering elsewhere in the DOM. Key considerations include cleanup, accessibility, positioning, and event handling. Both features improve component composition and DOM management without compromising React's data flow.
