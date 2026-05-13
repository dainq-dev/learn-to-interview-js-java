# React Core & Hooks — Lý thuyết sâu + Ví dụ 3 cấp độ

---

## 1. VIRTUAL DOM & RECONCILIATION

### Lý thuyết & Bản chất
**Virtual DOM (VDOM)** là một bản sao nhẹ của DOM thật, được lưu trong bộ nhớ dưới dạng cây JavaScript object thuần (plain JS objects). React dùng nó như một "bản nháp".

**Luồng hoạt động:**
```
State/Props thay đổi
    ↓
React tạo VDOM mới (Render Phase)
    ↓
Diffing Algorithm so sánh VDOM mới vs VDOM cũ
    ↓
Tạo danh sách thay đổi tối thiểu (Reconciliation)
    ↓
Apply lên Real DOM (Commit Phase)
```

**Tại sao cần VDOM?**
- Thao tác trực tiếp với DOM thật rất chậm (reflow, repaint)
- Batch nhiều thay đổi → áp dụng 1 lần → tối ưu hơn

**React Fiber (React 16+):** Kiến trúc lại của reconciler. "Fiber" là đơn vị công việc nhỏ nhất. Fiber cho phép:
- **Incremental rendering**: chia nhỏ render, có thể dừng/tiếp tục/huỷ
- **Priority scheduling**: ưu tiên animation/input hơn data fetching
- **Concurrent Mode**: nhiều version của UI có thể tồn tại cùng lúc

---

### Ví dụ 1 — Cơ bản: Chứng minh Diffing với Key
```jsx
// React diffing heuristic:
// 1. Element khác type → destroy cũ, create mới
// 2. Element cùng type → update attributes, recurse vào children
// 3. List: dùng key để match element cũ với mới

function FruitList() {
  const [fruits, setFruits] = React.useState(['Apple', 'Banana', 'Cherry']);

  return (
    <div>
      <ul>
        {fruits.map((fruit, index) => (
          // BAD: dùng index làm key
          <li key={index}>{fruit}</li>
        ))}
      </ul>
      <button onClick={() => setFruits(['Mango', ...fruits])}>
        Add Mango at start
      </button>
    </div>
  );
}

// Với key=index:
// Trước: [Apple(0), Banana(1), Cherry(2)]
// Sau:   [Mango(0), Apple(1), Banana(2), Cherry(3)]
// React thấy key 0,1,2 vẫn còn → UPDATE tất cả (tốn kém + bug state)
// Cherry(3) là NEW → create mới

// Với key=fruit:
// React thấy 'Apple', 'Banana', 'Cherry' vẫn còn → giữ nguyên
// 'Mango' là NEW → chỉ insert Mango (hiệu quả!)
```

---

### Ví dụ 2 — Trung cấp: Render phase vs Commit phase
```jsx
// Render phase: PURE — không side effects, có thể bị interrupt và retry
// Commit phase: có thể có side effects (DOM mutation, effects...)

function ExpensiveComponent({ data }) {
  // Render phase: chỉ tính toán, trả JSX
  // Có thể bị React interrupt nếu có higher priority work (Concurrent Mode)
  const processed = processData(data); // phải là pure, không side effects

  // useLayoutEffect: chạy SAU DOM mutation, TRƯỚC browser paint
  // Dùng khi cần measure DOM hoặc prevent visual flicker
  React.useLayoutEffect(() => {
    // Đây là commit phase — DOM đã cập nhật, chưa paint
    const height = ref.current.getBoundingClientRect().height;
    setHeight(height); // synchronous update trước paint
  });

  // useEffect: chạy SAU browser paint (async)
  // Dùng cho fetch, subscription, analytics
  React.useEffect(() => {
    // Đây là sau paint — safe cho side effects
    fetch('/api/track').then(() => {});
  });

  return <div ref={ref}>{processed}</div>;
}
```

---

