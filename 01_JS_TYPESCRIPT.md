# JavaScript ES6+ & TypeScript — Lý thuyết sâu + Ví dụ 3 cấp độ

---

## 1. CLOSURE

### Lý thuyết & Bản chất
Closure là **hàm có khả năng "nhớ" (capture) biến từ scope bên ngoài** ngay cả sau khi scope đó đã kết thúc thực thi.

**Bản chất:** Khi một hàm được tạo ra trong JavaScript, nó không chỉ lưu code của hàm đó mà còn lưu một tham chiếu đến **Lexical Environment** (môi trường từ vựng) — tức là tất cả các biến có thể truy cập tại thời điểm hàm được khai báo. Tham chiếu này tồn tại miễn là hàm còn tồn tại, dù scope cha đã kết thúc.

**Cơ chế:**
```
Lexical Environment = { biến cục bộ } + tham chiếu đến Lexical Environment cha
```
Khi JS engine cần resolve một biến, nó đi theo chuỗi này (Scope Chain) từ trong ra ngoài cho đến `global`.

---

### Ví dụ 1 — Cơ bản: Counter đơn giản
```js
function makeCounter() {
  let count = 0; // biến trong scope của makeCounter

  return function increment() {
    count++; // increment "nhớ" count dù makeCounter đã chạy xong
    return count;
  };
}

const counter = makeCounter();
console.log(counter()); // 1
console.log(counter()); // 2
console.log(counter()); // 3

// Tạo counter mới → lexical environment MỚI → count riêng biệt
const counter2 = makeCounter();
console.log(counter2()); // 1 (không bị ảnh hưởng bởi counter)
```
**Giải thích:** `makeCounter` trả về hàm `increment`. Hàm này giữ tham chiếu đến `count` trong lexical environment của `makeCounter`. Mỗi lần gọi `counter()`, cùng một `count` được tăng lên.

---

### Ví dụ 2 — Trung cấp: Module Pattern + Private State
```js
// Pattern này mô phỏng private/public trước khi có class
function createBankAccount(initialBalance) {
  let balance = initialBalance; // "private" — không thể access trực tiếp từ ngoài
  const transactions = [];       // "private"

  return {
    // "public" methods — closure giúp chúng access private state
    deposit(amount) {
      if (amount <= 0) throw new Error('Amount must be positive');
      balance += amount;
      transactions.push({ type: 'deposit', amount, date: new Date() });
      return this;
    },
    withdraw(amount) {
      if (amount > balance) throw new Error('Insufficient funds');
      balance -= amount;
      transactions.push({ type: 'withdraw', amount, date: new Date() });
      return this;
    },
    getBalance() { return balance; },
    getHistory() { return [...transactions]; }, // trả bản sao để tránh mutation
  };
}

const account = createBankAccount(1000);
account.deposit(500).withdraw(200); // method chaining
console.log(account.getBalance()); // 1300
console.log(account.balance);      // undefined — không thể access trực tiếp!
```
**Giải thích:** `balance` và `transactions` là "private" vì chúng chỉ tồn tại trong scope của `createBankAccount`. Các method được trả về có closure trên chúng, nhưng code bên ngoài thì không.

---

### Ví dụ 3 — Nâng cao: Memoization + Closure pitfall trong vòng lặp
```js
// --- Memoization ---
function memoize(fn) {
  const cache = new Map(); // closure giữ cache này tồn tại suốt vòng đời của memoized fn

  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      console.log('Cache hit:', key);
      return cache.get(key);
    }
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

const expensiveAdd = memoize((a, b) => {
  // Giả sử tính toán nặng
  return a + b;
});

expensiveAdd(1, 2); // Tính toán lần đầu, lưu cache
expensiveAdd(1, 2); // Cache hit → trả ngay, không tính lại


// --- Classic Closure Pitfall trong loop ---
// BUG: tất cả handler đều in ra 3 vì var là function-scoped, không block-scoped
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log('var:', i), 0); // In ra: 3, 3, 3
}

// FIX 1: Dùng let (block-scoped → mỗi iteration có i riêng)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log('let:', i), 0); // In ra: 0, 1, 2
}

// FIX 2: IIFE (Immediately Invoked Function Expression) — cách cũ trước khi có let
for (var i = 0; i < 3; i++) {
  ((j) => {
    setTimeout(() => console.log('IIFE:', j), 0); // In ra: 0, 1, 2
  })(i); // truyền i như argument → tạo scope mới với giá trị hiện tại
}
```
**Giải thích Pitfall:** Với `var`, tất cả 3 callback chia sẻ cùng 1 biến `i` (function-scoped). Khi setTimeout chạy, vòng lặp đã xong, `i = 3`. Với `let`, mỗi iteration tạo một block scope mới với một `i` riêng.

