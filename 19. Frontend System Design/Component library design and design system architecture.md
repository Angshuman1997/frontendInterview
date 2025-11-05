# Component library design and design system architecture

Building scalable component libraries and design systems is crucial for large applications and teams. This covers the architecture, patterns, and best practices for creating maintainable, flexible, and reusable component ecosystems.

## Design System Foundation

### 1. Design Tokens Architecture

**Token Hierarchy:**
```typescript
// Design tokens structure
interface DesignTokens {
  colors: {
    primitive: PrimitiveColors;
    semantic: SemanticColors;
    component: ComponentColors;
  };
  typography: {
    fontFamilies: FontFamilies;
    fontSizes: FontSizes;
    fontWeights: FontWeights;
    lineHeights: LineHeights;
  };
  spacing: SpacingScale;
  breakpoints: Breakpoints;
  shadows: Shadows;
  borders: Borders;
  motion: MotionTokens;
}

// Primitive tokens (raw values)
interface PrimitiveColors {
  blue: {
    50: '#eff6ff';
    100: '#dbeafe';
    500: '#3b82f6';
    900: '#1e3a8a';
  };
  gray: {
    50: '#f9fafb';
    100: '#f3f4f6';
    500: '#6b7280';
    900: '#111827';
  };
}

// Semantic tokens (purpose-based)
interface SemanticColors {
  background: {
    primary: string;
    secondary: string;
    tertiary: string;
  };
  text: {
    primary: string;
    secondary: string;
    disabled: string;
  };
  interactive: {
    primary: string;
    secondary: string;
    danger: string;
    success: string;
  };
  border: {
    default: string;
    subtle: string;
    strong: string;
  };
}

// Component-specific tokens
interface ComponentColors {
  button: {
    primary: {
      background: string;
      backgroundHover: string;
      backgroundActive: string;
      text: string;
      border: string;
    };
    secondary: {
      background: string;
      backgroundHover: string;
      text: string;
      border: string;
    };
  };
  input: {
    background: string;
    backgroundFocus: string;
    border: string;
    borderFocus: string;
    text: string;
    placeholder: string;
  };
}

// Token implementation with CSS custom properties
const createCSSTokens = (tokens: DesignTokens) => {
  const cssVariables: Record<string, string> = {};
  
  const flatten = (obj: any, prefix = '') => {
    Object.entries(obj).forEach(([key, value]) => {
      const cssKey = `--${prefix}${prefix ? '-' : ''}${key.replace(/([A-Z])/g, '-$1').toLowerCase()}`;
      
      if (typeof value === 'object' && value !== null) {
        flatten(value, `${prefix}${prefix ? '-' : ''}${key}`);
      } else {
        cssVariables[cssKey] = String(value);
      }
    });
  };
  
  flatten(tokens);
  return cssVariables;
};

// Theme provider implementation
const ThemeProvider: React.FC<{
  theme: DesignTokens;
  children: React.ReactNode;
}> = ({ theme, children }) => {
  useEffect(() => {
    const cssTokens = createCSSTokens(theme);
    
    Object.entries(cssTokens).forEach(([property, value]) => {
      document.documentElement.style.setProperty(property, value);
    });
    
    return () => {
      Object.keys(cssTokens).forEach(property => {
        document.documentElement.style.removeProperty(property);
      });
    };
  }, [theme]);

  return (
    <ThemeContext.Provider value={theme}>
      {children}
    </ThemeContext.Provider>
  );
};
```

### 2. Component API Design

