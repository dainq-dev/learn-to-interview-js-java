# React — Lifecycle, Full Hooks & React 19 Features

---

## PHẦN 1 — REACT LIFECYCLE

### 1.1 Class Component Lifecycle — Toàn bộ

```
MOUNT:
  constructor()
      ↓
  static getDerivedStateFromProps()
      ↓
  render()
      ↓
  componentDidMount()

UPDATE (state/props thay đổi):
  static getDerivedStateFromProps()
      ↓
  shouldComponentUpdate()  ← return false → dừng, không re-render
      ↓
  render()
      ↓
  getSnapshotBeforeUpdate()  ← chụp DOM trước khi update
      ↓
  componentDidUpdate()

UNMOUNT:
  componentWillUnmount()

ERROR:
  static getDerivedStateFromError()   ← render fallback UI
  componentDidCatch()                 ← log error
```

---

### Ví dụ 1 — Cơ bản: Class lifecycle cơ bản
```jsx
class Timer extends React.Component {
  constructor(props) {
    super(props);
    // Khởi tạo state, bind methods
    // KHÔNG gọi fetch hoặc subscriptions ở đây
    this.state = { seconds: 0 };
    this.handleClick = this.handleClick.bind(this);
  }

  componentDidMount() {
    // DOM đã sẵn sàng, safe để:
    // - Fetch data
    // - Subscribe (WebSocket, EventEmitter)
    // - Set up timer
    this.interval = setInterval(() => {
      this.setState(prev => ({ seconds: prev.seconds + 1 }));
    }, 1000);
  }

  componentDidUpdate(prevProps, prevState) {
    // Chạy sau mỗi update, nhận giá trị TRƯỚC update
    if (prevProps.userId !== this.props.userId) {
      // props thay đổi → fetch lại data
      this.fetchUser(this.props.userId);
    }
    if (prevState.seconds !== this.state.seconds && this.state.seconds === 60) {
      console.log('One minute passed!');
    }
  }

  componentWillUnmount() {
    // Cleanup: clear timer, unsubscribe, cancel fetch
    clearInterval(this.interval);
  }

  handleClick() {
    this.setState({ seconds: 0 });
  }

  render() {
    return (
      <div>
        <span>{this.state.seconds}s</span>
        <button onClick={this.handleClick}>Reset</button>
      </div>
    );
  }
}
```

---

### Ví dụ 2 — Trung cấp: shouldComponentUpdate + getSnapshotBeforeUpdate
```jsx
class OptimizedList extends React.Component {
  // Kiểm soát re-render thủ công
  // Trả về false → bỏ qua render này
  shouldComponentUpdate(nextProps, nextState) {
    // Chỉ re-render nếu items hoặc selectedId thay đổi
    return (
      nextProps.items !== this.props.items ||
      nextState.selectedId !== this.state.selectedId
    );
    // PureComponent làm điều này tự động (shallow compare)
    // React.memo là tương đương cho function components
  }

  getSnapshotBeforeUpdate(prevProps, prevState) {
    // Gọi TRƯỚC khi React apply DOM changes
    // Return value được truyền vào componentDidUpdate thứ 3
    // Use case: lưu scroll position trước khi list thay đổi
    if (prevProps.items.length < this.props.items.length) {
      const list = this.listRef.current;
      return list.scrollHeight - list.scrollTop;
    }
    return null;
  }

  componentDidUpdate(prevProps, prevState, snapshot) {
    // snapshot = giá trị trả về từ getSnapshotBeforeUpdate
    // Restore scroll position sau khi items mới được thêm vào đầu list
    if (snapshot !== null) {
      const list = this.listRef.current;
      list.scrollTop = list.scrollHeight - snapshot;
    }
  }

  render() {
    return (
      <ul ref={this.listRef}>
        {this.props.items.map(item => <li key={item.id}>{item.name}</li>)}
      </ul>
    );
  }
}
```

---