---

## 2. EVENT LOOP & ASYNC

### Lý thuyết & Bản chất
JavaScript là **single-threaded** — chỉ có 1 luồng thực thi. Event Loop là cơ chế cho phép JS xử lý **bất đồng bộ (non-blocking)** mà không cần đa luồng.

**Các thành phần:**
- **Call Stack**: Nơi function được thực thi. LIFO (Last In, First Out). Chỉ có 1.
- **Heap**: Bộ nhớ lưu object.
- **Web APIs** (browser) / **C++ APIs** (Node.js): Xử lý async operations (setTimeout, fetch, DOM events...). **Chạy ngoài JS engine.**
- **Callback Queue (Task Queue / Macrotask Queue)**: Hàng đợi callback từ Web APIs.
- **Microtask Queue**: Hàng đợi Promise `.then`, `queueMicrotask`, `MutationObserver`. **Ưu tiên cao hơn Macrotask.**

**Vòng lặp Event Loop:**
```
1. Thực thi toàn bộ synchronous code (làm trống Call Stack)
2. Chạy HẾT tất cả microtasks (bao gồm cả microtask được thêm bởi microtask khác)
3. Render (nếu cần) — chỉ ở browser
4. Lấy 1 macrotask từ Callback Queue → chạy
5. Quay lại bước 2
```

---

### Ví dụ 1 — Cơ bản: Phân tích thứ tự execution
```js
console.log('1 - sync');

setTimeout(() => console.log('2 - macrotask'), 0);

Promise.resolve()
  .then(() => console.log('3 - microtask 1'))
  .then(() => console.log('4 - microtask 2')); // được thêm bởi microtask 1

console.log('5 - sync');

// Output:
// 1 - sync       ← synchronous
// 5 - sync       ← synchronous
// 3 - microtask 1 ← microtask queue (chạy trước macrotask)
// 4 - microtask 2 ← microtask được thêm bởi microtask trước → vẫn trong cùng "turn"
// 2 - macrotask  ← macrotask queue
```

---

### Ví dụ 2 — Trung cấp: async/await bản chất là gì?
```js
// async/await chỉ là "syntactic sugar" trên Promise
// await "trả" lại control cho Event Loop, tiếp tục sau khi Promise resolve

async function fetchData() {
  console.log('A - start of async fn'); // sync

  const data = await Promise.resolve('hello'); // await = .then()
  // Mọi thứ SAU await chạy như microtask
  console.log('B - after await:', data);
}

console.log('1 - before');
fetchData();
console.log('2 - after call'); // chạy TRƯỚC 'B' vì fetchData đã "nhường" control

// Output:
// 1 - before
// A - start of async fn
// 2 - after call
// B - after await: hello

// Giải thích: khi gặp `await`, hàm async tạm dừng, trả control về caller.
// Caller tiếp tục chạy synchronous code ('2 - after call').
// Sau đó Promise resolve, callback của .then được đẩy vào microtask queue.
```

---

### Ví dụ 3 — Nâng cao: Starving macrotasks + Long task blocking
```js
// Vấn đề: Recursive microtask starves macrotasks
function recursiveMicrotask(count) {
  if (count <= 0) return;
  Promise.resolve().then(() => {
    console.log('microtask', count);
    recursiveMicrotask(count - 1); // thêm microtask MỚI → macrotask không bao giờ chạy!
  });
}

setTimeout(() => console.log('macrotask — sẽ không bao giờ in nếu microtask vô hạn'), 0);
recursiveMicrotask(3);
// Output: microtask 3, microtask 2, microtask 1, macrotask
// Với count lớn vô hạn → macrotask bị "starve"

// --- Long task blocking UI ---
// BAD: block call stack 1 giây → UI freeze
function blockingTask() {
  const start = Date.now();
  while (Date.now() - start < 1000) {} // blocking!
  console.log('done');
}

// GOOD: chia nhỏ task với scheduler
function yieldToMain() {
  return new Promise(resolve => setTimeout(resolve, 0));
}

async function nonBlockingTask(items) {
  for (let i = 0; i < items.length; i++) {
    processItem(items[i]);
    // Sau mỗi N items, yield → browser có thể render, handle input
    if (i % 50 === 0) await yieldToMain();
  }
}
// Đây là pattern React Fiber dùng để không block UI khi render tree lớn
```