**Consistent Component Interface:**
```typescript
// Base component props interface
interface BaseComponentProps {
  /** Unique identifier for testing */
  'data-testid'?: string;
  /** Additional CSS class names */
  className?: string;
  /** Inline styles (use sparingly) */
  style?: React.CSSProperties;
  /** Component size variant */
  size?: 'small' | 'medium' | 'large';
  /** Visual variant */
  variant?: string;
  /** Disabled state */
  disabled?: boolean;
  /** Loading state */
  loading?: boolean;
  /** ARIA label for accessibility */
  'aria-label'?: string;
  /** ARIA described by */
  'aria-describedby'?: string;
}

// Button component with comprehensive API
interface ButtonProps extends BaseComponentProps, 
  Omit<React.ButtonHTMLAttributes<HTMLButtonElement>, 'size'> {
  /** Button visual variant */
  variant?: 'primary' | 'secondary' | 'tertiary' | 'danger' | 'success';
  /** Button size */
  size?: 'small' | 'medium' | 'large';
  /** Full width button */
  fullWidth?: boolean;
  /** Icon to display before text */
  startIcon?: React.ReactNode;
  /** Icon to display after text */
  endIcon?: React.ReactNode;
  /** Button content */
  children: React.ReactNode;
}

// Implementation with forwarded refs
const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(({
  variant = 'primary',
  size = 'medium',
  fullWidth = false,
  startIcon,
  endIcon,
  disabled = false,
  loading = false,
  className,
  children,
  'data-testid': testId,
  ...props
}, ref) => {
  const classes = cn(
    'btn',
    `btn--${variant}`,
    `btn--${size}`,
    {
      'btn--full-width': fullWidth,
      'btn--disabled': disabled,
      'btn--loading': loading,
    },
    className
  );

  return (
    <button
      ref={ref}
      className={classes}
      disabled={disabled || loading}
      data-testid={testId}
      aria-disabled={disabled || loading}
      {...props}
    >
      {loading && <Spinner size="small" />}
      {!loading && startIcon && (
        <span className="btn__start-icon">{startIcon}</span>
      )}
      <span className="btn__content">{children}</span>
      {!loading && endIcon && (
        <span className="btn__end-icon">{endIcon}</span>
      )}
    </button>
  );
});

Button.displayName = 'Button';
```

## Component Architecture Patterns

### 1. Compound Components

**Tab Component Example:**
```typescript
// Compound component pattern for flexible composition
interface TabsContextValue {
  activeTab: string;
  setActiveTab: (tab: string) => void;
  orientation: 'horizontal' | 'vertical';
}

const TabsContext = createContext<TabsContextValue | null>(null);

const useTabs = () => {
  const context = useContext(TabsContext);
  if (!context) {
    throw new Error('Tab components must be used within Tabs');
  }
  return context;
};

// Root Tabs component
interface TabsProps {
  defaultTab?: string;
  orientation?: 'horizontal' | 'vertical';
  children: React.ReactNode;
  onTabChange?: (tab: string) => void;
}

const Tabs: React.FC<TabsProps> & {
  List: typeof TabList;
  Tab: typeof Tab;
  Panels: typeof TabPanels;
  Panel: typeof TabPanel;
} = ({ 
  defaultTab, 
  orientation = 'horizontal', 
  children, 
  onTabChange 
}) => {
  const [activeTab, setActiveTab] = useState(defaultTab || '');

  const handleTabChange = (tab: string) => {
    setActiveTab(tab);
    onTabChange?.(tab);
  };

  return (
    <TabsContext.Provider value={{
      activeTab,
      setActiveTab: handleTabChange,
      orientation,
    }}>
      <div className={`tabs tabs--${orientation}`}>
        {children}
      </div>
    </TabsContext.Provider>
  );
};

// Tab List component
const TabList: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const { orientation } = useTabs();
  
  return (
    <div 
      className="tabs__list" 
      role="tablist"
      aria-orientation={orientation}
    >
      {children}
    </div>
  );
};

// Individual Tab component
interface TabProps {
  value: string;
  children: React.ReactNode;
  disabled?: boolean;
}

const Tab: React.FC<TabProps> = ({ value, children, disabled = false }) => {
  const { activeTab, setActiveTab } = useTabs();
  const isActive = activeTab === value;

  return (
    <button
      className={cn('tabs__tab', {
        'tabs__tab--active': isActive,
        'tabs__tab--disabled': disabled,
      })}
      role="tab"
      aria-selected={isActive}
      aria-controls={`panel-${value}`}
      id={`tab-${value}`}
      disabled={disabled}
      onClick={() => !disabled && setActiveTab(value)}
    >
      {children}
    </button>
  );
};

// Tab Panels container
const TabPanels: React.FC<{ children: React.ReactNode }> = ({ children }) => (
  <div className="tabs__panels">
    {children}
  </div>
);

// Individual Tab Panel
interface TabPanelProps {
  value: string;
  children: React.ReactNode;
}

const TabPanel: React.FC<TabPanelProps> = ({ value, children }) => {
  const { activeTab } = useTabs();
  const isActive = activeTab === value;

  if (!isActive) return null;

  return (
    <div
      className="tabs__panel"
      role="tabpanel"
      id={`panel-${value}`}
      aria-labelledby={`tab-${value}`}
    >
      {children}
    </div>
  );
};

// Attach sub-components
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panels = TabPanels;
Tabs.Panel = TabPanel;

// Usage
const Example: React.FC = () => (
  <Tabs defaultTab="profile">
    <Tabs.List>
      <Tabs.Tab value="profile">Profile</Tabs.Tab>
      <Tabs.Tab value="settings">Settings</Tabs.Tab>
      <Tabs.Tab value="billing">Billing</Tabs.Tab>
    </Tabs.List>
    
    <Tabs.Panels>
      <Tabs.Panel value="profile">
        <ProfileForm />
      </Tabs.Panel>
      <Tabs.Panel value="settings">
        <SettingsForm />
      </Tabs.Panel>
      <Tabs.Panel value="billing">
        <BillingInfo />
      </Tabs.Panel>
    </Tabs.Panels>
  </Tabs>
);
```