### Ví dụ 3 — Nâng cao: Concurrent Features
```jsx
// startTransition: đánh dấu update là "non-urgent" → có thể bị interrupt
// Dùng cho: filter list lớn, navigation, search

function SearchPage() {
  const [query, setQuery] = React.useState('');
  const [results, setResults] = React.useState([]);
  const [isPending, startTransition] = React.useTransition();

  function handleSearch(e) {
    setQuery(e.target.value); // urgent: update input ngay

    startTransition(() => {
      // non-urgent: có thể delay nếu có input mới đến
      const filtered = expensiveFilter(allItems, e.target.value);
      setResults(filtered);
    });
  }

  return (
    <div>
      <input value={query} onChange={handleSearch} />
      {isPending && <Spinner />}
      <ResultList results={results} /> {/* có thể render stale trong lúc pending */}
    </div>
  );
}

// useDeferredValue: defer việc render với value mới
function FilteredList({ filter }) {
  const deferredFilter = React.useDeferredValue(filter);
  // filter = giá trị mới nhất (dùng cho input)
  // deferredFilter = có thể lag sau → dùng cho expensive computation
  
  const isStale = filter !== deferredFilter;

  return (
    <div style={{ opacity: isStale ? 0.6 : 1 }}>
      <ExpensiveList filter={deferredFilter} />
    </div>
  );
}
```

---

## 2. useState & useReducer

### Lý thuyết & Bản chất
`useState` là hook lưu trữ state trong một React component. Khi state thay đổi, React **schedule** re-render component.

**Bản chất:** React giữ state trong một "fiber node" liên kết với component instance. Mỗi lần render, `useState` trả về giá trị hiện tại từ fiber. Thứ tự gọi hooks phải nhất quán (lý do tại sao hooks không được gọi trong if/loop).

**Batching (React 18):** React tự động batch nhiều state updates vào 1 re-render dù trong async code (setTimeout, fetch callbacks...).

---

### Ví dụ 1 — Cơ bản: Functional update & Batching
```jsx
function Counter() {
  const [count, setCount] = React.useState(0);

  function handleClick() {
    // BAD: cả hai đọc cùng giá trị count (stale closure)
    setCount(count + 1); // count = 0 → set 1
    setCount(count + 1); // count vẫn = 0 → set 1 (không phải 2!)

    // GOOD: functional update — luôn nhận prev value mới nhất
    setCount(c => c + 1); // prev = 0 → set 1
    setCount(c => c + 1); // prev = 1 → set 2 ✅
  }

  // Lazy initialization — fn chỉ gọi 1 lần lúc mount
  const [data] = React.useState(() => {
    return JSON.parse(localStorage.getItem('data') ?? '[]'); // tránh parse mỗi render
  });

  return <button onClick={handleClick}>{count}</button>;
}
```

---

### Ví dụ 2 — Trung cấp: useReducer cho state phức tạp
```jsx
// useReducer tốt hơn useState khi:
// - Multiple related state values
// - Next state phụ thuộc previous state theo cách phức tạp
// - Logic update muốn tách riêng để test

const initialState = {
  items: [],
  loading: false,
  error: null,
  total: 0,
};

function cartReducer(state, action) {
  switch (action.type) {
    case 'ADD_ITEM': {
      const exists = state.items.find(i => i.id === action.item.id);
      const items = exists
        ? state.items.map(i => i.id === action.item.id
            ? { ...i, qty: i.qty + 1 }
            : i)
        : [...state.items, { ...action.item, qty: 1 }];
      return { ...state, items, total: calculateTotal(items) };
    }
    case 'REMOVE_ITEM': {
      const items = state.items.filter(i => i.id !== action.id);
      return { ...state, items, total: calculateTotal(items) };
    }
    case 'SET_LOADING':
      return { ...state, loading: action.loading };
    case 'SET_ERROR':
      return { ...state, error: action.error, loading: false };
    default:
      return state;
  }
}

function Cart() {
  const [state, dispatch] = React.useReducer(cartReducer, initialState);

  return (
    <div>
      {state.items.map(item => (
        <CartItem
          key={item.id}
          item={item}
          onRemove={() => dispatch({ type: 'REMOVE_ITEM', id: item.id })}
        />
      ))}
      <strong>Total: {state.total}</strong>
    </div>
  );
}
```

---

### Ví dụ 3 — Nâng cao: Immer + Complex state updates
```tsx
import { useImmerReducer } from 'use-immer'; // immer cho phép "mutate" state

interface AppState {
  users: Record<string, User>;
  selectedIds: Set<string>;
  ui: { sidebarOpen: boolean; theme: 'light' | 'dark' };
}

type Action =
  | { type: 'UPDATE_USER'; id: string; patch: Partial<User> }
  | { type: 'TOGGLE_SELECT'; id: string }
  | { type: 'TOGGLE_SIDEBAR' };

function appReducer(draft: AppState, action: Action) {
  // Immer: có thể mutate draft trực tiếp — bên dưới vẫn là immutable update
  switch (action.type) {
    case 'UPDATE_USER':
      Object.assign(draft.users[action.id], action.patch);
      break;
    case 'TOGGLE_SELECT':
      if (draft.selectedIds.has(action.id)) draft.selectedIds.delete(action.id);
      else draft.selectedIds.add(action.id);
      break;
    case 'TOGGLE_SIDEBAR':
      draft.ui.sidebarOpen = !draft.ui.sidebarOpen;
      break;
  }
}
```