### Ví dụ 3 — Nâng cao: Error Boundary (chỉ có thể làm với class)
```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null, errorInfo: null };

  // Gọi khi có error trong child tree
  // Trả về state mới để render fallback → không được có side effects
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  // Gọi sau khi getDerivedStateFromError
  // Được phép có side effects (log error)
  componentDidCatch(error, errorInfo) {
    // errorInfo.componentStack: stack trace của component tree
    console.error('Uncaught error:', error, errorInfo);
    logErrorToService(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div>
          <h2>Something went wrong.</h2>
          <details style={{ whiteSpace: 'pre-wrap' }}>
            {this.state.error?.message}
            {this.state.errorInfo?.componentStack}
          </details>
          <button onClick={() => this.setState({ hasError: false, error: null })}>
            Try again
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}

// Dùng
function App() {
  return (
    <ErrorBoundary>
      <ProductList />
    </ErrorBoundary>
  );
}

// Lưu ý: Error Boundary CHỈ bắt lỗi trong:
// - render()
// - lifecycle methods
// - constructors của child components
// KHÔNG bắt: event handlers, async code, server-side errors
// → Dùng try/catch trong event handlers và async code
```

---

### 1.2 Functional Component Lifecycle — So sánh với Class

```
CLASS                          FUNCTIONAL (hooks)
──────────────────────────────────────────────────────
constructor()              →   useState(initialValue) / useReducer
                               (lazy init: useState(() => compute()))

componentDidMount()        →   useEffect(() => { ... }, [])
                               useLayoutEffect(() => { ... }, [])

componentDidUpdate()       →   useEffect(() => { ... }, [deps])

componentWillUnmount()     →   useEffect cleanup: return () => { ... }

shouldComponentUpdate()    →   React.memo + useMemo + useCallback

getDerivedStateFromProps() →   Không cần — tính trực tiếp trong render
                               hoặc useMemo

getSnapshotBeforeUpdate()  →   useLayoutEffect (gần nhất, nhưng không hoàn toàn tương đương)

componentDidCatch()        →   Không có — vẫn cần class ErrorBoundary
getDerivedStateFromError() →   Không có — vẫn cần class ErrorBoundary
                               (React 19 có hook mới: use(ErrorBoundary))
```

---

### 1.3 Functional Lifecycle Timeline chi tiết

```
MOUNT:
  1. Function body chạy (render) — initial state từ useState
  2. React apply DOM (commit)
  3. useLayoutEffect run (sync, blocking paint)
  4. Browser paint
  5. useEffect run (async, after paint)

UPDATE (state/props thay đổi):
  1. Function body chạy lại (re-render)
     - useMemo recompute nếu deps thay đổi
     - useCallback tạo mới nếu deps thay đổi
  2. React diff và apply DOM changes
  3. useLayoutEffect cleanup (sync)
  4. useLayoutEffect run (sync)
  5. Browser paint
  6. useEffect cleanup (từ render trước)
  7. useEffect run

UNMOUNT:
  1. useLayoutEffect cleanup (sync)
  2. useEffect cleanup
  (DOM vẫn còn cho đến sau cleanup)

CONCURRENT MODE thêm:
  - Render phase có thể chạy nhiều lần (double invoke dev mode)
  - Render phase có thể interrupt và retry
  - Commit phase luôn synchronous
```

```jsx
function LifecycleDemo({ id }) {
  console.log('1. render'); // mỗi render

  const [data, setData] = useState(() => {
    console.log('1a. lazy init (chỉ 1 lần)');
    return null;
  });

  // Tương đương componentDidMount + componentWillUnmount
  useEffect(() => {
    console.log('5. effect (after paint)');
    return () => console.log('5c. effect cleanup (before next effect or unmount)');
  }, []);

  // Tương đương componentDidMount + componentDidUpdate(id) + componentWillUnmount
  useEffect(() => {
    console.log('6. effect with deps, id =', id);
    return () => console.log('6c. cleanup for id', id);
  }, [id]);

  useLayoutEffect(() => {
    console.log('3. layoutEffect (before paint, after DOM)');
    return () => console.log('3c. layoutEffect cleanup');
  }, [id]);

  console.log('2. return JSX');
  return <div>{id}</div>;
}

// Mount sequence:
// 1. render
// 1a. lazy init
// 2. return JSX
// → DOM update
// 3. layoutEffect
// → browser paint
// 5. effect
// 6. effect with deps

// Update (id thay đổi):
// 1. render
// 2. return JSX
// → DOM update
// 3c. layoutEffect cleanup
// 3. layoutEffect
// → paint
// 5c. effect cleanup (nếu deps rỗng vẫn cleanup khi unmount thôi)
// 6c. cleanup for old id
// 6. effect with new id
```