### 2. Render Props / Polymorphic Components

**Polymorphic Button:**
```typescript
// Polymorphic component that can render as different elements
type AsProp<C extends React.ElementType> = {
  as?: C;
};

type PropsToOmit<C extends React.ElementType, P> = keyof (AsProp<C> & P);

type PolymorphicComponentProp<
  C extends React.ElementType,
  Props = {}
> = React.PropsWithChildren<Props & AsProp<C>> &
  Omit<React.ComponentPropsWithoutRef<C>, PropsToOmit<C, Props>>;

type PolymorphicRef<C extends React.ElementType> =
  React.ComponentPropsWithRef<C>['ref'];

type PolymorphicComponentPropWithRef<
  C extends React.ElementType,
  Props = {}
> = PolymorphicComponentProp<C, Props> & { ref?: PolymorphicRef<C> };

interface ButtonOwnProps {
  variant?: 'primary' | 'secondary';
  size?: 'small' | 'medium' | 'large';
}

type ButtonProps<C extends React.ElementType> = 
  PolymorphicComponentPropWithRef<C, ButtonOwnProps>;

type ButtonComponent = <C extends React.ElementType = 'button'>(
  props: ButtonProps<C>
) => React.ReactElement | null;

const Button: ButtonComponent = React.forwardRef(
  <C extends React.ElementType = 'button'>(
    {
      as,
      variant = 'primary',
      size = 'medium',
      className,
      children,
      ...props
    }: ButtonProps<C>,
    ref?: PolymorphicRef<C>
  ) => {
    const Component = as || 'button';
    
    const classes = cn(
      'btn',
      `btn--${variant}`,
      `btn--${size}`,
      className
    );

    return (
      <Component
        ref={ref}
        className={classes}
        {...props}
      >
        {children}
      </Component>
    );
  }
);

// Usage examples
const Examples: React.FC = () => (
  <div>
    {/* Renders as button */}
    <Button onClick={() => console.log('clicked')}>
      Click me
    </Button>
    
    {/* Renders as link */}
    <Button as="a" href="/profile">
      Go to Profile
    </Button>
    
    {/* Renders as Next.js Link */}
    <Button as={Link} to="/dashboard">
      Dashboard
    </Button>
    
    {/* Renders as custom component */}
    <Button as={CustomLink} variant="secondary">
      Custom Link
    </Button>
  </div>
);
```

## Advanced Component Patterns

### 1. Headless Components

