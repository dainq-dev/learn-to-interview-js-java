# Interview Q&A — Câu hỏi hay gặp + Trả lời chuẩn theo cấp độ

---

## PHẦN 1 — JAVASCRIPT / TYPESCRIPT

### Q1: "Giải thích Event Loop và cho ví dụ"

**Trả lời chuẩn:**
Event Loop là cơ chế cho phép JavaScript single-threaded xử lý bất đồng bộ. JS engine có Call Stack để thực thi code, Web APIs (setTimeout, fetch...) chạy song song ngoài engine. Khi xong, callback được đẩy vào Callback Queue. Event Loop liên tục kiểm tra: khi Call Stack rỗng → lấy callback từ Microtask Queue trước (Promise .then), rồi mới Macrotask Queue (setTimeout).

```js
console.log('A');
setTimeout(() => console.log('B'), 0);
Promise.resolve().then(() => console.log('C'));
console.log('D');
// Output: A → D → C → B
// Giải thích:
// A, D: synchronous → chạy ngay
// C: microtask → chạy sau khi call stack rỗng, TRƯỚC macrotask
// B: macrotask → chạy sau cùng
```

---

### Q2: "`==` vs `===`"

**Trả lời:**
- `===` (strict equality): so sánh cả giá trị và type, không ép kiểu
- `==` (loose equality): ép kiểu trước khi so sánh (type coercion)

```js
0 == false     // true  (false được coerce thành 0)
0 === false    // false (number !== boolean)
'' == false    // true
null == undefined // true (đặc biệt)
null === undefined // false

// Rule: Luôn dùng === trong production code
// Trường hợp dùng ==: check cả null và undefined cùng lúc
if (value == null) { ... } // bắt cả null và undefined
```

---

### Q3: "Sự khác biệt giữa `var`, `let`, `const`"

| | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function | Block | Block |
| Hoisting | Có (undefined) | Có (TDZ) | Có (TDZ) |
| Re-declare | Được | Không | Không |
| Re-assign | Được | Được | Không |

**TDZ (Temporal Dead Zone):** Vùng từ đầu block đến chỗ khai báo. Truy cập biến trong TDZ → ReferenceError (khác `var` trả `undefined`).

```js
// var hoisting
console.log(x); // undefined (không phải ReferenceError)
var x = 5;

// let TDZ
console.log(y); // ReferenceError: Cannot access 'y' before initialization
let y = 5;

// Const với object: reference không đổi, content có thể thay đổi
const obj = { x: 1 };
obj.x = 2; // OK: thay đổi content
obj = {};  // Error: không thể reassign reference
```

---

### Q4: "Deep clone vs Shallow clone"

```js
// Shallow: copy references, không copy nested objects
const shallow = { ...original };
const shallow2 = Object.assign({}, original);

// Deep: copy hoàn toàn, mỗi cấp độ nested
const deep = structuredClone(original); // ES2022, handles Date, Map, Set, RegExp

// JSON method (hạn chế: không handle undefined, function, circular refs, Date)
const deep2 = JSON.parse(JSON.stringify(original));

// Khi nào dùng:
// Shallow: state update trong React (immutable top-level đủ nếu không dùng nested)
// Deep: cần truly isolated copy (undo/redo, snapshot...)
```

---

### Q5: "TypeScript `any` vs `unknown` vs `never`"

```ts
// any: tắt type checking hoàn toàn — tránh dùng
let a: any = 'hello';
a.toFixed(); // no error — DANGEROUS

// unknown: type-safe any — phải narrow trước khi dùng
let u: unknown = 'hello';
u.toUpperCase(); // Error!
if (typeof u === 'string') u.toUpperCase(); // OK

// never: type không bao giờ có giá trị
// - Return type của function throw hoặc infinite loop
// - Kết quả của union sau khi đã narrow hết cases
function fail(msg: string): never { throw new Error(msg); }
function exhaustiveCheck(x: never): never { throw new Error('Should not reach'); }
```