---

---

## PHẦN 2 — FULL REACT HOOKS REFERENCE

### 2.1 State Hooks

#### `useState`
```jsx
const [state, setState] = useState(initialValue);
const [state, setState] = useState(() => expensiveCompute()); // lazy init

// Functional update — khi state mới phụ thuộc state cũ
setState(prev => prev + 1);

// Gotcha: setState với object KHÔNG merge — phải spread
setState(prev => ({ ...prev, name: 'Alice' }));
```

#### `useReducer`
```jsx
const [state, dispatch] = useReducer(reducer, initialArg, init?);
// init function: useReducer(reducer, arg, init) → state = init(arg)
// Giống useState nhưng tốt hơn khi:
// - State phức tạp (nhiều sub-values)
// - State update phụ thuộc state cũ theo nhiều cách
// - Muốn tách logic update ra ngoài component
```

---

### 2.2 Context Hooks

#### `useContext`
```jsx
const value = useContext(MyContext);
// Re-render khi context value thay đổi
// Không có selector → re-render dù chỉ dùng 1 phần
// FIX: tách context, dùng useMemo, hoặc dùng state library
```

---

### 2.3 Ref Hooks

#### `useRef`
```jsx
const ref = useRef(initialValue);
// ref.current = initialValue
// Thay đổi ref.current KHÔNG trigger re-render
// 2 use cases: DOM access + mutable instance variable
```

#### `useImperativeHandle`
```jsx
// Dùng với forwardRef — kiểm soát những gì expose qua ref
const Input = forwardRef((props, ref) => {
  const inputRef = useRef();
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    // Chỉ expose focus(), không expose toàn bộ DOM element
  }), []); // deps array
  return <input ref={inputRef} />;
});
```

---

### 2.4 Effect Hooks

#### `useEffect`
```jsx
useEffect(() => {
  // side effect
  return () => { /* cleanup */ };
}, [deps]);
// Chạy SAU browser paint (async)
// Dùng cho: data fetching, subscriptions, analytics, timers
```

#### `useLayoutEffect`
```jsx
useLayoutEffect(() => {
  // Giống useEffect nhưng chạy TRƯỚC browser paint (sync)
  // Dùng cho: measure DOM, prevent visual flicker
  const { height } = ref.current.getBoundingClientRect();
  setHeight(height); // update trước khi user thấy
}, [deps]);
```

#### `useInsertionEffect` *(React 18)*
```jsx
useInsertionEffect(() => {
  // Chạy TRƯỚC useLayoutEffect và DOM mutations
  // CHỈ DÙNG trong CSS-in-JS libraries (styled-components, Emotion)
  // Inject styles trước khi component đọc layout
}, [deps]);

// Thứ tự: useInsertionEffect → DOM update → useLayoutEffect → paint → useEffect
```

---

### 2.5 Performance Hooks

#### `useMemo`
```jsx
const memoizedValue = useMemo(() => expensiveCompute(a, b), [a, b]);
// Cache giá trị, recompute khi deps thay đổi
// Dùng cho: expensive computation, stable object/array reference
```

#### `useCallback`
```jsx
const memoizedFn = useCallback(() => doSomething(id), [id]);
// Cache function reference
// === useMemo(() => fn, deps)
// Dùng khi: function là prop của memo'd child, hoặc là dep của effect
```

#### `useTransition` *(React 18)*
```jsx
const [isPending, startTransition] = useTransition();
// Đánh dấu state update là non-urgent
// React có thể interrupt và ưu tiên urgent updates (input, click)
startTransition(() => {
  setResults(heavyFilter(data, query));
});
// isPending: true khi transition đang chờ
```

#### `useDeferredValue` *(React 18)*
```jsx
const deferredQuery = useDeferredValue(query);
// Tương tự useTransition nhưng cho value, không phải setter
// React giữ giá trị cũ cho đến khi không có urgent work
// Dùng khi không control được setter (prop từ parent)
const results = useMemo(() => filter(data, deferredQuery), [data, deferredQuery]);
```

---

### 2.6 Other Hooks

#### `useId` *(React 18)*
```jsx
const id = useId();
// Tạo unique ID stable giữa server và client (tránh hydration mismatch)
// KHÔNG dùng cho list keys (dùng data IDs)
return (
  <div>
    <label htmlFor={id}>Name</label>
    <input id={id} />
  </div>
);
```