**Headless Select Component:**
```typescript
// Headless select hook for maximum flexibility
interface UseSelectProps<T> {
  options: T[];
  value?: T;
  defaultValue?: T;
  onSelectionChange?: (value: T) => void;
  getOptionLabel?: (option: T) => string;
  getOptionValue?: (option: T) => string;
  isOptionDisabled?: (option: T) => boolean;
  multiple?: boolean;
}

interface UseSelectReturn<T> {
  // State
  isOpen: boolean;
  selectedValue: T | T[] | undefined;
  highlightedIndex: number;
  
  // Actions
  openMenu: () => void;
  closeMenu: () => void;
  toggleMenu: () => void;
  selectOption: (option: T) => void;
  highlightOption: (index: number) => void;
  
  // Helper functions
  getOptionProps: (option: T, index: number) => {
    role: string;
    'aria-selected': boolean;
    'aria-disabled': boolean;
    onClick: () => void;
    onMouseEnter: () => void;
  };
  getMenuProps: () => {
    role: string;
    'aria-labelledby': string;
  };
  getTriggerProps: () => {
    'aria-haspopup': boolean;
    'aria-expanded': boolean;
    onClick: () => void;
    onKeyDown: (event: React.KeyboardEvent) => void;
  };
}

function useSelect<T>({
  options,
  value,
  defaultValue,
  onSelectionChange,
  getOptionLabel = (option) => String(option),
  getOptionValue = (option) => String(option),
  isOptionDisabled = () => false,
  multiple = false,
}: UseSelectProps<T>): UseSelectReturn<T> {
  const [isOpen, setIsOpen] = useState(false);
  const [selectedValue, setSelectedValue] = useState(value || defaultValue);
  const [highlightedIndex, setHighlightedIndex] = useState(-1);

  const selectOption = useCallback((option: T) => {
    if (isOptionDisabled(option)) return;

    let newValue: T | T[];
    
    if (multiple) {
      const currentArray = (selectedValue as T[]) || [];
      const optionValue = getOptionValue(option);
      const isSelected = currentArray.some(
        item => getOptionValue(item) === optionValue
      );
      
      newValue = isSelected
        ? currentArray.filter(item => getOptionValue(item) !== optionValue)
        : [...currentArray, option];
    } else {
      newValue = option;
      setIsOpen(false);
    }

    setSelectedValue(newValue);
    onSelectionChange?.(newValue);
  }, [selectedValue, multiple, getOptionValue, isOptionDisabled, onSelectionChange]);

  const handleKeyDown = useCallback((event: React.KeyboardEvent) => {
    switch (event.key) {
      case 'ArrowDown':
        event.preventDefault();
        if (!isOpen) {
          setIsOpen(true);
        } else {
          setHighlightedIndex(prev => 
            prev < options.length - 1 ? prev + 1 : prev
          );
        }
        break;
        
      case 'ArrowUp':
        event.preventDefault();
        setHighlightedIndex(prev => prev > 0 ? prev - 1 : prev);
        break;
        
      case 'Enter':
        event.preventDefault();
        if (isOpen && highlightedIndex >= 0) {
          selectOption(options[highlightedIndex]);
        }
        break;
        
      case 'Escape':
        setIsOpen(false);
        setHighlightedIndex(-1);
        break;
    }
  }, [isOpen, highlightedIndex, options, selectOption]);

  return {
    isOpen,
    selectedValue,
    highlightedIndex,
    openMenu: () => setIsOpen(true),
    closeMenu: () => setIsOpen(false),
    toggleMenu: () => setIsOpen(prev => !prev),
    selectOption,
    highlightOption: setHighlightedIndex,
    
    getOptionProps: (option, index) => ({
      role: 'option',
      'aria-selected': multiple 
        ? (selectedValue as T[])?.some(item => 
            getOptionValue(item) === getOptionValue(option)
          ) || false
        : getOptionValue(selectedValue as T) === getOptionValue(option),
      'aria-disabled': isOptionDisabled(option),
      onClick: () => selectOption(option),
      onMouseEnter: () => setHighlightedIndex(index),
    }),
    
    getMenuProps: () => ({
      role: 'listbox',
      'aria-labelledby': 'select-label',
    }),
    
    getTriggerProps: () => ({
      'aria-haspopup': true as const,
      'aria-expanded': isOpen,
      onClick: () => setIsOpen(prev => !prev),
      onKeyDown: handleKeyDown,
    }),
  };
}

// Select component using the headless hook
interface SelectProps<T> extends UseSelectProps<T> {
  placeholder?: string;
  className?: string;
}

function Select<T>({
  placeholder = 'Select an option...',
  className,
  getOptionLabel = (option) => String(option),
  ...props
}: SelectProps<T>) {
  const {
    isOpen,
    selectedValue,
    highlightedIndex,
    getOptionProps,
    getMenuProps,
    getTriggerProps,
  } = useSelect(props);

  return (
    <div className={cn('select', className)}>
      <button
        className={cn('select__trigger', {
          'select__trigger--open': isOpen,
        })}
        {...getTriggerProps()}
      >
        <span className="select__value">
          {selectedValue ? getOptionLabel(selectedValue as T) : placeholder}
        </span>
        <ChevronIcon 
          className={cn('select__icon', {
            'select__icon--rotated': isOpen,
          })} 
        />
      </button>
      
      {isOpen && (
        <div className="select__menu" {...getMenuProps()}>
          {props.options.map((option, index) => (
            <div
              key={getOptionValue ? getOptionValue(option) : index}
              className={cn('select__option', {
                'select__option--highlighted': index === highlightedIndex,
                'select__option--selected': selectedValue === option,
              })}
              {...getOptionProps(option, index)}
            >
              {getOptionLabel(option)}
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

### 2. Component Composition Patterns

**Form Component System:**
```typescript
// Context-based form system
interface FormContextValue {
  values: Record<string, any>;
  errors: Record<string, string>;
  touched: Record<string, boolean>;
  isSubmitting: boolean;
  setValue: (name: string, value: any) => void;
  setError: (name: string, error: string) => void;
  setTouched: (name: string, touched: boolean) => void;
  validateField: (name: string) => void;
  validateForm: () => boolean;
}