---

## 3. PROTOTYPE & `this`

### Lý thuyết & Bản chất
**Prototype:** Mỗi object trong JS có một thuộc tính ẩn `[[Prototype]]` trỏ đến một object khác (prototype của nó). Khi truy cập property không tồn tại trên object, JS tìm lên chuỗi prototype (prototype chain) cho đến `null`.

**`this`:** Giá trị `this` **không được xác định khi hàm được khai báo** mà khi hàm được **gọi**. Có 4 quy tắc:
1. **Default binding**: `fn()` → `this = window` (strict mode: `undefined`)
2. **Implicit binding**: `obj.fn()` → `this = obj`
3. **Explicit binding**: `fn.call(obj)`, `fn.apply(obj)`, `fn.bind(obj)` → `this = obj`
4. **New binding**: `new Fn()` → `this = object mới được tạo`

**Arrow function**: KHÔNG có `this` riêng. Kế thừa `this` từ **lexical scope** (scope bao quanh lúc khai báo).

---

### Ví dụ 1 — Cơ bản: `this` trong các context
```js
const user = {
  name: 'Alice',

  // Regular function: this = object gọi hàm (implicit binding)
  greet() {
    return `Hello, ${this.name}`;
  },

  // Arrow function: this = this tại thời điểm khai báo (lexical) = global/undefined
  greetArrow: () => {
    return `Hello, ${this?.name}`; // this.name là undefined!
  },

  // Gotcha: callback mất context
  greetLater() {
    setTimeout(function() {
      console.log(this.name); // undefined! setTimeout gọi callback với this = window
    }, 100);

    setTimeout(() => {
      console.log(this.name); // 'Alice' — arrow fn giữ this từ greetLater
    }, 100);
  }
};

console.log(user.greet());      // 'Hello, Alice'
console.log(user.greetArrow()); // 'Hello, undefined'
```

---

### Ví dụ 2 — Trung cấp: Prototype chain & Inheritance
```js
function Animal(name) {
  this.name = name; // own property
}

// Thêm method vào prototype → tất cả instances dùng chung → tiết kiệm bộ nhớ
Animal.prototype.speak = function() {
  return `${this.name} makes a sound.`;
};

function Dog(name, breed) {
  Animal.call(this, name); // explicit binding để "steal" Animal's constructor
  this.breed = breed;
}

// Thiết lập prototype chain: Dog.prototype → Animal.prototype
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog; // fix bị ghi đè

Dog.prototype.speak = function() {
  return `${this.name} barks.`; // override
};

const d = new Dog('Rex', 'Labrador');
console.log(d.speak());                            // 'Rex barks.' — own prototype
console.log(d instanceof Dog);                     // true
console.log(d instanceof Animal);                  // true — chain!
console.log(Object.getPrototypeOf(d) === Dog.prototype); // true

// Lookup chain: d → Dog.prototype → Animal.prototype → Object.prototype → null
```

---

### Ví dụ 3 — Nâng cao: Class syntax, `bind` và React gotcha
```js
// Class chỉ là syntactic sugar trên prototype
class EventEmitter {
  #listeners = new Map(); // private field (ES2022)

  on(event, listener) {
    if (!this.#listeners.has(event)) this.#listeners.set(event, []);
    this.#listeners.get(event).push(listener);
    return this; // fluent API
  }

  emit(event, ...args) {
    this.#listeners.get(event)?.forEach(fn => fn(...args));
    return this;
  }

  off(event, listener) {
    const fns = this.#listeners.get(event);
    if (fns) this.#listeners.set(event, fns.filter(fn => fn !== listener));
    return this;
  }
}

// React class component - this binding issue
class Counter extends React.Component {
  state = { count: 0 };

  // BUG: this là undefined khi được dùng như event handler
  handleClick_BAD() {
    this.setState({ count: this.state.count + 1 });
  }

  // FIX 1: Bind trong constructor
  constructor(props) {
    super(props);
    this.handleClick = this.handleClick.bind(this);
  }

  // FIX 2: Class field arrow function (phổ biến hơn)
  handleClick = () => { // arrow fn → lexical this → luôn là instance
    this.setState({ count: this.state.count + 1 });
  };

  render() {
    return <button onClick={this.handleClick}>{this.state.count}</button>;
  }
}

// Tại sao FIX 2 hoạt động?
// Arrow function dùng `this` từ lexical scope khi CLASS được instantiate
// `this` lúc đó = class instance → luôn đúng
```