---

## PHẦN 2 — REACT

### Q6: "Giải thích React rendering lifecycle"

**Render Phase (có thể interrupt):**
1. Gọi function component / render method
2. React chạy diffing với VDOM trước đó
3. Tạo list of changes (effects)

**Commit Phase (không interrupt):**
1. `useLayoutEffect` cleanup (sync)
2. Apply DOM changes
3. `useLayoutEffect` run (sync, trước paint)
4. Browser paint
5. `useEffect` cleanup
6. `useEffect` run

```
Mount:  Render → Commit → LayoutEffect → Paint → Effect
Update: Render → Commit → LayoutEffect cleanup → LayoutEffect → Paint → Effect cleanup → Effect
Unmount:                  LayoutEffect cleanup → Effect cleanup
```

---

### Q7: "Khi nào component re-render?"

Component re-render khi:
1. **State thay đổi** (`setState` / `dispatch`)
2. **Props thay đổi** (parent re-render → child re-render by default)
3. **Context value thay đổi** (mọi consumer re-render)
4. **Parent re-render** (ngay cả khi props không đổi, trừ khi dùng `React.memo`)

**Cách prevent:**
```jsx
// React.memo: skip re-render nếu props không đổi (shallow compare)
const Child = React.memo(function Child({ name, onClick }) {
  return <button onClick={onClick}>{name}</button>;
});

// Lưu ý: React.memo vô nghĩa nếu:
// 1. onClick là inline arrow function (mới mỗi render) → dùng useCallback
// 2. style/object prop là inline object → dùng useMemo
// 3. Children prop là JSX → khó memo
```

---

### Q8: "Controlled vs Uncontrolled component"

```jsx
// Controlled: state trong React
function ControlledInput() {
  const [value, setValue] = React.useState('');
  return (
    <input
      value={value}      // React kiểm soát giá trị
      onChange={e => setValue(e.target.value)}
    />
  );
}
// Ưu điểm: realtime validation, conditional disable, format input
// Nhược: nhiều re-render hơn với forms lớn

// Uncontrolled: state trong DOM
function UncontrolledInput() {
  const ref = React.useRef(null);
  function handleSubmit() {
    console.log(ref.current.value); // đọc khi cần
  }
  return <input ref={ref} defaultValue="initial" />;
}
// Ưu: ít code, ít re-render, tương thích thư viện non-React
// Nhược: khó validate realtime, khó reset

// React Hook Form: kết hợp cả hai — uncontrolled by default, controlled khi cần
```

---

### Q9: "Tại sao không được gọi hooks trong conditions/loops?"

```jsx
// BAD:
function Component({ isLoggedIn }) {
  if (isLoggedIn) {
    const [user, setUser] = useState(null); // RULE VIOLATION!
  }
}

// Lý do: React track hooks theo THỨ TỰ (index trong fiber)
// Mỗi render phải gọi hooks theo cùng thứ tự

// Render 1: useState(0), useState(''), useEffect → hooks[0], hooks[1], hooks[2]
// Render 2 (isLoggedIn = true): useState(null) [extra!] → thứ tự sai → bug!

// FIX: gọi hook ngoài condition, dùng condition bên trong
function Component({ isLoggedIn }) {
  const [user, setUser] = useState(null);
  useEffect(() => {
    if (isLoggedIn) fetchUser().then(setUser);
  }, [isLoggedIn]);
}
```

---

### Q10: "Giải thích useEffect dependency array"

```jsx
// [] — run once after mount
useEffect(() => { fetchData(); }, []);

// [id] — re-run when id changes
useEffect(() => { fetchUser(id); }, [id]);

// Không có array — run after every render (HIẾM KHI muốn điều này)
useEffect(() => { document.title = count; });

// COMMON MISTAKE: function trong deps
const fetchFn = () => fetch('/api/data'); // mới mỗi render!
useEffect(() => { fetchFn(); }, [fetchFn]); // infinite loop!

// FIX:
const fetchFn = useCallback(() => fetch('/api/data'), []); // stable reference
useEffect(() => { fetchFn(); }, [fetchFn]); // OK

// eslint-plugin-react-hooks: auto detect missing/extra deps
// Luôn nghe warning của exhaustive-deps rule!
```