const FormContext = createContext<FormContextValue | null>(null);

const useFormContext = () => {
  const context = useContext(FormContext);
  if (!context) {
    throw new Error('Form components must be used within Form');
  }
  return context;
};

// Form root component
interface FormProps {
  initialValues?: Record<string, any>;
  validationSchema?: any; // Yup schema or similar
  onSubmit: (values: Record<string, any>) => void | Promise<void>;
  children: React.ReactNode;
}

const Form: React.FC<FormProps> & {
  Field: typeof FormField;
  Input: typeof FormInput;
  Textarea: typeof FormTextarea;
  Select: typeof FormSelect;
  Error: typeof FormError;
  Submit: typeof FormSubmit;
} = ({ initialValues = {}, validationSchema, onSubmit, children }) => {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState<Record<string, string>>({});
  const [touched, setTouched] = useState<Record<string, boolean>>({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const setValue = (name: string, value: any) => {
    setValues(prev => ({ ...prev, [name]: value }));
    // Clear error when user starts typing
    if (errors[name]) {
      setErrors(prev => ({ ...prev, [name]: '' }));
    }
  };

  const setError = (name: string, error: string) => {
    setErrors(prev => ({ ...prev, [name]: error }));
  };

  const setFieldTouched = (name: string, isTouched: boolean) => {
    setTouched(prev => ({ ...prev, [name]: isTouched }));
  };

  const validateField = (name: string) => {
    if (!validationSchema) return;
    
    try {
      validationSchema.validateSyncAt(name, values);
      setError(name, '');
    } catch (error) {
      setError(name, error.message);
    }
  };

  const validateForm = () => {
    if (!validationSchema) return true;
    
    try {
      validationSchema.validateSync(values, { abortEarly: false });
      setErrors({});
      return true;
    } catch (error) {
      const validationErrors: Record<string, string> = {};
      error.inner.forEach((err: any) => {
        validationErrors[err.path] = err.message;
      });
      setErrors(validationErrors);
      return false;
    }
  };

  const handleSubmit = async (event: React.FormEvent) => {
    event.preventDefault();
    
    if (!validateForm()) return;
    
    setIsSubmitting(true);
    try {
      await onSubmit(values);
    } catch (error) {
      console.error('Form submission error:', error);
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <FormContext.Provider value={{
      values,
      errors,
      touched,
      isSubmitting,
      setValue,
      setError,
      setTouched: setFieldTouched,
      validateField,
      validateForm,
    }}>
      <form onSubmit={handleSubmit}>
        {children}
      </form>
    </FormContext.Provider>
  );
};

// Form field wrapper
interface FormFieldProps {
  name: string;
  label?: string;
  required?: boolean;
  children: React.ReactNode;
}

const FormField: React.FC<FormFieldProps> = ({ 
  name, 
  label, 
  required, 
  children 
}) => {
  const { errors, touched } = useFormContext();
  const hasError = touched[name] && errors[name];

  return (
    <div className={cn('form-field', {
      'form-field--error': hasError,
    })}>
      {label && (
        <label htmlFor={name} className="form-field__label">
          {label}
          {required && <span className="form-field__required">*</span>}
        </label>
      )}
      {children}
      {hasError && (
        <Form.Error name={name} />
      )}
    </div>
  );
};

// Form input component
interface FormInputProps extends Omit<React.InputHTMLAttributes<HTMLInputElement>, 'name'> {
  name: string;
}

const FormInput: React.FC<FormInputProps> = ({ name, ...props }) => {
  const { values, setValue, setTouched, validateField } = useFormContext();

  return (
    <input
      id={name}
      name={name}
      value={values[name] || ''}
      onChange={(e) => setValue(name, e.target.value)}
      onBlur={() => {
        setTouched(name, true);
        validateField(name);
      }}
      className="form-input"
      {...props}
    />
  );
};

// Error display component
const FormError: React.FC<{ name: string }> = ({ name }) => {
  const { errors, touched } = useFormContext();
  
  if (!touched[name] || !errors[name]) return null;

  return (
    <span className="form-error" role="alert">
      {errors[name]}
    </span>
  );
};

// Submit button
const FormSubmit: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const { isSubmitting } = useFormContext();

  return (
    <Button type="submit" disabled={isSubmitting} loading={isSubmitting}>
      {children}
    </Button>
  );
};

// Attach sub-components
Form.Field = FormField;
Form.Input = FormInput;
Form.Error = FormError;
Form.Submit = FormSubmit;

// Usage
const UserForm: React.FC = () => (
  <Form
    initialValues={{ name: '', email: '' }}
    validationSchema={userSchema}
    onSubmit={handleSubmit}
  >
    <Form.Field name="name" label="Name" required>
      <Form.Input name="name" placeholder="Enter your name" />
    </Form.Field>
    
    <Form.Field name="email" label="Email" required>
      <Form.Input name="email" type="email" placeholder="Enter your email" />
    </Form.Field>
    
    <Form.Submit>
      Create User
    </Form.Submit>
  </Form>
);
```

## Documentation & Developer Experience

### 1. Component Documentation

**Storybook Integration:**
```typescript
// Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';
import { action } from '@storybook/addon-actions';

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
  parameters: {
    docs: {
      description: {
        component: 'A versatile button component that supports multiple variants, sizes, and states.',
      },
    },
  },
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'tertiary', 'danger'],
      description: 'Visual style variant of the button',
    },
    size: {
      control: 'radio',
      options: ['small', 'medium', 'large'],
      description: 'Size of the button',
    },
    disabled: {
      control: 'boolean',
      description: 'Disabled state of the button',
    },
    loading: {
      control: 'boolean',
      description: 'Loading state with spinner',
    },
    fullWidth: {
      control: 'boolean',
      description: 'Whether button should take full width of container',
    },
    onClick: {
      action: 'clicked',
      description: 'Click event handler',
    },
  },
  args: {
    children: 'Button',
    onClick: action('clicked'),
  },
};