---

## 4. PROMISE & ASYNC PATTERNS

### Lý thuyết & Bản chất
Promise là object đại diện cho **một giá trị sẽ có trong tương lai** (hoặc lý do tại sao không có). Có 3 trạng thái: `pending` → `fulfilled` hoặc `rejected` (không thể quay lại).

**Bản chất `.then()`:** Trả về một Promise MỚI. Điều này cho phép chaining. Nếu callback trả về Promise → `.then` tiếp theo chờ Promise đó resolve (flattening).

---

### Ví dụ 1 — Cơ bản: Promise chain vs async/await
```js
// Promise chain
fetchUser(1)
  .then(user => fetchPosts(user.id)) // trả Promise → auto flatten
  .then(posts => console.log(posts))
  .catch(err => console.error(err)) // catch mọi lỗi trong chain
  .finally(() => hideLoading());    // luôn chạy

// Tương đương async/await (dễ đọc hơn, nhất là khi có điều kiện)
async function loadUserPosts() {
  try {
    const user = await fetchUser(1);
    const posts = await fetchPosts(user.id);
    console.log(posts);
  } catch (err) {
    console.error(err);
  } finally {
    hideLoading();
  }
}
```

---

### Ví dụ 2 — Trung cấp: Parallel, Race, AllSettled
```js
// Sequential (chậm): await từng cái → tổng = sum(latencies)
async function sequential() {
  const user = await fetchUser(1);    // 200ms
  const posts = await fetchPosts(1);  // 300ms
  // Tổng: 500ms
}

// Parallel (nhanh): kick off cùng lúc → tổng = max(latencies)
async function parallel() {
  const [user, posts] = await Promise.all([
    fetchUser(1),   // 200ms
    fetchPosts(1),  // 300ms  ← chạy song song
  ]);
  // Tổng: 300ms + overhead nhỏ
  // Lưu ý: nếu 1 cái reject → cả Promise.all reject (fail-fast)
}

// Khi muốn biết kết quả của tất cả dù có lỗi
async function resilient() {
  const results = await Promise.allSettled([
    fetchUser(1),
    fetchPosts(1),
    fetchComments(1),
  ]);
  
  results.forEach(result => {
    if (result.status === 'fulfilled') console.log('OK:', result.value);
    else console.error('FAIL:', result.reason);
  });
}

// Race: dùng cho timeout pattern
async function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error(`Timeout after ${ms}ms`)), ms)
  );
  return Promise.race([promise, timeout]);
}
const data = await withTimeout(fetchSlowAPI(), 5000);
```

---

### Ví dụ 3 — Nâng cao: Promise từ đầu + Concurrency control
```js
// Implement Promise.all từ đầu (interview question)
function myPromiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (promises.length === 0) return resolve([]);
    
    const results = new Array(promises.length);
    let completed = 0;

    promises.forEach((p, i) => {
      Promise.resolve(p) // wrap để handle non-Promise values
        .then(value => {
          results[i] = value; // giữ đúng thứ tự
          if (++completed === promises.length) resolve(results);
        })
        .catch(reject); // any rejection → reject all
    });
  });
}

// Concurrency limiter — chạy tối đa N promises cùng lúc
async function runWithConcurrency(tasks, limit) {
  const results = [];
  const running = new Set();

  for (const task of tasks) {
    const promise = task().then(result => {
      running.delete(promise);
      return result;
    });
    running.add(promise);
    results.push(promise);

    if (running.size >= limit) {
      await Promise.race(running); // đợi 1 cái xong → slot trống
    }
  }

  return Promise.all(results);
}

// Dùng: tránh spam 1000 API calls cùng lúc
const results = await runWithConcurrency(
  urls.map(url => () => fetch(url).then(r => r.json())),
  5 // tối đa 5 requests song song
);
```