#### `useSyncExternalStore` *(React 18)*
```jsx
const value = useSyncExternalStore(
  subscribe,      // (callback) => unsubscribe — đăng ký lắng nghe store
  getSnapshot,    // () => currentValue — đọc current value
  getServerSnapshot? // cho SSR
);
// Dùng để integrate với external stores (Redux, Zustand, browser APIs)
// React đảm bảo consistent reads trong Concurrent Mode

// Ví dụ: theo dõi network status
const isOnline = useSyncExternalStore(
  cb => { window.addEventListener('online', cb); window.addEventListener('offline', cb); return () => { window.removeEventListener('online', cb); window.removeEventListener('offline', cb); }; },
  () => navigator.onLine,
  () => true // server luôn "online"
);
```

#### `useDebugValue` *(development only)*
```jsx
function useCustomHook(value) {
  useDebugValue(value, v => `Formatted: ${v}`);
  // Hiển thị label trong React DevTools
  // Second arg: formatter fn (expensive format chỉ chạy khi DevTools mở)
}
```

---

### 2.7 React 19 — NEW HOOKS

#### `use` — Universal Resource Hook *(React 19)*
```jsx
import { use } from 'react';

// 1. Đọc Context (thay thế useContext)
function Component() {
  const theme = use(ThemeContext); // giống useContext nhưng flexible hơn
}

// 2. Đọc Promise — CÓ THỂ gọi trong conditions/loops!
function UserProfile({ userPromise }) {
  // use() "unwrap" Promise — suspend component cho đến khi resolve
  // Phải wrap trong <Suspense>
  const user = use(userPromise);
  return <div>{user.name}</div>;
}

// Dùng với Suspense
function App() {
  const userPromise = fetchUser(1); // tạo Promise ở ngoài component (không trong render)
  return (
    <Suspense fallback={<Spinner />}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}

// QUAN TRỌNG: Khác với các hooks khác:
// use() CÓ THỂ gọi trong conditions và loops!
function Component({ condition }) {
  if (condition) {
    const user = use(UserContext); // ✅ OK với use(), KHÔNG OK với useContext
  }
}
```

#### `useFormStatus` *(React 19 + react-dom)*
```jsx
import { useFormStatus } from 'react-dom';

// Đọc trạng thái của form CHA gần nhất
// PHẢI nằm trong component con của <form>
function SubmitButton() {
  const { pending, data, method, action } = useFormStatus();
  // pending: true khi form đang submit
  // data: FormData của form
  return (
    <button disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  );
}

function LoginForm() {
  return (
    <form action={loginAction}> {/* Server Action */}
      <input name="email" type="email" />
      <input name="password" type="password" />
      <SubmitButton /> {/* useFormStatus đọc trạng thái của form này */}
    </form>
  );
}
```

#### `useActionState` *(React 19 — đổi tên từ useFormState)*
```jsx
import { useActionState } from 'react';

// Wrap một action, theo dõi state của nó
async function submitAction(prevState, formData) {
  const name = formData.get('name');
  if (!name) return { error: 'Name is required' };
  await saveToDatabase(name);
  return { success: true, message: `Saved ${name}` };
}

function MyForm() {
  const [state, formAction, isPending] = useActionState(submitAction, null);
  // state: giá trị trả về từ action (hoặc initialState lúc đầu)
  // formAction: truyền vào form action prop
  // isPending: true khi action đang chạy

  return (
    <form action={formAction}>
      <input name="name" />
      {state?.error && <p className="error">{state.error}</p>}
      {state?.success && <p className="success">{state.message}</p>}
      <button disabled={isPending}>
        {isPending ? 'Saving...' : 'Save'}
      </button>
    </form>
  );
}
```