export default meta;
type Story = StoryObj<typeof Button>;

export const Default: Story = {};

export const Variants: Story = {
  render: () => (
    <div style={{ display: 'flex', gap: '1rem', flexWrap: 'wrap' }}>
      <Button variant="primary">Primary</Button>
      <Button variant="secondary">Secondary</Button>
      <Button variant="tertiary">Tertiary</Button>
      <Button variant="danger">Danger</Button>
    </div>
  ),
};

export const Sizes: Story = {
  render: () => (
    <div style={{ display: 'flex', gap: '1rem', alignItems: 'center' }}>
      <Button size="small">Small</Button>
      <Button size="medium">Medium</Button>
      <Button size="large">Large</Button>
    </div>
  ),
};

export const WithIcons: Story = {
  render: () => (
    <div style={{ display: 'flex', gap: '1rem', flexWrap: 'wrap' }}>
      <Button startIcon={<PlusIcon />}>Add Item</Button>
      <Button endIcon={<ArrowRightIcon />}>Continue</Button>
      <Button 
        startIcon={<DownloadIcon />} 
        endIcon={<ExternalLinkIcon />}
      >
        Download
      </Button>
    </div>
  ),
};

export const States: Story = {
  render: () => (
    <div style={{ display: 'flex', gap: '1rem', flexWrap: 'wrap' }}>
      <Button>Normal</Button>
      <Button disabled>Disabled</Button>
      <Button loading>Loading</Button>
    </div>
  ),
};
```

### 2. API Documentation Generation

**TypeScript Props Extraction:**
```typescript
// docs/generate-props-docs.ts
import * as ts from 'typescript';
import * as path from 'path';

interface PropInfo {
  name: string;
  type: string;
  required: boolean;
  description?: string;
  defaultValue?: string;
}