---

## 3. useEffect

### Lý thuyết & Bản chất
`useEffect` là hook để thực hiện **side effects** — những việc "bên ngoài" React như: fetch data, subscribe, set timer, thao tác DOM.

**Thứ tự chạy:**
1. Component render (returns JSX)
2. React update DOM
3. Browser paint
4. `useEffect` cleanup (từ render trước) chạy
5. `useEffect` (render này) chạy

**Dependency array:**
- `[]`: chạy 1 lần sau mount
- `[a, b]`: chạy lại khi `a` hoặc `b` thay đổi (so sánh bằng `Object.is`)
- Không có: chạy sau MỖI render

---

### Ví dụ 1 — Cơ bản: Fetch data + Cleanup
```jsx
function UserProfile({ userId }) {
  const [user, setUser] = React.useState(null);
  const [loading, setLoading] = React.useState(true);

  React.useEffect(() => {
    let cancelled = false; // cleanup flag để tránh setState sau unmount

    setLoading(true);

    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(data => {
        if (!cancelled) { // chỉ update nếu effect vẫn còn active
          setUser(data);
          setLoading(false);
        }
      });

    return () => {
      cancelled = true; // cleanup: cancel khi userId thay đổi hoặc unmount
    };
  }, [userId]); // re-run khi userId thay đổi

  if (loading) return <Spinner />;
  return <div>{user?.name}</div>;
}
```

---

### Ví dụ 2 — Trung cấp: Subscription + Event listener
```jsx
function useWindowSize() {
  const [size, setSize] = React.useState({
    width: window.innerWidth,
    height: window.innerHeight,
  });

  React.useEffect(() => {
    function handleResize() {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    }

    window.addEventListener('resize', handleResize);

    // Cleanup: remove listener khi component unmount
    // Nếu không cleanup → memory leak + multiple listeners khi re-render
    return () => window.removeEventListener('resize', handleResize);
  }, []); // [] vì không phụ thuộc props/state

  return size;
}

// WebSocket subscription
function useChatRoom(roomId) {
  const [messages, setMessages] = React.useState([]);

  React.useEffect(() => {
    const ws = new WebSocket(`wss://chat.example.com/room/${roomId}`);
    
    ws.onmessage = (event) => {
      setMessages(prev => [...prev, JSON.parse(event.data)]);
    };

    return () => ws.close(); // cleanup: đóng connection khi roomId thay đổi
  }, [roomId]);

  return messages;
}
```

---

### Ví dụ 3 — Nâng cao: Effect dependencies deep dive + Avoiding stale closures
```jsx
// Stale closure: effect capture giá trị cũ
function Timer() {
  const [count, setCount] = React.useState(0);

  // BUG: count luôn là 0 trong effect vì dependency array rỗng
  // count bị "đóng băng" giá trị lúc effect tạo ra
  React.useEffect(() => {
    const id = setInterval(() => {
      console.log(count); // luôn in 0!
      setCount(count + 1); // luôn set 1!
    }, 1000);
    return () => clearInterval(id);
  }, []); // ❌ missing dependency

  // FIX 1: Dùng functional update (không cần đọc state trong effect)
  React.useEffect(() => {
    const id = setInterval(() => {
      setCount(c => c + 1); // không cần đọc count từ closure
    }, 1000);
    return () => clearInterval(id);
  }, []);

  // FIX 2: useRef để đọc giá trị mới nhất
  const countRef = React.useRef(count);
  countRef.current = count; // cập nhật ref mỗi render

  React.useEffect(() => {
    const id = setInterval(() => {
      console.log(countRef.current); // luôn là giá trị mới nhất
    }, 1000);
    return () => clearInterval(id);
  }, []); // OK vì ref.current không trigger re-render
}