#### `useOptimistic` *(React 19)*
```jsx
import { useOptimistic } from 'react';

// Hiện thị UI optimistically trước khi async action hoàn thành
function MessageList({ messages, sendMessage }) {
  const [optimisticMessages, addOptimisticMessage] = useOptimistic(
    messages,
    // Reducer: (currentState, optimisticValue) => newState
    (currentMessages, newMessage) => [
      ...currentMessages,
      { ...newMessage, pending: true }, // đánh dấu pending
    ]
  );

  async function handleSubmit(formData) {
    const message = formData.get('message');

    // Cập nhật UI ngay lập tức
    addOptimisticMessage({ id: Date.now(), text: message });

    // Gửi lên server (async)
    await sendMessage(message);
    // Khi xong, React tự động sync với messages prop thật
    // optimisticMessages tự reset về messages sau khi action hoàn thành
  }

  return (
    <div>
      {optimisticMessages.map(msg => (
        <div key={msg.id} style={{ opacity: msg.pending ? 0.5 : 1 }}>
          {msg.text}
          {msg.pending && ' (sending...)'}
        </div>
      ))}
      <form action={handleSubmit}>
        <input name="message" />
        <button type="submit">Send</button>
      </form>
    </div>
  );
}
```

---

---

## PHẦN 3 — REACT 19 NEW FEATURES

### 3.1 Actions — Async Transitions

#### Lý thuyết & Bản chất
React 19 giới thiệu **Actions** — convention cho async functions được truyền vào form `action` prop hoặc các transition APIs. Actions tự động quản lý: pending states, error handling, optimistic updates.

**Trước React 19:**
```jsx
// Phải manually manage tất cả states
function UpdateName() {
  const [name, setName] = useState('');
  const [isPending, setIsPending] = useState(false);
  const [error, setError] = useState(null);

  async function handleSubmit() {
    setIsPending(true);
    setError(null);
    try {
      await updateName(name);
    } catch (err) {
      setError(err.message);
    } finally {
      setIsPending(false);
    }
  }
  // ...
}
```

---

### Ví dụ 1 — Cơ bản: Form với Actions
```jsx
// React 19: form action trực tiếp là async function
function UpdateNameForm() {
  const [error, setError] = useState(null);

  // "Action": async function dùng với form
  async function updateNameAction(formData) {
    const newName = formData.get('name');
    const error = await updateProfile({ name: newName });
    if (error) setError(error);
  }

  return (
    <form action={updateNameAction}>
      {/* useFormStatus trong SubmitButton tự detect pending */}
      <input name="name" />
      <SubmitButton />
      {error && <p>{error}</p>}
    </form>
  );
}

// useFormStatus — đọc pending state của form cha
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? 'Updating...' : 'Update'}</button>;
}
```

---

### Ví dụ 2 — Trung cấp: useActionState đầy đủ
```jsx
import { useActionState } from 'react';

// Action function — nhận prevState + formData
async function registerAction(prevState, formData) {
  const email = formData.get('email');
  const password = formData.get('password');

  // Validation
  if (!email.includes('@')) {
    return { success: false, error: 'Invalid email', field: 'email' };
  }
  if (password.length < 8) {
    return { success: false, error: 'Password too short', field: 'password' };
  }

  try {
    await createUser({ email, password });
    return { success: true, message: 'Account created!' };
  } catch (err) {
    if (err.code === 'DUPLICATE_EMAIL') {
      return { success: false, error: 'Email already exists', field: 'email' };
    }
    return { success: false, error: 'Something went wrong' };
  }
}

function RegisterForm() {
  const [state, formAction, isPending] = useActionState(
    registerAction,
    null // initial state
  );

  if (state?.success) {
    return <div className="success">{state.message}</div>;
  }

  return (
    <form action={formAction}>
      <div>
        <input
          name="email"
          type="email"
          aria-invalid={state?.field === 'email'}
        />
        {state?.field === 'email' && (
          <span className="error">{state.error}</span>
        )}
      </div>
      <div>
        <input
          name="password"
          type="password"
          aria-invalid={state?.field === 'password'}
        />
        {state?.field === 'password' && (
          <span className="error">{state.error}</span>
        )}
      </div>
      {state?.error && !state.field && (
        <div className="error">{state.error}</div>
      )}
      <button disabled={isPending}>
        {isPending ? 'Creating account...' : 'Register'}
      </button>
    </form>
  );
}
```

---