export function extractPropsFromComponent(filePath: string): PropInfo[] {
  const program = ts.createProgram([filePath], {});
  const sourceFile = program.getSourceFile(filePath);
  const checker = program.getTypeChecker();

  if (!sourceFile) return [];

  const props: PropInfo[] = [];

  function visit(node: ts.Node) {
    // Find interface declarations ending with "Props"
    if (ts.isInterfaceDeclaration(node) && node.name.text.endsWith('Props')) {
      node.members.forEach(member => {
        if (ts.isPropertySignature(member) && member.name) {
          const propName = member.name.getText();
          const type = checker.typeToString(
            checker.getTypeOfSymbolAtLocation(
              checker.getSymbolAtLocation(member.name!)!,
              member.name!
            )
          );
          
          const required = !member.questionToken;
          const jsDocTags = ts.getJSDocTags(member);
          const description = jsDocTags
            .find(tag => tag.tagName.text === 'description')
            ?.comment as string;

          props.push({
            name: propName,
            type,
            required,
            description,
          });
        }
      });
    }

    ts.forEachChild(node, visit);
  }

  visit(sourceFile);
  return props;
}

// Generate markdown documentation
export function generateComponentDocs(componentPath: string): string {
  const props = extractPropsFromComponent(componentPath);
  const componentName = path.basename(componentPath, '.tsx');

  let markdown = `# ${componentName}\n\n`;
  
  if (props.length > 0) {
    markdown += '## Props\n\n';
    markdown += '| Prop | Type | Required | Description |\n';
    markdown += '|------|------|----------|-------------|\n';
    
    props.forEach(prop => {
      markdown += `| ${prop.name} | \`${prop.type}\` | ${prop.required ? '✅' : '❌'} | ${prop.description || '-'} |\n`;
    });
  }

  return markdown;
}
```

## Testing Architecture

### 1. Component Testing Strategy

**Comprehensive Test Suite:**
```typescript
// Button.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from './Button';
import { ThemeProvider } from '../ThemeProvider';

// Test utilities
const renderWithTheme = (ui: React.ReactElement) => {
  return render(
    <ThemeProvider theme={defaultTheme}>
      {ui}
    </ThemeProvider>
  );
};

describe('Button Component', () => {
  describe('Rendering', () => {
    it('renders with default props', () => {
      renderWithTheme(<Button>Click me</Button>);
      
      const button = screen.getByRole('button', { name: /click me/i });
      expect(button).toBeInTheDocument();
      expect(button).toHaveClass('btn', 'btn--primary', 'btn--medium');
    });

    it('renders different variants correctly', () => {
      const variants = ['primary', 'secondary', 'tertiary', 'danger'] as const;
      
      variants.forEach(variant => {
        const { unmount } = renderWithTheme(
          <Button variant={variant}>Button</Button>
        );
        
        const button = screen.getByRole('button');
        expect(button).toHaveClass(`btn--${variant}`);
        
        unmount();
      });
    });

    it('renders with icons', () => {
      renderWithTheme(
        <Button 
          startIcon={<span data-testid="start-icon">+</span>}
          endIcon={<span data-testid="end-icon">→</span>}
        >
          Button
        </Button>
      );

      expect(screen.getByTestId('start-icon')).toBeInTheDocument();
      expect(screen.getByTestId('end-icon')).toBeInTheDocument();
    });
  });

  describe('Interaction', () => {
    it('calls onClick handler when clicked', async () => {
      const handleClick = jest.fn();
      const user = userEvent.setup();
      
      renderWithTheme(
        <Button onClick={handleClick}>Click me</Button>
      );

      await user.click(screen.getByRole('button'));
      expect(handleClick).toHaveBeenCalledTimes(1);
    });

    it('does not call onClick when disabled', async () => {
      const handleClick = jest.fn();
      const user = userEvent.setup();
      
      renderWithTheme(
        <Button onClick={handleClick} disabled>
          Click me
        </Button>
      );

      await user.click(screen.getByRole('button'));
      expect(handleClick).not.toHaveBeenCalled();
    });

    it('supports keyboard navigation', async () => {
      const handleClick = jest.fn();
      const user = userEvent.setup();
      
      renderWithTheme(
        <Button onClick={handleClick}>Click me</Button>
      );

      const button = screen.getByRole('button');
      button.focus();
      await user.keyboard('{Enter}');
      
      expect(handleClick).toHaveBeenCalledTimes(1);
    });
  });

  describe('Accessibility', () => {
    it('has correct ARIA attributes', () => {
      renderWithTheme(
        <Button 
          disabled 
          aria-label="Save document"
          aria-describedby="save-help"
        >
          Save
        </Button>
      );

      const button = screen.getByRole('button');
      expect(button).toHaveAttribute('aria-disabled', 'true');
      expect(button).toHaveAttribute('aria-label', 'Save document');
      expect(button).toHaveAttribute('aria-describedby', 'save-help');
    });

    it('has correct loading state accessibility', () => {
      renderWithTheme(
        <Button loading>Loading</Button>
      );

      const button = screen.getByRole('button');
      expect(button).toHaveAttribute('aria-disabled', 'true');
      expect(button).toBeDisabled();
    });
  });

  describe('Visual Regression', () => {
    it('matches snapshot for all variants', () => {
      const { container } = renderWithTheme(
        <div>
          <Button variant="primary">Primary</Button>
          <Button variant="secondary">Secondary</Button>
          <Button variant="tertiary">Tertiary</Button>
          <Button variant="danger">Danger</Button>
        </div>
      );

      expect(container.firstChild).toMatchSnapshot();
    });
  });
});
```

### 2. Integration Testing

**Component Integration Tests:**
```typescript
// Form.integration.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Form } from './Form';
import * as yup from 'yup';