// useEffectEvent (React 19 / experimental): giải quyết vấn đề này sạch hơn
// const onTick = useEffectEvent(() => { console.log(count); });
```

---

## 4. useMemo & useCallback

### Lý thuyết & Bản chất
Cả hai đều là **memoization** hooks — cache giá trị để tránh tính toán lại.

- **`useMemo`**: Cache **giá trị**. Recompute khi deps thay đổi.
- **`useCallback`**: Cache **function reference**. Trả về function mới khi deps thay đổi.

**QUAN TRỌNG:** `useCallback(fn, deps)` === `useMemo(() => fn, deps)`

**Khi nào dùng?**
- `useMemo`: Tính toán expensive (sort/filter list lớn, heavy computation)
- `useCallback`: Function truyền vào child component được wrap bởi `React.memo`, hoặc dùng trong deps của `useEffect`/`useMemo` khác

**Khi nào KHÔNG dùng?**
- Computation đơn giản (tốn overhead hơn là tiết kiệm)
- Component re-render không phải vấn đề
- Premature optimization → thêm complexity

---

### Ví dụ 1 — Cơ bản: Tránh re-render không cần thiết
```jsx
// React.memo: HOC wrap component, re-render CHỈ khi props thay đổi (shallow compare)
const ExpensiveChild = React.memo(function({ name, onClick }) {
  console.log('ExpensiveChild render');
  return <button onClick={onClick}>{name}</button>;
});

function Parent() {
  const [count, setCount] = React.useState(0);
  const [name] = React.useState('Alice');

  // BAD: mỗi render của Parent tạo object mới → ExpensiveChild luôn re-render
  // dù React.memo — vì {} !== {} (reference equality)
  const style = { color: 'red' };

  // GOOD: useMemo → cùng reference khi deps không đổi
  const style2 = React.useMemo(() => ({ color: 'red' }), []);

  // BAD: mỗi render của Parent tạo function mới → phá vỡ React.memo
  const handleClick = () => console.log(name);

  // GOOD: useCallback → cùng reference khi deps không đổi
  const handleClick2 = React.useCallback(() => console.log(name), [name]);

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <ExpensiveChild name={name} onClick={handleClick2} />
    </div>
  );
}
```

---

### Ví dụ 2 — Trung cấp: useMemo cho expensive computation
```jsx
function ProductCatalog({ products, searchQuery, sortBy, filterCategory }) {
  // Tính toán này có thể tốn kém với list lớn
  const processedProducts = React.useMemo(() => {
    console.log('Recomputing processed products...');
    
    return products
      .filter(p => {
        const matchesSearch = p.name.toLowerCase().includes(searchQuery.toLowerCase());
        const matchesCategory = !filterCategory || p.category === filterCategory;
        return matchesSearch && matchesCategory;
      })
      .sort((a, b) => {
        if (sortBy === 'price') return a.price - b.price;
        if (sortBy === 'name') return a.name.localeCompare(b.name);
        return 0;
      });
  }, [products, searchQuery, sortBy, filterCategory]);
  // Chỉ recompute khi 1 trong 4 deps này thay đổi
  // Nếu chỉ thay đổi unrelated state → không tính lại

  return <ProductList products={processedProducts} />;
}
```

---

### Ví dụ 3 — Nâng cao: useCallback trong custom hooks + avoiding infinite loops
```jsx
// Vấn đề: useCallback trong custom hook
function useSearch(query) {
  const [results, setResults] = React.useState([]);

  // Nếu không có useCallback → fetchResults tạo mới mỗi render
  // → useEffect deps thay đổi → re-fetch → re-render → loop!
  const fetchResults = React.useCallback(async () => {
    const data = await searchAPI(query);
    setResults(data);
  }, [query]); // chỉ tạo mới khi query thay đổi

  React.useEffect(() => {
    fetchResults();
  }, [fetchResults]); // safe để include vì useCallback ổn định

  return results;
}