---

## 5. TYPESCRIPT GENERICS & ADVANCED TYPES

### Lý thuyết & Bản chất
TypeScript là **structural type system** — type compatibility dựa trên cấu trúc (shape), không phải tên. `{ x: number }` compatible với bất kỳ type nào có ít nhất field `x: number`.

**Generic** cho phép viết code hoạt động với nhiều type mà không mất type safety. Compiler "điền" type cụ thể vào lúc sử dụng.

---

### Ví dụ 1 — Cơ bản: Generic function & Utility types
```ts
// Generic function
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

const n = first([1, 2, 3]);     // TypeScript infer: n: number | undefined
const s = first(['a', 'b']);    // s: string | undefined

// Utility types phổ biến
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
}

type UserPreview = Pick<User, 'id' | 'name'>;    // { id: number; name: string }
type UserSafe = Omit<User, 'password'>;           // User không có password
type UpdateUser = Partial<User>;                  // tất cả optional — dùng cho PATCH request
type CreateUser = Omit<Required<User>, 'id'>;     // Required nhưng không có id

// Record type — map
type RolePermissions = Record<'admin' | 'user' | 'guest', string[]>;
const permissions: RolePermissions = {
  admin: ['read', 'write', 'delete'],
  user: ['read', 'write'],
  guest: ['read'],
};
```

---

### Ví dụ 2 — Trung cấp: Discriminated Union + Type Guards
```ts
// Discriminated Union — pattern rất quan trọng trong React/TypeScript
type LoadingState = { status: 'loading' };
type SuccessState<T> = { status: 'success'; data: T };
type ErrorState = { status: 'error'; error: Error };
type AsyncState<T> = LoadingState | SuccessState<T> | ErrorState;

// Type guard function
function isSuccess<T>(state: AsyncState<T>): state is SuccessState<T> {
  return state.status === 'success';
}

// Component sử dụng exhaustive checking
function renderUser(state: AsyncState<User>) {
  switch (state.status) {
    case 'loading': return <Spinner />;
    case 'success': return <UserCard user={state.data} />; // TS biết state.data tồn tại
    case 'error':   return <ErrorMsg error={state.error} />;
    default:
      // Exhaustive check: nếu thêm trạng thái mới → compiler báo lỗi ở đây
      const _: never = state;
      throw new Error(`Unhandled state: ${_}`);
  }
}

// Mapped Types
type Nullable<T> = { [K in keyof T]: T[K] | null };
type ReadonlyDeep<T> = {
  readonly [K in keyof T]: T[K] extends object ? ReadonlyDeep<T[K]> : T[K];
};
```

---

### Ví dụ 3 — Nâng cao: Conditional Types + Infer + Template Literal Types
```ts
// Conditional types
type IsArray<T> = T extends any[] ? true : false;
type ElementType<T> = T extends (infer E)[] ? E : never;

type A = ElementType<string[]>;  // string
type B = ElementType<number[]>;  // number
type C = ElementType<string>;    // never

// Infer — trích xuất type từ type khác
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type FirstArg<T> = T extends (first: infer F, ...rest: any[]) => any ? F : never;

type Fn = (name: string, age: number) => boolean;
type R = ReturnType<Fn>;   // boolean
type F = FirstArg<Fn>;     // string

// Template Literal Types
type EventName = 'click' | 'focus' | 'blur';
type Handler = `on${Capitalize<EventName>}`;  // 'onClick' | 'onFocus' | 'onBlur'

// Đặc biệt hữu ích cho API types
type ApiEndpoint = '/users' | '/posts' | '/comments';
type GetEndpoint = `GET ${ApiEndpoint}`;  // 'GET /users' | 'GET /posts' | ...

// Recursive types
type JSONValue =
  | string | number | boolean | null
  | JSONValue[]
  | { [key: string]: JSONValue };

// Builder pattern với TypeScript
class QueryBuilder<T extends Record<string, unknown>> {
  private filters: Partial<T> = {};
  
  where<K extends keyof T>(key: K, value: T[K]): this {
    this.filters[key] = value;
    return this;
  }
  
  build(): Partial<T> { return this.filters; }
}
```