const validationSchema = yup.object({
  name: yup.string().required('Name is required'),
  email: yup.string().email('Invalid email').required('Email is required'),
});

describe('Form Integration', () => {
  it('handles complete form submission flow', async () => {
    const onSubmit = jest.fn();
    const user = userEvent.setup();

    render(
      <Form
        validationSchema={validationSchema}
        onSubmit={onSubmit}
      >
        <Form.Field name="name" label="Name" required>
          <Form.Input name="name" />
        </Form.Field>
        
        <Form.Field name="email" label="Email" required>
          <Form.Input name="email" type="email" />
        </Form.Field>
        
        <Form.Submit>Submit</Form.Submit>
      </Form>
    );

    // Fill form
    await user.type(screen.getByLabelText(/name/i), 'John Doe');
    await user.type(screen.getByLabelText(/email/i), 'john@example.com');

    // Submit form
    await user.click(screen.getByRole('button', { name: /submit/i }));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({
        name: 'John Doe',
        email: 'john@example.com',
      });
    });
  });

  it('displays validation errors', async () => {
    const onSubmit = jest.fn();
    const user = userEvent.setup();

    render(
      <Form
        validationSchema={validationSchema}
        onSubmit={onSubmit}
      >
        <Form.Field name="name" label="Name" required>
          <Form.Input name="name" />
        </Form.Field>
        
        <Form.Field name="email" label="Email" required>
          <Form.Input name="email" type="email" />
        </Form.Field>
        
        <Form.Submit>Submit</Form.Submit>
      </Form>
    );

    // Try to submit empty form
    await user.click(screen.getByRole('button', { name: /submit/i }));

    await waitFor(() => {
      expect(screen.getByText('Name is required')).toBeInTheDocument();
      expect(screen.getByText('Email is required')).toBeInTheDocument();
    });

    expect(onSubmit).not.toHaveBeenCalled();
  });
});
```

## Interview-Ready Summary

**Component library architecture requires:**

1. **Design Token System** - Hierarchical tokens (primitive → semantic → component), CSS custom properties integration, theme provider pattern

2. **Consistent API Design** - Base component props, forwarded refs, polymorphic components, compound component patterns

3. **Advanced Patterns** - Headless components for flexibility, render props for customization, context-based composition

4. **Developer Experience** - Comprehensive documentation, Storybook integration, TypeScript prop extraction, automated testing

5. **Scalability Considerations** - Tree-shakeable exports, bundle size optimization, performance monitoring, accessibility compliance

**Key principles:** Start with design tokens, build composable primitives, prioritize accessibility, document everything, test comprehensively. The goal is creating a system that scales with team growth while maintaining consistency and developer productivity.

**Common challenges:** Balancing flexibility vs. consistency, managing breaking changes, ensuring accessibility compliance, optimizing bundle size, and maintaining documentation quality.