// Selector pattern với useCallback
function useSelector<T, S>(store: Store<T>, selector: (state: T) => S): S {
  const selectorRef = React.useRef(selector);
  selectorRef.current = selector; // luôn mới nhất

  const [selected, setSelected] = React.useState(() => selector(store.getState()));

  React.useEffect(() => {
    // subscribe không phụ thuộc selector — tránh re-subscribe
    return store.subscribe(() => {
      const newSelected = selectorRef.current(store.getState());
      setSelected(prev => Object.is(prev, newSelected) ? prev : newSelected);
    });
  }, [store]); // chỉ re-subscribe khi store thay đổi

  return selected;
}
```

---

## 5. useRef

### Lý thuyết & Bản chất
`useRef` trả về một **mutable ref object** `{ current: initialValue }` tồn tại suốt lifecycle của component. Thay đổi `ref.current` **KHÔNG trigger re-render**.

**2 use cases chính:**
1. **DOM access**: trỏ đến DOM element
2. **Mutable value**: lưu giá trị thay đổi theo thời gian nhưng không cần trigger render (timer ID, previous value, abortController...)

---

### Ví dụ 1 — Cơ bản: DOM manipulation
```jsx
function FocusInput() {
  const inputRef = React.useRef(null);

  // Imperative action: focus, scroll, measure, animate
  function focusInput() {
    inputRef.current?.focus();
  }

  function scrollToElement() {
    inputRef.current?.scrollIntoView({ behavior: 'smooth' });
  }

  return (
    <div>
      <input ref={inputRef} type="text" />
      <button onClick={focusInput}>Focus</button>
    </div>
  );
}

// Forwarding ref: truyền ref từ parent vào DOM element trong child
const Input = React.forwardRef(function({ label, ...props }, ref) {
  return (
    <label>
      {label}
      <input ref={ref} {...props} />
    </label>
  );
});

function Form() {
  const ref = React.useRef(null);
  return <Input ref={ref} label="Email" />;
}
```

---

### Ví dụ 2 — Trung cấp: Previous value + Interval management
```jsx
// usePrevious hook — pattern phổ biến
function usePrevious(value) {
  const ref = React.useRef(undefined);

  React.useEffect(() => {
    ref.current = value; // update sau render → ref.current là giá trị render TRƯỚC
  });

  return ref.current; // trả về giá trị render trước
}

function Counter() {
  const [count, setCount] = React.useState(0);
  const prevCount = usePrevious(count);

  return (
    <div>
      Current: {count}, Previous: {prevCount}
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}

// Timer management với ref
function Stopwatch() {
  const [time, setTime] = React.useState(0);
  const [running, setRunning] = React.useState(false);
  const intervalRef = React.useRef(null); // lưu timer ID, không cần re-render khi thay đổi

  function start() {
    if (running) return;
    setRunning(true);
    intervalRef.current = setInterval(() => setTime(t => t + 1), 1000);
  }

  function stop() {
    clearInterval(intervalRef.current);
    setRunning(false);
  }

  React.useEffect(() => {
    return () => clearInterval(intervalRef.current); // cleanup on unmount
  }, []);

  return <div>{time}s <button onClick={start}>Start</button> <button onClick={stop}>Stop</button></div>;
}
```

---

### Ví dụ 3 — Nâng cao: useImperativeHandle + Abort Controller
```jsx
// useImperativeHandle: kiểm soát những gì expose qua ref
const VideoPlayer = React.forwardRef(function VideoPlayer({ src }, ref) {
  const videoRef = React.useRef(null);

  // Thay vì expose toàn bộ DOM element, chỉ expose các method cụ thể
  React.useImperativeHandle(ref, () => ({
    play() { videoRef.current.play(); },
    pause() { videoRef.current.pause(); },
    seek(time) { videoRef.current.currentTime = time; },
    // videoRef.current.innerHTML = ... không accessible từ parent!
  }));

  return <video ref={videoRef} src={src} />;
});

// Parent
function App() {
  const playerRef = React.useRef(null);
  return (
    <div>
      <VideoPlayer ref={playerRef} src="/video.mp4" />
      <button onClick={() => playerRef.current.play()}>Play</button>
    </div>
  );
}

// AbortController với useRef
function useAbortableFetch(url) {
  const [data, setData] = React.useState(null);
  const abortRef = React.useRef(null);

  React.useEffect(() => {
    abortRef.current?.abort(); // abort request trước (nếu có)
    abortRef.current = new AbortController();

    fetch(url, { signal: abortRef.current.signal })
      .then(r => r.json())
      .then(setData)
      .catch(err => {
        if (err.name !== 'AbortError') throw err; // bỏ qua abort error
      });

    return () => abortRef.current?.abort();
  }, [url]);

  return data;
}
```

---

## 6. CUSTOM HOOKS

### Lý thuyết & Bản chất
Custom Hook là một **function JavaScript bắt đầu bằng `use`** và có thể gọi các React hooks khác. Mục đích: **tái sử dụng stateful logic** giữa các component mà không cần share state.

**Khác với component:** Custom hook không trả về JSX, chỉ trả về data/functions.

**Khác với utility function:** Custom hook có thể dùng React hooks (`useState`, `useEffect`...).

---

### Ví dụ 1 — Cơ bản: useLocalStorage
```jsx
function useLocalStorage(key, initialValue) {
  const [storedValue, setStoredValue] = React.useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = React.useCallback((value) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error(error);
    }
  }, [key, storedValue]);

  return [storedValue, setValue];
}