---

## 6. DESTRUCTURING, SPREAD, OPTIONAL CHAINING

### Lý thuyết & Bản chất
Đây là **syntactic sugar** — code sugar giúp viết ngắn gọn hơn nhưng không thêm khả năng mới. Tuy nhiên, hiểu bản chất giúp tránh bug.

**Destructuring**: Gán giá trị từ array/object vào biến. Dùng **pattern matching** trên cấu trúc.

**Spread `...`**: Trong object/array literal = sao chép (shallow). Trong function call = unpack.

---

### Ví dụ 1 — Cơ bản: Object & Array Destructuring
```js
// Object destructuring
const { name, age, address: { city } = {} } = user; // nested + default
const { id: userId, ...rest } = user; // rename + rest

// Array destructuring
const [first, , third] = [1, 2, 3]; // bỏ qua phần tử giữa
const [head, ...tail] = array;

// Swap không cần temp variable
let a = 1, b = 2;
[a, b] = [b, a]; // a = 2, b = 1

// Function parameters
function displayUser({ name, age = 'Unknown', role = 'user' } = {}) {
  return `${name} (${age}) - ${role}`;
}
```

---

### Ví dụ 2 — Trung cấp: Shallow copy gotcha
```js
const original = {
  name: 'Alice',
  address: { city: 'Hanoi', zip: '10000' }, // nested object
  hobbies: ['coding', 'reading'],
};

const copy = { ...original }; // SHALLOW copy

copy.name = 'Bob';           // ✅ original.name vẫn là 'Alice'
copy.address.city = 'HCMC';  // ❌ original.address.city cũng thay đổi! (shared reference)
copy.hobbies.push('gaming'); // ❌ original.hobbies cũng thay đổi!

// Deep clone
const deepCopy = structuredClone(original); // ES2022, works với Date, Map, Set...
// Cũ hơn: JSON.parse(JSON.stringify(obj)) — không handle undefined, function, Date

// Immutable update pattern (Redux/React state)
const updatedUser = {
  ...original,
  address: { ...original.address, city: 'HCMC' }, // tạo object address mới
  hobbies: [...original.hobbies, 'gaming'],         // tạo array mới
};
```

---

### Ví dụ 3 — Nâng cao: Optional chaining + Nullish coalescing
```ts
// Optional chaining: trả undefined thay vì throw nếu path bị null/undefined
const city = user?.address?.city;                 // undefined nếu bất kỳ step nào null
const firstHobby = user?.hobbies?.[0];            // array access
const display = user?.getDisplayName?.();          // method call

// Nullish coalescing: fallback CHỈ khi null/undefined (không phải 0, '', false)
const name = user?.name ?? 'Anonymous';
const count = item?.count ?? 0;  // 0 là valid value! (khác || sẽ fallback khi count = 0)

// Practical example: safe deep access với transform
function getUserCity(data: unknown): string {
  if (
    typeof data === 'object' && data !== null &&
    'user' in data &&
    typeof (data as any).user === 'object'
  ) {
    return (data as any).user?.address?.city ?? 'Unknown';
  }
  return 'Unknown';
}

// Logical assignment operators (ES2021)
user.name ||= 'Default';       // assign nếu falsy
user.name &&= user.name.trim(); // assign nếu truthy
user.cache ??= {};              // assign nếu null/undefined
```

---

## TỔNG KẾT — Quick Reference

| Khái niệm | Bản chất 1 câu | Hay bị hỏi |
|-----------|----------------|-----------|
| Closure | Hàm nhớ lexical environment khi được khai báo | Var loop bug, module pattern |
| Event Loop | Microtask trước macrotask, sau khi call stack rỗng | Thứ tự output với setTimeout + Promise |
| `this` | Xác định lúc gọi, arrow fn thì lexical | React handler binding |
| Prototype | Chain lookup cho properties | instanceof, inheritance |
| Promise | Object đại diện giá trị tương lai, immutable khi settled | parallel vs sequential |
| Generic | Type variable, điền lúc dùng | Constraint với extends keyof |
| Discriminated Union | Union với common literal field | Exhaustive switch |