### Ví dụ 3 — Nâng cao: useOptimistic + useTransition kết hợp
```jsx
// Todo app với optimistic updates
function TodoApp() {
  const [todos, setTodos] = useState([]);
  const [isPending, startTransition] = useTransition();

  const [optimisticTodos, updateOptimisticTodos] = useOptimistic(
    todos,
    (state, action) => {
      switch (action.type) {
        case 'ADD':
          return [...state, { ...action.todo, optimistic: true }];
        case 'TOGGLE':
          return state.map(t =>
            t.id === action.id ? { ...t, completed: !t.completed, optimistic: true } : t
          );
        case 'DELETE':
          return state.filter(t => t.id !== action.id);
        default:
          return state;
      }
    }
  );

  async function addTodo(text) {
    const tempId = crypto.randomUUID();
    const newTodo = { id: tempId, text, completed: false };

    startTransition(async () => {
      // Optimistic update ngay lập tức
      updateOptimisticTodos({ type: 'ADD', todo: newTodo });

      try {
        // Actual server call
        const savedTodo = await createTodo({ text });
        // Server trả về ID thật → replace temp todo
        setTodos(prev => [...prev.filter(t => t.id !== tempId), savedTodo]);
      } catch (error) {
        // Optimistic state tự rollback khi transition kết thúc
        toast.error('Failed to add todo');
      }
    });
  }

  async function toggleTodo(id) {
    startTransition(async () => {
      updateOptimisticTodos({ type: 'TOGGLE', id });
      try {
        await toggleTodoApi(id);
        setTodos(prev => prev.map(t =>
          t.id === id ? { ...t, completed: !t.completed } : t
        ));
      } catch {
        toast.error('Failed to update');
      }
    });
  }

  return (
    <div>
      <AddTodoForm onAdd={addTodo} />
      <ul style={{ opacity: isPending ? 0.7 : 1 }}>
        {optimisticTodos.map(todo => (
          <li
            key={todo.id}
            style={{ opacity: todo.optimistic ? 0.6 : 1 }}
            onClick={() => toggleTodo(todo.id)}
          >
            <input type="checkbox" checked={todo.completed} readOnly />
            {todo.text}
            {todo.optimistic && ' •'}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

### 3.2 ref như prop thông thường *(React 19)*

#### Lý thuyết & Bản chất
Trước React 19: `ref` là "reserved prop" — phải dùng `forwardRef` để truyền ref xuống child. React 19 cho phép truyền `ref` như prop bình thường.

---

### Ví dụ 1 — Cơ bản
```jsx
// TRƯỚC React 19: phải dùng forwardRef
const Input = forwardRef(function Input({ label, ...props }, ref) {
  return (
    <label>
      {label}
      <input ref={ref} {...props} />
    </label>
  );
});

// REACT 19: ref là prop bình thường — forwardRef không còn cần thiết
function Input({ label, ref, ...props }) {
  return (
    <label>
      {label}
      <input ref={ref} {...props} />
    </label>
  );
}

// Dùng giống nhau
function Form() {
  const inputRef = useRef(null);
  return <Input ref={inputRef} label="Name" />;
}
```

---

### Ví dụ 2 — Trung cấp: ref callback cleanup
```jsx
// React 19: ref callback có thể return cleanup function
function MeasuredComponent() {
  const [size, setSize] = useState({ width: 0, height: 0 });

  return (
    <div
      ref={(node) => {
        if (!node) return;

        // Setup
        const observer = new ResizeObserver(([entry]) => {
          setSize({
            width: entry.contentRect.width,
            height: entry.contentRect.height,
          });
        });
        observer.observe(node);

        // React 19: return cleanup function!
        // Tự động gọi khi element unmount hoặc ref thay đổi
        return () => observer.disconnect();
        // Trước React 19: phải check node === null trong callback
      }}
    >
      Content: {size.width}x{size.height}
    </div>
  );
}
```

---

### 3.3 Document Metadata *(React 19)*

#### Lý thuyết & Bản chất
React 19 hỗ trợ render `<title>`, `<meta>`, `<link>` **trực tiếp trong component** — React tự động hoist chúng lên `<head>`. Không cần `next/head` hay `react-helmet` nữa (trong Next.js vẫn dùng metadata API).

---

### Ví dụ 1 — Cơ bản
```jsx
// React 19: document metadata trong component
function ProductPage({ product }) {
  return (
    <article>
      {/* React tự hoist các thẻ này lên <head> */}
      <title>{product.name} — My Store</title>
      <meta name="description" content={product.description} />
      <meta property="og:title" content={product.name} />
      <meta property="og:image" content={product.image} />
      <link rel="canonical" href={`https://store.com/products/${product.slug}`} />

      {/* Content bình thường */}
      <h1>{product.name}</h1>
      <p>{product.description}</p>
    </article>
  );
}
// Trước React 19: phải dùng next/head, react-helmet, hoặc useEffect + document.title
```

---

### 3.4 Asset Loading (Suspense cho CSS, Fonts, Scripts) *(React 19)*

```jsx
import { preload, preinit, prefetchDNS, preconnect } from 'react-dom';