// Dùng như useState nhưng persistent
function Settings() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');
  return <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>{theme}</button>;
}
```

---

### Ví dụ 2 — Trung cấp: useAsync — wrapper cho async operations
```tsx
type AsyncStatus = 'idle' | 'loading' | 'success' | 'error';

interface AsyncState<T> {
  status: AsyncStatus;
  data: T | null;
  error: Error | null;
}

function useAsync<T>(asyncFn: () => Promise<T>, deps: React.DependencyList) {
  const [state, setState] = React.useState<AsyncState<T>>({
    status: 'idle',
    data: null,
    error: null,
  });

  React.useEffect(() => {
    let cancelled = false;
    setState({ status: 'loading', data: null, error: null });

    asyncFn()
      .then(data => {
        if (!cancelled) setState({ status: 'success', data, error: null });
      })
      .catch(error => {
        if (!cancelled) setState({ status: 'error', data: null, error });
      });

    return () => { cancelled = true; };
  }, deps); // eslint-disable-line react-hooks/exhaustive-deps

  return state;
}

// Dùng
function UserList() {
  const { status, data: users, error } = useAsync(() => fetchUsers(), []);

  if (status === 'loading') return <Spinner />;
  if (status === 'error') return <div>Error: {error.message}</div>;
  if (status === 'success') return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
  return null;
}
```

---

### Ví dụ 3 — Nâng cao: useIntersectionObserver + useVirtualList
```jsx
// useIntersectionObserver — infinite scroll, lazy load images
function useIntersectionObserver(options = {}) {
  const [isIntersecting, setIsIntersecting] = React.useState(false);
  const ref = React.useRef(null);

  React.useEffect(() => {
    const element = ref.current;
    if (!element) return;

    const observer = new IntersectionObserver(([entry]) => {
      setIsIntersecting(entry.isIntersecting);
    }, options);

    observer.observe(element);
    return () => observer.disconnect();
  }, [options.threshold, options.root, options.rootMargin]);

  return [ref, isIntersecting];
}

// Lazy image
function LazyImage({ src, alt }) {
  const [ref, isVisible] = useIntersectionObserver({ threshold: 0.1 });
  const [loaded, setLoaded] = React.useState(false);

  return (
    <div ref={ref}>
      {isVisible && (
        <img
          src={src}
          alt={alt}
          onLoad={() => setLoaded(true)}
          style={{ opacity: loaded ? 1 : 0, transition: 'opacity 0.3s' }}
        />
      )}
    </div>
  );
}

// useDebounce — tránh quá nhiều calls khi user typing
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = React.useState(value);

  React.useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

function SearchInput() {
  const [query, setQuery] = React.useState('');
  const debouncedQuery = useDebounce(query, 300);

  const { data } = useAsync(() => searchAPI(debouncedQuery), [debouncedQuery]);

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <SearchResults results={data} />
    </div>
  );
}
```

---

## TỔNG KẾT — React Hooks Quick Reference

| Hook | Bản chất | Khi dùng | Pitfall |
|------|----------|----------|---------|
| `useState` | Local state, trigger re-render | Simple state | Dùng functional update khi state phụ thuộc state cũ |
| `useReducer` | State machine, pure reducer | Complex state, nhiều actions | Không mutate state trực tiếp |
| `useEffect` | Side effects sau render | Fetch, subscribe, DOM | Cleanup, missing deps, stale closure |
| `useMemo` | Cache giá trị | Expensive computation | Không phải silver bullet, có overhead |
| `useCallback` | Cache function reference | Truyền vào memo child | Chỉ có ý nghĩa nếu child được memo |
| `useRef` | Mutable, không trigger render | DOM access, timer, prev value | Đọc trong render → có thể stale |
| Custom Hook | Tái sử dụng logic | Bất kỳ logic stateful nào | Tên phải bắt đầu bằng `use` |