---

## PHẦN 3 — NEXT.JS

### Q11: "SSR vs SSG vs ISR — khi nào dùng gì?"

```
SSG (Static Site Generation):
✅ Blog posts, documentation, marketing pages
✅ SEO important + data không thay đổi thường
✅ Cực kỳ nhanh (served from CDN)
❌ Data out-of-date cho đến khi rebuild

ISR (Incremental Static Regeneration):
✅ E-commerce product pages, news articles
✅ SEO important + data thay đổi định kỳ
✅ Nhanh như SSG, data tươi hơn
❌ Một số user có thể thấy stale data (stale-while-revalidate)

SSR (Server-Side Rendering):
✅ Pages cần real-time data (stock price, live scores)
✅ Pages cần user-specific data (dashboard, profile)
✅ Pages cần request-time info (cookies, geolocation)
❌ Chậm hơn SSG/ISR, tốn server resources

CSR (Client-Side Rendering):
✅ Highly interactive (admin panels, complex forms)
✅ Behind auth wall (không cần SEO)
✅ Real-time features (chat, live updates)
❌ Chậm initial load, SEO kém nếu không có SSR wrapper
```

---

### Q12: "Server Components vs Client Components"

```tsx
// Server Components (default trong App Router):
// - Render trên server
// - KHÔNG thể dùng useState, useEffect, event handlers, browser APIs
// - CÓ THỂ: async/await, truy cập DB, đọc env vars, headers/cookies
// - Không gửi JS xuống client → bundle nhỏ hơn

async function ProductList() {
  const products = await db.products.findAll(); // trực tiếp DB!
  return <ul>{products.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}

// Client Components ('use client'):
// - Render trên cả server (initial) VÀ client (hydration + updates)
// - CÓ THỂ dùng tất cả React features
// - Gửi JS xuống client

'use client';
function AddToCartButton({ productId }) {
  const [added, setAdded] = useState(false);
  return (
    <button onClick={() => { addToCart(productId); setAdded(true); }}>
      {added ? 'Added!' : 'Add to Cart'}
    </button>
  );
}

// Pattern: Server Component chứa Client Component
async function ProductPage({ id }) {
  const product = await getProduct(id); // Server
  return (
    <div>
      <ProductInfo product={product} /> {/* Server Component */}
      <AddToCartButton productId={id} /> {/* Client Component */}
    </div>
  );
}
```

---

## PHẦN 4 — BACKEND

### Q13: "@Transactional pitfalls"

```java
@Service
public class UserService {

    // PITFALL 1: Self-invocation bypasses proxy
    public void processUser(Long id) {
        this.updateEmail(id, "new@email.com"); // self-call → @Transactional bị bỏ qua!
    }

    @Transactional
    public void updateEmail(Long id, String email) { ... }

    // FIX: inject self hoặc tách service
    @Autowired private UserService self; // ApplicationContext sẽ inject proxy
    public void processUser(Long id) {
        self.updateEmail(id, "new@email.com"); // gọi qua proxy → @Transactional hoạt động
    }

    // PITFALL 2: Checked exception không rollback by default
    @Transactional
    public void risky() throws IOException {
        repo.save(entity);
        throw new IOException("fail"); // KHÔNG rollback!
    }

    @Transactional(rollbackFor = Exception.class) // FIX
    public void risky() throws IOException { ... }

    // PITFALL 3: @Transactional trên private method
    @Transactional // VÔ NGHĨA — Spring không thể proxy private methods
    private void privateMethod() { ... }
}
```

---

### Q14: "Giải thích N+1 và cách fix"