function App() {
  // Preload resources khi component render
  preload('/fonts/inter.woff2', { as: 'font', crossOrigin: 'anonymous' });
  preload('/styles/critical.css', { as: 'style' });
  preinit('/scripts/analytics.js', { as: 'script' }); // load và execute

  prefetchDNS('https://api.example.com'); // DNS prefetch
  preconnect('https://api.example.com');  // preconnect

  return <div>App</div>;
}

// Stylesheet với Suspense: component suspend cho đến khi CSS load xong
function StyledComponent() {
  return (
    <div>
      <link rel="stylesheet" href="/styles/component.css" precedence="default" />
      {/* Component này không render cho đến khi CSS load xong */}
      <div className="styled-content">Content</div>
    </div>
  );
}
```

---

### 3.5 React Compiler *(React 19 / experimental)*

#### Lý thuyết & Bản chất
React Compiler (trước đây: React Forget) là **build-time compiler** tự động thêm memoization — không cần `useMemo`, `useCallback`, `React.memo` thủ công nữa.

```jsx
// TRƯỚC: phải viết thủ công
const ExpensiveList = React.memo(function({ items, onSelect }) {
  const processed = useMemo(() => processItems(items), [items]);
  const handleSelect = useCallback((id) => onSelect(id), [onSelect]);
  return <List items={processed} onSelect={handleSelect} />;
});

// VỚI REACT COMPILER: compiler tự phân tích và thêm memoization
// Viết code tự nhiên, compiler lo optimization
function ExpensiveList({ items, onSelect }) {
  const processed = processItems(items); // compiler tự memo
  const handleSelect = (id) => onSelect(id); // compiler tự memo
  return <List items={processed} onSelect={handleSelect} />;
}

// Setup trong Next.js (experimental):
// next.config.js
module.exports = {
  experimental: {
    reactCompiler: true,
  },
};
```

---

### 3.6 Improved Error Messages *(React 19)*

```jsx
// React 19 cải thiện hydration error messages
// TRƯỚC: "Hydration failed because the server-rendered HTML didn't match the client."
// SAU: Chỉ rõ element nào, diff cụ thể là gì

// React 19 cũng cải thiện Error Boundary
// Trước: lỗi trong ErrorBoundary xảy ra 2 lần trong development
// Sau: chỉ xảy ra 1 lần, dễ debug hơn
```

---

### 3.7 Context như Provider *(React 19)*

```jsx
// TRƯỚC React 19
const ThemeContext = createContext('light');
function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Page />
    </ThemeContext.Provider>
  );
}

