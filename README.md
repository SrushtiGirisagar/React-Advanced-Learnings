# ⚛️ React Internals & Patterns — Deep Dive

> A comprehensive reference for intermediate-to-senior React engineers covering the engine under the hood, performance patterns, state management tradeoffs, and reusable hook design.

---

## Table of Contents

1. [Reconciliation & Virtual DOM](#1-reconciliation--virtual-dom)
2. [Rendering Optimization](#2-rendering-optimization)
   - [React.memo](#21-reactmemo)
   - [useMemo](#22-usememo)
   - [useCallback](#23-usecallback)
3. [State Management Tradeoffs](#3-state-management-tradeoffs)
   - [Context API](#31-context-api)
   - [Redux](#32-redux)
   - [Zustand](#33-zustand)
   - [Comparison Table](#34-comparison-table)
4. [Controlled vs Uncontrolled Components](#4-controlled-vs-uncontrolled-components)
5. [Custom Hooks](#5-custom-hooks)

---

## 1. Reconciliation & Virtual DOM

### What is the Virtual DOM?

The **Virtual DOM (VDOM)** is a lightweight, in-memory JavaScript representation of the real DOM tree. React maintains two VDOM snapshots at any point in time:

- **Current tree** — what is currently rendered on screen
- **Work-in-progress tree** — what the next render will look like

When state or props change, React builds a new work-in-progress tree and **diffs** it against the current tree to determine the minimal set of real DOM operations needed.

```
State Change
     │
     ▼
New VDOM Tree (work-in-progress)
     │
     ▼
Diffing Algorithm (Reconciliation)
     │
     ▼
Minimal DOM Mutations (Commit Phase)
     │
     ▼
Browser Paints Updated UI
```

### How Diffing Works

React's diffing algorithm runs in **O(n)** time (rather than the theoretical O(n³) of a general tree diff) by applying two key heuristics:

#### Heuristic 1 — Different element types produce different trees

If the root element type changes, React tears down the old tree entirely and builds a fresh one.

```jsx
// Before
<div>
  <Counter />
</div>

// After — React unmounts <div> subtree, mounts <section> subtree from scratch
<section>
  <Counter />
</section>
```

#### Heuristic 2 — The `key` prop stabilises list identity

Without keys, React diffs by position. With keys, it diffs by identity — allowing efficient reordering.

```jsx
// ❌ Bad — React diffs by index; inserting at top is O(n) mutations
{items.map((item, index) => (
  <Item key={index} data={item} />
))}

// ✅ Good — React tracks by stable ID; insert at top = 1 mutation
{items.map((item) => (
  <Item key={item.id} data={item} />
))}
```

### The Three Phases of Reconciliation

| Phase | What happens | Interruptible? |
|---|---|---|
| **Render** | React calls your component functions, builds VDOM | ✅ Yes (Concurrent Mode) |
| **Reconciliation (Diff)** | React compares old vs new VDOM trees | ✅ Yes |
| **Commit** | React applies changes to the real DOM, runs effects | ❌ No (synchronous) |

> **React Fiber** (introduced in React 16) makes the render and reconciliation phases interruptible, enabling features like Concurrent Mode, `Suspense`, and `useTransition`.

### Key Takeaways

- Avoid changing element **types** unnecessarily (e.g. swapping `div` → `section`) — it forces full remounts
- Always use **stable, unique keys** in lists — never array indices for dynamic lists
- Component identity is preserved as long as the element type and key remain the same across renders

---

## 2. Rendering Optimization

React re-renders a component whenever its **state changes**, its **parent re-renders**, or its **context value changes**. The tools below let you opt out of unnecessary re-renders.

### 2.1 `React.memo`

`React.memo` is a **Higher-Order Component (HOC)** that memoizes the rendered output of a functional component. It performs a **shallow comparison** of props and skips re-rendering if props haven't changed.

```jsx
// Without memo — re-renders every time parent renders, even if `name` is unchanged
const Greeting = ({ name }) => {
  console.log('Rendered!');
  return <h1>Hello, {name}</h1>;
};

// With memo — only re-renders when `name` actually changes
const Greeting = React.memo(({ name }) => {
  console.log('Rendered!');
  return <h1>Hello, {name}</h1>;
});
```

#### Custom comparison function

```jsx
const UserCard = React.memo(
  ({ user }) => <div>{user.name}</div>,
  (prevProps, nextProps) => prevProps.user.id === nextProps.user.id
  // Return true = skip re-render, false = re-render
);
```

#### When to use

| ✅ Use `React.memo` when | ❌ Skip it when |
|---|---|
| Component renders frequently with same props | Component almost always receives new props |
| Rendering is expensive (large lists, heavy computation) | Component is cheap to render |
| Component receives primitive or stable object props | Props are always new object references |

> ⚠️ `React.memo` only does a **shallow comparison**. Passing a new object or array reference on every render defeats it — pair with `useMemo` / `useCallback` on the parent side.

---

### 2.2 `useMemo`

`useMemo` memoizes the **result of a computation**, recomputing it only when dependencies change.

```jsx
const sortedList = useMemo(() => {
  return items.sort((a, b) => a.price - b.price); // Only re-runs when `items` changes
}, [items]);
```

#### Practical example — expensive filter

```jsx
const ProductGrid = ({ products, searchTerm }) => {
  // ❌ Without useMemo — filters on every render
  const filtered = products.filter(p =>
    p.name.toLowerCase().includes(searchTerm.toLowerCase())
  );

  // ✅ With useMemo — only re-filters when products or searchTerm changes
  const filtered = useMemo(
    () => products.filter(p =>
      p.name.toLowerCase().includes(searchTerm.toLowerCase())
    ),
    [products, searchTerm]
  );

  return filtered.map(p => <ProductCard key={p.id} product={p} />);
};
```

#### Referential stability for objects/arrays

```jsx
// ❌ New object reference on every render — breaks React.memo on child
const config = { theme: 'dark', size: 'lg' };

// ✅ Stable reference — React.memo on child works correctly
const config = useMemo(() => ({ theme: 'dark', size: 'lg' }), []);
```

---

### 2.3 `useCallback`

`useCallback` memoizes a **function reference**, returning the same function instance across renders unless dependencies change.

```jsx
// ❌ New function reference on every render — child wrapped in React.memo still re-renders
const handleClick = () => doSomething(id);

// ✅ Stable function reference
const handleClick = useCallback(() => doSomething(id), [id]);
```

#### Full example

```jsx
const Parent = () => {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');

  // Without useCallback: new reference on every keystroke → Child always re-renders
  // With useCallback: reference only changes when count changes
  const handleIncrement = useCallback(() => {
    setCount(c => c + 1);
  }, []); // No deps — uses functional updater pattern

  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <ExpensiveChild onIncrement={handleIncrement} />
    </>
  );
};

const ExpensiveChild = React.memo(({ onIncrement }) => {
  console.log('ExpensiveChild rendered');
  return <button onClick={onIncrement}>Increment</button>;
});
```

### Mental Model — When to use which

```
Is the value an expensive computed result?  →  useMemo
Is the value a function passed as a prop?   →  useCallback
Is the component receiving stable props?    →  React.memo
```

---

## 3. State Management Tradeoffs

### 3.1 Context API

React's built-in solution. Passes values through the component tree without prop drilling.

```jsx
// 1. Create context
const ThemeContext = createContext('light');

// 2. Provide value
const App = () => (
  <ThemeContext.Provider value="dark">
    <Layout />
  </ThemeContext.Provider>
);

// 3. Consume anywhere in the tree
const Button = () => {
  const theme = useContext(ThemeContext);
  return <button className={theme}>Click</button>;
};
```

#### ⚠️ The re-render problem

Every component that calls `useContext` re-renders whenever the context **value reference** changes — even if it only uses part of the value.

```jsx
// ❌ Avoid — new object reference on every render; ALL consumers re-render
<UserContext.Provider value={{ user, setUser }}>

// ✅ Better — memoize the value
const value = useMemo(() => ({ user, setUser }), [user]);
<UserContext.Provider value={value}>
```

**Best for:** Theme, locale, auth state — low-frequency updates consumed widely.

---

### 3.2 Redux

A predictable, centralised state container with a strict unidirectional data flow. Best used with **Redux Toolkit (RTK)**, which eliminates most boilerplate.

```jsx
// store/counterSlice.js
import { createSlice } from '@reduxjs/toolkit';

const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: state => { state.value += 1; },
    decrement: state => { state.value -= 1; },
    incrementByAmount: (state, action) => { state.value += action.payload; },
  },
});

export const { increment, decrement, incrementByAmount } = counterSlice.actions;
export default counterSlice.reducer;

// Component usage
const Counter = () => {
  const count = useSelector(state => state.counter.value);
  const dispatch = useDispatch();

  return (
    <>
      <span>{count}</span>
      <button onClick={() => dispatch(increment())}>+</button>
    </>
  );
};
```

**Best for:** Large-scale apps, complex state logic, teams that benefit from strict conventions and devtools.

---

### 3.3 Zustand

A minimalist, hook-based store with zero boilerplate and no provider required.

```jsx
import { create } from 'zustand';

const useCounterStore = create(set => ({
  count: 0,
  increment: () => set(state => ({ count: state.count + 1 })),
  decrement: () => set(state => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}));

// Component — only subscribes to `count`; won't re-render when other state changes
const Counter = () => {
  const count = useCounterStore(state => state.count);
  const increment = useCounterStore(state => state.increment);

  return <button onClick={increment}>{count}</button>;
};
```

**Best for:** Mid-size apps, fast prototyping, teams that want Redux-like patterns without Redux ceremony.

---

### 3.4 Comparison Table

| | Context API | Redux (RTK) | Zustand |
|---|---|---|---|
| **Boilerplate** | Minimal | Medium | Minimal |
| **Bundle size** | 0 kb (built-in) | ~40 kb | ~1 kb |
| **DevTools** | ❌ | ✅ Excellent | ✅ Basic |
| **Async support** | Manual | RTK Query / Thunk | Middleware |
| **Selective re-renders** | ❌ (manual split) | ✅ `useSelector` | ✅ Selectors |
| **Provider needed** | ✅ | ✅ | ❌ |
| **Learning curve** | Low | Medium–High | Low |
| **Best for** | Simple/shared UI state | Enterprise, complex flows | Mid-scale, DX-first teams |

---

## 4. Controlled vs Uncontrolled Components

The distinction defines **who owns the source of truth** for a form element's value.

### Controlled Components

React state is the single source of truth. Every keystroke updates state, and the input value is driven by that state.

```jsx
const ControlledForm = () => {
  const [email, setEmail] = useState('');

  const handleSubmit = (e) => {
    e.preventDefault();
    console.log(email); // Always in sync with input
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}                          // Driven by state
        onChange={e => setEmail(e.target.value)} // State updated on every keystroke
      />
      <button type="submit">Submit</button>
    </form>
  );
};
```

**Use when:** You need instant validation, conditional rendering based on input, or controlled field state (e.g. character count, format masking).

### Uncontrolled Components

The DOM owns the value. React reads it on demand using a `ref`.

```jsx
const UncontrolledForm = () => {
  const emailRef = useRef(null);

  const handleSubmit = (e) => {
    e.preventDefault();
    console.log(emailRef.current.value); // Read from DOM on submit
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        ref={emailRef}           // Ref wired to DOM node
        defaultValue=""          // Initial value only — not controlled
      />
      <button type="submit">Submit</button>
    </form>
  );
};
```

**Use when:** Integrating with non-React libraries, file inputs (`<input type="file">` is always uncontrolled), or performance-sensitive large forms.

### Quick Comparison

| | Controlled | Uncontrolled |
|---|---|---|
| **Source of truth** | React state | DOM |
| **Access value** | `state` variable | `ref.current.value` |
| **Instant validation** | ✅ Easy | ❌ Requires ref reads |
| **Re-renders on change** | ✅ Yes | ❌ No |
| **File input support** | ❌ | ✅ |
| **Form library fit** | React Hook Form (perf) | Formik (controlled) |

---

## 5. Custom Hooks

Custom hooks are **plain JavaScript functions** that start with `use` and can call other hooks. They are the primary mechanism for extracting and reusing stateful logic across components.

### Rules

1. Name starts with `use` — required for lint rules and React DevTools
2. Can call other hooks (`useState`, `useEffect`, `useContext`, etc.)
3. Not a component — returns data/functions, not JSX

### Example 1 — `useFetch` (data fetching with loading state)

```jsx
const useFetch = (url) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);

    fetch(url)
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(json => { if (!cancelled) setData(json); })
      .catch(err => { if (!cancelled) setError(err); })
      .finally(() => { if (!cancelled) setLoading(false); });

    return () => { cancelled = true; }; // Cleanup — prevent state update on unmounted component
  }, [url]);

  return { data, loading, error };
};

// Usage
const UserProfile = ({ userId }) => {
  const { data, loading, error } = useFetch(`/api/users/${userId}`);

  if (loading) return <Spinner />;
  if (error) return <Error message={error.message} />;
  return <div>{data.name}</div>;
};
```

### Example 2 — `useDebounce` (input debouncing)

```jsx
const useDebounce = (value, delay = 300) => {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
};

// Usage — prevents API call on every keystroke
const SearchBar = () => {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 400);

  const { data } = useFetch(`/api/search?q=${debouncedQuery}`);

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <SearchResults results={data} />
    </>
  );
};
```

### Example 3 — `useLocalStorage` (persisted state)

```jsx
const useLocalStorage = (key, initialValue) => {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = useCallback((value) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.warn(`useLocalStorage: Failed to set "${key}"`, error);
    }
  }, [key, storedValue]);

  return [storedValue, setValue];
};

// Usage — drop-in replacement for useState, with persistence
const Settings = () => {
  const [theme, setTheme] = useLocalStorage('theme', 'light');
  return <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>{theme}</button>;
};
```

### Example 4 — `usePrevious` (track previous value)

```jsx
const usePrevious = (value) => {
  const ref = useRef(undefined);

  useEffect(() => {
    ref.current = value;
  }); // No deps — runs after every render

  return ref.current; // Returns value from previous render
};

// Usage
const Counter = () => {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);

  return <p>Now: {count}, Before: {prevCount}</p>;
};
```

### Custom Hook Design Principles

| Principle | Description |
|---|---|
| **Single responsibility** | One hook = one concern. Don't combine unrelated logic. |
| **Return a stable API** | Return an object `{ data, loading }` for multiple values, not a long tuple. |
| **Handle cleanup** | Always return a cleanup function from `useEffect` for subscriptions, timers, and fetch calls. |
| **Accept config via args** | Make hooks reusable by accepting options (delay, key, url) as parameters. |
| **Memoize returned functions** | Wrap returned callbacks in `useCallback` so consumers can safely use them as deps. |

---

## Summary

```
Reconciliation  →  VDOM diff → minimal DOM ops (O(n) via type + key heuristics)
React.memo      →  Skip re-render when props are shallowly equal
useMemo         →  Cache expensive computed values
useCallback     →  Cache function references (pairs with React.memo)
Context         →  Simple shared state, low-frequency updates
Redux           →  Complex, large-scale state with strict conventions
Zustand         →  Lightweight Redux alternative, zero boilerplate
Controlled      →  React owns input value (state-driven)
Uncontrolled    →  DOM owns input value (ref-read on demand)
Custom Hooks    →  Extract + reuse stateful logic without JSX
```

---

*Maintained by [Srushti Girisagar](https://linkedin.com/in/srushti-girisagar) · React Frontend Engineer*