```
N+1 Problem:
Query 1: SELECT * FROM orders LIMIT 10      → 10 orders
Query 2: SELECT * FROM users WHERE id = 1  → user của order 1
Query 3: SELECT * FROM users WHERE id = 2  → user của order 2
...
Query 11: SELECT * FROM users WHERE id = 10
Tổng: 11 queries thay vì 1

FIX 1 - JOIN FETCH:
@Query("SELECT o FROM Order o LEFT JOIN FETCH o.user WHERE o.status = :status")
List<Order> findByStatus(OrderStatus status);
// Kết quả: 1 query với JOIN

FIX 2 - @EntityGraph:
@EntityGraph(attributePaths = {"user", "items"})
List<Order> findAll();

FIX 3 - @BatchSize (cho collection):
@BatchSize(size = 25)
@OneToMany
List<OrderItem> items;
// Thay vì N queries riêng lẻ: SELECT * FROM items WHERE order_id IN (1,2,...,25)

FIX 4 - DTO Projection (tốt nhất cho performance):
@Query("SELECT new com.app.dto.OrderSummary(o.id, u.name, o.total) FROM Order o JOIN o.user u")
List<OrderSummary> findSummaries();
```

---

### Q15: "JWT vs Session-based auth"

```
JWT (JSON Web Token):
✅ Stateless: server không cần lưu session → dễ scale horizontally
✅ Có thể decode mà không cần DB (verify signature bằng secret)
✅ Phù hợp microservices, mobile apps
❌ Không thể revoke ngay (phải đợi hết hạn hoặc maintain blacklist)
❌ Payload lớn → mỗi request tốn bandwidth
❌ Secret bị lộ → toàn bộ tokens invalid

Session-based:
✅ Có thể revoke ngay (xóa session khỏi store)
✅ Payload nhỏ (chỉ session ID trong cookie)
✅ Phù hợp khi cần strict logout
❌ Stateful: cần session store (Redis) khi scale
❌ Sticky sessions nếu không dùng centralized store

Best practice kết hợp:
- Access token (JWT, ngắn hạn 15m): stateless, nhanh
- Refresh token (stored in DB + HttpOnly cookie, dài hạn 7d): có thể revoke
```

---

## PHẦN 5 — CODING CHALLENGES

### Challenge 1 — Cơ bản: Implement debounce
```js
function debounce(fn, delay) {
  let timerId;
  return function(...args) {
    clearTimeout(timerId);
    timerId = setTimeout(() => fn.apply(this, args), delay);
  };
}

// Test
const search = debounce((query) => console.log('Searching:', query), 300);
search('a');   // cancelled
search('ab');  // cancelled
search('abc'); // executed after 300ms: "Searching: abc"
```

---

### Challenge 2 — Trung cấp: Flatten nested array + Deep equal
```js
// Flatten (không dùng .flat())
function flatten(arr, depth = Infinity) {
  return depth > 0
    ? arr.reduce((acc, val) =>
        Array.isArray(val)
          ? acc.concat(flatten(val, depth - 1))
          : [...acc, val]
      , [])
    : arr.slice();
}

flatten([1, [2, [3, [4]]]]);        // [1, 2, 3, 4]
flatten([1, [2, [3, [4]]]], 1);     // [1, 2, [3, [4]]]

// Deep equal
function deepEqual(a, b) {
  if (a === b) return true;
  if (typeof a !== typeof b) return false;
  if (typeof a !== 'object' || a === null || b === null) return false;
  if (Array.isArray(a) !== Array.isArray(b)) return false;

  const keysA = Object.keys(a);
  const keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;

  return keysA.every(key => deepEqual(a[key], b[key]));
}
```

---