// REACT 19: Context object có thể dùng trực tiếp như Provider
function App() {
  return (
    <ThemeContext value="dark"> {/* Không cần .Provider */}
      <Page />
    </ThemeContext>
  );
}
```

---

---

## PHẦN 4 — HOOKS COMPARISON TABLE (Toàn bộ)

| Hook | Introduced | Bản chất | Khi dùng |
|------|-----------|----------|----------|
| `useState` | 16.8 | Local state | Bất kỳ component state nào |
| `useReducer` | 16.8 | State machine | State phức tạp, nhiều actions |
| `useEffect` | 16.8 | Side effect sau paint | Fetch, subscribe, timer |
| `useLayoutEffect` | 16.8 | Side effect trước paint | Measure DOM, prevent flicker |
| `useContext` | 16.8 | Đọc context | Shared data không cần drill |
| `useRef` | 16.8 | Mutable, no re-render | DOM access, timers, prev value |
| `useMemo` | 16.8 | Cache value | Expensive computation |
| `useCallback` | 16.8 | Cache function | Prop cho memo'd child |
| `useImperativeHandle` | 16.8 | Expose API qua ref | Library components, media players |
| `useDebugValue` | 16.8 | DevTools label | Custom hooks |
| `useTransition` | 18 | Non-urgent state | Filter list, navigation |
| `useDeferredValue` | 18 | Defer re-render với value | Search, filter (prop từ parent) |
| `useId` | 18 | Stable unique ID | Form labels, aria attributes |
| `useSyncExternalStore` | 18 | External store | Redux, Zustand, browser APIs |
| `useInsertionEffect` | 18 | Before DOM mutations | CSS-in-JS libraries only |
| `use` | **19** | Unwrap Promise/Context | Async data, flexible context |
| `useFormStatus` | **19** | Form pending state | Submit buttons |
| `useActionState` | **19** | Action state management | Forms với server actions |
| `useOptimistic` | **19** | Optimistic UI | Mutations, likes, cart |

---

## PHẦN 5 — INTERVIEW Q&A REACT 19

### Q: "React 19 có gì mới nhất?"

**Trả lời:**
React 19 tập trung vào **Actions** và **Server/Client integration**:

1. **Actions**: Async functions dùng với form `action` prop — tự động quản lý pending, error states
2. **`use()` hook**: Universal hook có thể unwrap Promise và Context, có thể dùng trong conditionals
3. **`useActionState`**: Quản lý state của form actions (đổi tên từ `useFormState`)
4. **`useFormStatus`**: Đọc pending state của form cha gần nhất
5. **`useOptimistic`**: Built-in optimistic updates
6. **`ref` như prop**: Không cần `forwardRef` nữa
7. **Document metadata**: `<title>`, `<meta>` trực tiếp trong JSX
8. **React Compiler**: Tự động memoization (experimental)
9. **Context trực tiếp như Provider**: `<ThemeContext value="">` thay vì `<ThemeContext.Provider>`

---

### Q: "Giải thích sự khác biệt giữa `useTransition` và `useDeferredValue`"

```
useTransition:
- Bạn CONTROL state setter
- Wrap setter trong startTransition()
- Dùng khi: bạn own cả state và việc update

useDeferredValue:
- Bạn NHẬN value từ bên ngoài (props)
- Wrap value trong useDeferredValue()
- Dùng khi: parent truyền value xuống, bạn muốn defer expensive re-render

VD:
// useTransition: bạn own state
const [query, setQuery] = useState('');
startTransition(() => setQuery(newValue));

// useDeferredValue: prop từ parent
function SearchResults({ query }) { // query từ parent
  const deferredQuery = useDeferredValue(query);
  const results = useMemo(() => filter(data, deferredQuery), [deferredQuery]);
}
```

---

### Q: "`use()` hook khác `useContext()` thế nào?"

```jsx
// useContext: chỉ đọc Context, KHÔNG thể dùng trong conditions
function Component() {
  const theme = useContext(ThemeContext); // luôn gọi ở top level
}

// use(): đọc cả Context VÀ Promise, CÓ THỂ dùng trong conditions
function Component({ showTheme }) {
  if (showTheme) {
    const theme = use(ThemeContext); // ✅ trong condition — chỉ với use()!
  }

  const user = use(userPromise); // ✅ unwrap Promise — suspend cho đến khi resolve
}

// use() với Promise: component suspend → Suspense boundary hiện fallback
function UserCard({ userPromise }) {
  const user = use(userPromise); // suspend nếu chưa resolve
  return <div>{user.name}</div>;
}
// Wrap trong Suspense
<Suspense fallback={<Spinner />}>
  <UserCard userPromise={fetchUser(id)} />
</Suspense>
```

---

### Q: "Khi nào dùng `useLayoutEffect` thay `useEffect`?"

```
useEffect: async, sau browser paint
useLayoutEffect: sync, sau DOM update, TRƯỚC browser paint

Dùng useLayoutEffect khi:
1. Cần đọc DOM layout (getBoundingClientRect, offsetHeight...)
   và update state dựa trên đó
   → nếu dùng useEffect: user thấy flicker (render → paint → update → paint)
   → useLayoutEffect: update trước paint → không flicker

2. Cần animate, tooltip positioning, scroll restoration

3. Cần synchronously fire một DOM update

VD:
useLayoutEffect(() => {
  // Đo kích thước, update state
  const height = ref.current.getBoundingClientRect().height;
  setHeight(height); // cập nhật trước khi user thấy
});

Lưu ý: useLayoutEffect block paint → tránh expensive work trong đây
```