### Challenge 3 — Nâng cao: Implement Promise.all + Throttle
```js
// Promise.all
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (!promises.length) return resolve([]);
    const results = new Array(promises.length);
    let remaining = promises.length;

    promises.forEach((p, i) => {
      Promise.resolve(p).then(val => {
        results[i] = val;
        if (--remaining === 0) resolve(results);
      }).catch(reject);
    });
  });
}

// Throttle: limit execution rate
// Khác debounce: debounce delay sau last call, throttle execute mỗi N ms
function throttle(fn, limit) {
  let inThrottle = false;
  let lastArgs;

  return function(...args) {
    lastArgs = args;
    if (!inThrottle) {
      fn.apply(this, args);
      inThrottle = true;
      setTimeout(() => {
        inThrottle = false;
        if (lastArgs !== args) fn.apply(this, lastArgs); // call với args mới nhất
      }, limit);
    }
  };
}

// React: custom useThrottle hook
function useThrottle(value, limit) {
  const [throttled, setThrottled] = React.useState(value);
  const lastUpdated = React.useRef(null);

  React.useEffect(() => {
    const now = Date.now();
    const timeSinceLast = lastUpdated.current ? now - lastUpdated.current : Infinity;

    if (timeSinceLast >= limit) {
      lastUpdated.current = now;
      setThrottled(value);
    } else {
      const timer = setTimeout(() => {
        lastUpdated.current = Date.now();
        setThrottled(value);
      }, limit - timeSinceLast);
      return () => clearTimeout(timer);
    }
  }, [value, limit]);

  return throttled;
}
```

---

## PHẦN 6 — BEHAVIORAL QUESTIONS

### Q: "Kể về bug production khó nhất bạn từng fix"

**Framework STAR:**
```
Situation: Mô tả context (app gì, scale thế nào)
Task: Vấn đề cụ thể là gì
Action: Bạn làm gì để debug và fix
Result: Kết quả, bài học rút ra
```

**Ví dụ model answer:**
```
"Chúng tôi có một React app với danh sách lớn 10,000 items.
Users report page bị freeze khi scroll nhanh.

Tôi dùng React DevTools Profiler để xác định component nào gây issue.
Phát hiện ListItem re-render toàn bộ list mỗi khi scroll
vì parent state thay đổi (scroll position) khiến mọi item re-render.

Fix: Virtualize list với react-window + React.memo cho ListItem.
Kết quả: từ 2s freeze xuống còn smooth 60fps.
Bài học: profile trước khi optimize, không assume nguyên nhân."
```

---

### Q: "Làm thế nào design một feature phức tạp?"

```
1. Clarify requirements
   - User story rõ ràng
   - Edge cases
   - Non-functional: performance, scale, security

2. API contract first
   - Thiết kế API endpoints trước
   - Frontend và backend có thể làm song song

3. Frontend: mock API với MSW (Mock Service Worker)
   → Develop UI mà không cần backend xong

4. Backend: implement theo contract
   → Unit test business logic
   → Integration test API

5. Integration & E2E test
   → Cypress / Playwright cho critical paths
```

---

## CHECKLIST CUỐI — Self-Assessment

### Frontend (phải biết chắc)
- [ ] Giải thích Event Loop với output code question
- [ ] Closure: viết module pattern, memoize
- [ ] `this` binding: 4 rules, arrow vs regular
- [ ] React rendering: khi nào re-render, cách prevent
- [ ] Hooks: useState pitfalls, useEffect cleanup, useMemo vs useCallback
- [ ] Next.js: 4 strategies, Server vs Client Components, caching
- [ ] State management: Redux flow, Zustand selector
- [ ] JWT: flow, secure storage, refresh token
- [ ] XSS/CSRF: nguyên nhân, cách prevent

### Backend (biết đủ để nói)
- [ ] IoC/DI: constructor injection, tại sao
- [ ] @Transactional: rollback rules, pitfalls
- [ ] N+1: nhận ra, fix bằng JOIN FETCH / EntityGraph
- [ ] JWT filter chain
- [ ] Redis caching: @Cacheable, @CacheEvict

### Coding
- [ ] Debounce / Throttle
- [ ] Promise.all / Promise.race implementation
- [ ] Deep clone methods
- [ ] Custom hooks: useDebounce, useLocalStorage
