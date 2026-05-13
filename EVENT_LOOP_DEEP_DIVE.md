# JavaScript Event Loop — Hiểu Tận Gốc

> Đọc từ đầu đến cuối theo thứ tự. Mỗi phần xây trên phần trước.

---

## PHẦN 1 — BỨC TRANH TỔNG QUAN

### Tại sao cần Event Loop?

JavaScript được thiết kế để chạy trong trình duyệt. Trình duyệt có 1 luồng chính (main thread) xử lý tất cả: chạy JS, render UI, xử lý click chuột. Nếu JS block luồng này → UI đứng hình, người dùng không click được gì.

Vấn đề: Nhiều việc tốn thời gian (gọi API, đọc file, set timer). Nếu JS đợi từng việc xong mới làm việc tiếp → ứng dụng cực kỳ chậm.

Giải pháp: **Non-blocking + Event Loop** — JS không đợi, giao việc nặng cho môi trường xử lý, khi xong thì quay lại.

```
Hình dung thế này:

Bạn (JS Engine) đang nấu ăn:
- Đặt nồi cơm vào nồi cơm điện (= gọi setTimeout/fetch)
- Không đứng chờ cơm chín
- Tiếp tục rửa rau, thái thịt (= chạy code tiếp)
- Khi cơm chín, chuông reo (= callback vào queue)
- Bạn nghe chuông, xới cơm ra (= Event Loop lấy callback chạy)
```

---

## PHẦN 2 — CÁC THÀNH PHẦN

Đây là toàn bộ hệ thống. Hiểu từng thành phần trước, sau đó xem chúng phối hợp thế nào.

```
┌─────────────────────────────────────────────────────────────┐
│                     JS ENGINE (V8)                          │
│                                                             │
│   ┌──────────────┐          ┌────────────────────────────┐  │
│   │  Call Stack  │          │           Heap             │  │
│   │              │          │  (Object memory storage)   │  │
│   │  main()      │          │                            │  │
│   │  fn2()       │          │  { user: {...} }           │  │
│   │  fn1()       │          │  [1, 2, 3, ...]            │  │
│   └──────────────┘          └────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│               BROWSER / NODE.js APIs                        │
│                                                             │
│   setTimeout()   fetch()   DOM Events   setInterval()       │
│   (Chạy NGOÀI JS engine — không block Call Stack)           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      QUEUES                                 │
│                                                             │
│  ┌────────────────────────────┐  ┌───────────────────────┐  │
│  │   Microtask Queue          │  │  Macrotask Queue       │  │
│  │   (ƯU TIÊN CAO HƠN)        │  │  (Task Queue)          │  │
│  │                            │  │                        │  │
│  │  Promise .then()           │  │  setTimeout callback   │  │
│  │  Promise .catch()          │  │  setInterval callback  │  │
│  │  queueMicrotask()          │  │  DOM event handlers    │  │
│  │  MutationObserver          │  │  I/O callbacks         │  │
│  │  await (tiếp theo)         │  │  MessageChannel        │  │
│  └────────────────────────────┘  └───────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     EVENT LOOP                              │
│                                                             │
│   Một vòng lặp vô hạn, liên tục kiểm tra:                  │
│   "Call Stack rỗng chưa? Có gì trong queue không?"         │
└─────────────────────────────────────────────────────────────┘
```

---

### 2.1 Call Stack — Nơi code thực sự chạy

**Bản chất:** Stack (LIFO — Last In, First Out). Mỗi khi gọi function → push frame vào stack. Function return → pop frame ra.

**Chỉ có 1 Call Stack** → JavaScript single-threaded.

```js
function multiply(a, b) {
  return a * b;       // frame 3: multiply
}

function square(n) {
  return multiply(n, n); // frame 2: square → gọi multiply
}

function main() {
  const result = square(4); // frame 1: main → gọi square
  console.log(result);
}

main();
```

```
Luồng Call Stack:

Push main()       → [main]
Push square(4)    → [main, square]
Push multiply(4,4)→ [main, square, multiply]
multiply return 16→ [main, square]           ← pop multiply
square return 16  → [main]                   ← pop square
console.log(16)   → [main, console.log]
console.log done  → [main]                   ← pop console.log
main return       → []                       ← pop main (STACK RỖNG)
```

**Stack Overflow:** Gọi đệ quy không có điều kiện dừng → stack đầy → `RangeError: Maximum call stack size exceeded`.

---

### 2.2 Heap — Nơi lưu data

Object, array, function được lưu trong Heap (bộ nhớ động). Call Stack chỉ giữ **reference** (địa chỉ) trỏ vào Heap.

```js
const user = { name: 'Alice' }; // object thực sự ở Heap
// Call Stack giữ biến `user` = địa chỉ trong Heap
```

---

### 2.3 Web APIs / Node.js APIs — Nơi xử lý async

Không phải JS Engine. Là môi trường bên ngoài cung cấp khả năng async.

| API | Môi trường |
|-----|-----------|
| `setTimeout`, `setInterval` | Browser + Node.js |
| `fetch`, `XMLHttpRequest` | Browser |
| `DOM events` (click, keydown...) | Browser |
| `fs.readFile`, `http.request` | Node.js |
| `MutationObserver` | Browser |

**Cơ chế:**
1. JS gọi `setTimeout(fn, 1000)` → **giao** cho Timer API, tiếp tục chạy code khác
2. Timer API đếm 1000ms (ngoài Call Stack, không block)
3. Hết 1000ms → Timer API đẩy `fn` vào **Macrotask Queue**
4. Event Loop lấy `fn` từ queue → chạy

---

### 2.4 Microtask Queue — Hàng đợi ưu tiên cao

**Những gì vào đây:**
- `Promise.then()`, `Promise.catch()`, `Promise.finally()`
- `await` (phần code sau `await`)
- `queueMicrotask(fn)`
- `MutationObserver` callback

**Quy tắc quan trọng nhất:**
> Sau khi Call Stack rỗng, **TOÀN BỘ** Microtask Queue phải được làm trống **TRƯỚC** khi lấy bất kỳ Macrotask nào.

---

### 2.5 Macrotask Queue (Task Queue) — Hàng đợi thường

**Những gì vào đây:**
- `setTimeout` callback
- `setInterval` callback
- DOM event handlers (`onclick`, `onkeydown`...)
- `MessageChannel`
- I/O callbacks

**Quy tắc:** Mỗi vòng Event Loop chỉ lấy **1 macrotask** → chạy → sau đó làm trống toàn bộ microtask queue → render (nếu cần) → lấy macrotask tiếp theo.

---

## PHẦN 3 — VÒNG LẶP EVENT LOOP — LUỒNG HOẠT ĐỘNG CHÍNH XÁC

```
┌─────────────────────────────────────────────────────┐
│                  EVENT LOOP CYCLE                   │
│                                                     │
│  1. Chạy toàn bộ synchronous code                  │
│     (cho đến khi Call Stack rỗng)                   │
│            │                                        │
│            ▼                                        │
│  2. Làm trống TOÀN BỘ Microtask Queue               │
│     (chạy hết microtask, kể cả microtask            │
│      được thêm bởi microtask khác)                  │
│            │                                        │
│            ▼                                        │
│  3. Render (browser repaint nếu cần)                │
│     [chỉ ở browser, không ở Node.js]                │
│            │                                        │
│            ▼                                        │
│  4. Lấy 1 Macrotask từ Macrotask Queue              │
│     → đưa vào Call Stack → chạy                    │
│            │                                        │
│            ▼                                        │
│  5. Quay lại bước 2                                 │
└─────────────────────────────────────────────────────┘
```

**Điểm mấu chốt:**
- Bước 2 luôn chạy sau bước 1 và sau mỗi macrotask
- Bước 2 không dừng cho đến khi queue **hoàn toàn rỗng**
- Bước 4 chỉ lấy **đúng 1** macrotask mỗi vòng

---

## PHẦN 4 — VÍ DỤ TỪNG BƯỚC (ĐỌC NHƯ TRACING)

### Ví dụ 1 — Cơ bản nhất: Thứ tự output

```js
console.log('A');           // (1)

setTimeout(() => {
  console.log('B');         // (4) — macrotask
}, 0);

Promise.resolve().then(() => {
  console.log('C');         // (3) — microtask
});

console.log('D');           // (2)
```

**Trace từng bước:**

```
BƯỚC 1: console.log('A') vào Call Stack
  → In ra: "A"
  → Pop ra khỏi stack

BƯỚC 2: setTimeout(..., 0) vào Call Stack
  → Giao callback cho Timer API với delay=0
  → Pop ra (setTimeout tự nó chạy xong ngay)
  → Timer API: delay=0 → ngay lập tức đẩy callback vào Macrotask Queue
  Macrotask Queue: [() => console.log('B')]

BƯỚC 3: Promise.resolve().then(...) vào Call Stack
  → Promise đã resolved → .then callback vào Microtask Queue ngay
  → Pop ra
  Microtask Queue: [() => console.log('C')]

BƯỚC 4: console.log('D') vào Call Stack
  → In ra: "D"
  → Pop ra

BƯỚC 5: Call Stack RỖNG → Event Loop kiểm tra Microtask Queue
  → Có: () => console.log('C')
  → Đưa vào Call Stack → chạy → In ra: "C"
  → Microtask Queue rỗng

BƯỚC 6: Event Loop kiểm tra Macrotask Queue
  → Có: () => console.log('B')
  → Đưa vào Call Stack → chạy → In ra: "B"

OUTPUT: A → D → C → B
```

---

### Ví dụ 2 — Microtask sinh ra Microtask

```js
console.log('start');

Promise.resolve()
  .then(() => {
    console.log('microtask 1');
    // Microtask này sinh ra microtask mới!
    return Promise.resolve();
  })
  .then(() => {
    console.log('microtask 2');
  });

setTimeout(() => console.log('macrotask'), 0);

console.log('end');
```

**Trace:**

```
Call Stack chạy sync code:
  → "start"
  → Tạo Promise chain, .then đầu → vào Microtask Queue
  → setTimeout → callback vào Macrotask Queue
  → "end"
  → Stack RỖNG

Microtask Queue: [handler1]  ← chỉ có handler1 lúc này

Event Loop làm trống Microtask Queue:
  → Chạy handler1 → In "microtask 1"
  → handler1 return Promise.resolve() → .then tiếp theo vào Microtask Queue
  Microtask Queue: [handler2]  ← mới được thêm vào!
  → Queue chưa rỗng → tiếp tục
  → Chạy handler2 → In "microtask 2"
  Microtask Queue: []  ← RỖNG

Event Loop lấy Macrotask:
  → Chạy setTimeout callback → In "macrotask"

OUTPUT: start → end → microtask 1 → microtask 2 → macrotask
```

**Kết luận:** Microtask mới được thêm trong khi đang xử lý Microtask Queue → vẫn được xử lý trong cùng "lượt", TRƯỚC khi macrotask.

---

### Ví dụ 3 — async/await là Promise "ẩn"

```js
async function foo() {
  console.log('foo start');        // (2)
  await Promise.resolve();
  console.log('foo after await');  // (5) — microtask
}

console.log('before');             // (1)
foo();
console.log('after');              // (3)
```

**Hiểu `await`:**
```
await X  ≡  .then(continuation)

Khi gặp await:
1. Evaluate X (Promise.resolve())
2. "Tạm dừng" foo(), trả control về caller
3. Đăng ký phần code sau await vào Microtask Queue (khi X resolve)
4. Caller tiếp tục chạy
```

**Trace:**

```
→ "before"
→ Gọi foo() → vào stack
  → "foo start"
  → Gặp await Promise.resolve()
    → Promise đã resolve ngay
    → Đăng ký ("foo after await") vào Microtask Queue
    → foo() tạm dừng, trả control về
→ "after"
→ Stack RỖNG

Microtask Queue: [("foo after await")]
→ Chạy → "foo after await"

OUTPUT: before → foo start → after → foo after await
```

---

### Ví dụ 4 — Nhiều await liên tiếp

```js
async function bar() {
  console.log('1');
  await null;           // 1 microtask tick
  console.log('2');
  await null;           // 1 microtask tick nữa
  console.log('3');
}

console.log('A');
bar();
console.log('B');
```

**Trace:**

```
→ "A"
→ bar() gọi
  → "1"
  → await null → đặt "2 và 3" vào microtask, tạm dừng
→ "B"
→ Stack rỗng → Microtask: phần code sau await đầu
  → "2"
  → await null → đặt "3" vào microtask
→ Stack rỗng → Microtask: phần code sau await thứ 2
  → "3"

OUTPUT: A → 1 → B → 2 → 3

Mỗi await = 1 "lần nhường" cho Event Loop.
Mỗi lần nhường = các sync code đang chờ có cơ hội chạy.
```

---

### Ví dụ 5 — Câu hỏi phỏng vấn kinh điển

```js
console.log('1');

setTimeout(function() {
  console.log('2');
  Promise.resolve().then(function() {
    console.log('3');
  });
}, 0);

Promise.resolve().then(function() {
  console.log('4');
  setTimeout(function() {
    console.log('5');
  }, 0);
});

console.log('6');
```

**Trace chi tiết:**

```
=== SYNC CODE ===
→ "1"
→ setTimeout A (→ console.log('2') + tạo promise) → Macrotask Queue: [A]
→ Promise.resolve().then(→ console.log('4') + setTimeout B) → Microtask: [M1]
→ "6"
→ Stack RỖNG

=== MICROTASK QUEUE (làm trống) ===
→ Chạy M1:
  → "4"
  → setTimeout B → Macrotask Queue: [A, B]
→ Microtask Queue RỖNG

=== MACROTASK: lấy A ===
→ Chạy A:
  → "2"
  → Promise.resolve().then(→ console.log('3')) → Microtask: [M2]
→ Stack RỖNG

=== MICROTASK QUEUE (làm trống) ===
→ Chạy M2:
  → "3"
→ Microtask Queue RỖNG

=== MACROTASK: lấy B ===
→ Chạy B:
  → "5"

OUTPUT: 1 → 6 → 4 → 2 → 3 → 5
```

**Quy tắc nhớ:** Sau mỗi Macrotask → làm trống toàn bộ Microtask → rồi mới lấy Macrotask tiếp.

---

## PHẦN 5 — async/await SÂU HƠN

### 5.1 await bên trong và bên ngoài async

```js
// KHÔNG phải await → chạy sync
async function fetchData() {
  const result = await fetch('/api/data'); // dừng ở đây
  return result.json();                    // chạy sau khi fetch resolve
}

// Gọi async function trả về Promise
const promise = fetchData();   // bắt đầu chạy, trả Promise ngay
// promise.then(data => ...)   // hoặc await ở ngoài
```

---

### 5.2 Parallel vs Sequential với await

```js
// SEQUENTIAL — chạy lần lượt: 1s + 2s = 3s
async function sequential() {
  const user  = await fetchUser();   // đợi 1s
  const posts = await fetchPosts();  // đợi 2s (sau khi user xong)
  return { user, posts };
}

// PARALLEL — chạy song song: max(1s, 2s) = 2s
async function parallel() {
  const userPromise  = fetchUser();    // kick off ngay (không await)
  const postsPromise = fetchPosts();   // kick off ngay (không await)

  const user  = await userPromise;    // đợi kết quả
  const posts = await postsPromise;   // đợi kết quả (có thể đã xong rồi)
  return { user, posts };
}

// Ngắn gọn hơn với Promise.all
async function parallelClean() {
  const [user, posts] = await Promise.all([fetchUser(), fetchPosts()]);
  return { user, posts };
}
```

---

### 5.3 Error handling với async/await

```js
// Cách 1: try/catch
async function loadUser(id) {
  try {
    const user = await fetchUser(id);
    return user;
  } catch (error) {
    console.error('Failed:', error);
    return null;
  }
}

// Cách 2: .catch() trên Promise
async function loadUserAlt(id) {
  const user = await fetchUser(id).catch(() => null);
  // nếu fetchUser reject → user = null (không throw)
  return user;
}

// Cách 3: wrapper helper
async function tryCatch(promise) {
  try {
    const data = await promise;
    return [null, data];
  } catch (error) {
    return [error, null];
  }
}

const [error, user] = await tryCatch(fetchUser(id));
if (error) { /* handle */ }
```

---

## PHẦN 6 — PROMISE INTERNALS

### 6.1 Promise States

```
           ┌─────────────────────────────────┐
           │                                 │
           │            PENDING              │
           │     (giá trị chưa có)           │
           │                                 │
           └─────────────┬───────────────────┘
                         │
          ┌──────────────┴──────────────────┐
          │                                 │
          ▼ resolve(value)                  ▼ reject(reason)
  ┌───────────────┐                ┌───────────────────┐
  │   FULFILLED   │                │     REJECTED      │
  │ (có giá trị)  │                │  (có lý do lỗi)   │
  └───────────────┘                └───────────────────┘

Một khi đã settled (fulfilled hoặc rejected) → KHÔNG thể thay đổi.
```

---

### 6.2 Promise Chain — Tại sao chain được?

```js
// .then() trả về Promise MỚI → có thể chain tiếp
fetch('/api/user')
  .then(res => res.json())       // Promise A
  .then(user => user.name)       // Promise B — nhận giá trị từ A
  .then(name => name.toUpperCase()) // Promise C
  .catch(err => console.error(err)); // bắt lỗi từ BẤT KỲ step nào trên

// Nếu .then callback trả về:
// - Giá trị thường → Promise mới wrap giá trị đó (fulfilled)
// - Promise → Promise mới "theo" Promise đó (flatten)
// - throw error → Promise mới rejected với error đó
```

---

### 6.3 Promise.all vs Promise.allSettled vs Promise.race vs Promise.any

```js
const p1 = fetch('/api/user');      // resolve sau 100ms
const p2 = fetch('/api/posts');     // resolve sau 200ms
const p3 = fetch('/api/error');     // reject sau 50ms

// Promise.all: TẤT CẢ resolve → fulfilled. 1 cái reject → toàn bộ reject (fail-fast)
await Promise.all([p1, p2, p3]);
// → reject ngay khi p3 reject ở 50ms

// Promise.allSettled: đợi TẤT CẢ xong (dù resolve hay reject)
const results = await Promise.allSettled([p1, p2, p3]);
// results = [
//   { status: 'fulfilled', value: userData },
//   { status: 'fulfilled', value: postsData },
//   { status: 'rejected',  reason: Error('...') }
// ]

// Promise.race: lấy KẾT QUẢ ĐẦU TIÊN (dù resolve hay reject)
await Promise.race([p1, p2, p3]);
// → reject ở 50ms (p3 reject trước)

// Promise.any (ES2021): lấy RESOLVE ĐẦU TIÊN, bỏ qua reject
// Chỉ reject khi TẤT CẢ đều reject
await Promise.any([p1, p2, p3]);
// → resolve ở 100ms (p1) — bỏ qua p3 reject
```

---

## PHẦN 7 — NODE.js EVENT LOOP (khác browser)

Node.js có Event Loop riêng, chi tiết hơn với nhiều **phases**:

```
   ┌───────────────────────────┐
┌─►│           timers          │  ← setTimeout, setInterval callbacks
│  └─────────────┬─────────────┘
│                │
│  ┌─────────────▼─────────────┐
│  │     pending callbacks     │  ← I/O errors từ vòng trước
│  └─────────────┬─────────────┘
│                │
│  ┌─────────────▼─────────────┐
│  │       idle, prepare       │  ← internal
│  └─────────────┬─────────────┘
│                │
│  ┌─────────────▼─────────────┐
│  │           poll            │  ← Chờ I/O events (đọc file, network...)
│  │                           │    Nếu không có gì → block chờ ở đây
│  └─────────────┬─────────────┘
│                │
│  ┌─────────────▼─────────────┐
│  │           check           │  ← setImmediate callbacks
│  └─────────────┬─────────────┘
│                │
│  ┌─────────────▼─────────────┐
└──┤      close callbacks      │  ← socket.on('close', ...)
   └───────────────────────────┘

Sau MỖI PHASE → làm trống Microtask Queue (process.nextTick + Promise)
```

### process.nextTick — ưu tiên cao hơn cả Promise!

```js
console.log('1');

setTimeout(() => console.log('setTimeout'), 0);

Promise.resolve().then(() => console.log('Promise'));

process.nextTick(() => console.log('nextTick'));

console.log('2');

// OUTPUT: 1 → 2 → nextTick → Promise → setTimeout

// process.nextTick chạy trước Promise.then!
// Thứ tự ưu tiên trong Node.js:
// 1. Synchronous code
// 2. process.nextTick queue  ← cao nhất trong async
// 3. Promise microtask queue
// 4. Macrotask (timers, I/O, setImmediate)
```

### setImmediate vs setTimeout(fn, 0) trong Node.js

```js
// Trong I/O callback → setImmediate luôn trước setTimeout
fs.readFile('file.txt', () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
  // OUTPUT: immediate → timeout (luôn luôn, trong I/O callback)
});

// Ngoài I/O callback → thứ tự không đảm bảo (phụ thuộc timing)
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
// OUTPUT: không xác định
```

---

## PHẦN 8 — GOTCHAS & PITFALLS

### 8.1 setTimeout(fn, 0) không phải "ngay lập tức"

```js
// Delay thực tế của setTimeout(fn, 0) là ≥ 4ms trong browser (HTML spec)
// Và không chạy cho đến khi Call Stack rỗng VÀ microtask xong

console.log('start');

setTimeout(() => console.log('timeout'), 0);

// Dù timeout = 0, vẫn phải đợi sync code + microtask xong
for (let i = 0; i < 1_000_000; i++) {} // block 50ms
Promise.resolve().then(() => console.log('microtask'));

console.log('end');

// OUTPUT: start → end → microtask → timeout
// timeout callback chạy KHÔNG PHẢI sau 0ms mà sau khi stack rỗng + microtask xong
```

---

### 8.2 Long task blocking — UI freeze

```js
// BAD: block Call Stack → UI đóng băng, không thể click, scroll
function processLargeArray(arr) {
  return arr.map(item => heavyCompute(item)); // block 500ms
}

// GOOD: chia nhỏ với scheduler
async function processLargeArrayAsync(arr) {
  const results = [];
  for (let i = 0; i < arr.length; i++) {
    results.push(heavyCompute(arr[i]));

    // Cứ mỗi 100 items → nhường cho Event Loop 1 tick
    // Browser có thể render frame, xử lý user input
    if (i % 100 === 0) {
      await new Promise(resolve => setTimeout(resolve, 0));
    }
  }
  return results;
}
```

---

### 8.3 Microtask infinite loop — starving macrotasks

```js
// NGUY HIỂM: microtask liên tục sinh microtask mới
// → Macrotask không bao giờ chạy → UI freeze dù không block sync

function infiniteMicrotask() {
  Promise.resolve().then(() => {
    // Sinh microtask mới → Event Loop không bao giờ thoát khỏi microtask phase!
    infiniteMicrotask();
  });
}

setTimeout(() => console.log('This will NEVER print'), 0);
infiniteMicrotask(); // block vĩnh viễn
```

---

### 8.4 Race condition với async

```js
// Component React fetch data khi userId thay đổi
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    // BUG: userId thay đổi nhanh (1 → 2 → 3)
    // Request có thể về theo thứ tự khác (3 → 1 → 2)
    // → setUser(user1) chạy sau → hiển thị sai user!
    fetchUser(userId).then(setUser);
  }, [userId]);

  // FIX: cancellation flag
  useEffect(() => {
    let cancelled = false;

    fetchUser(userId).then(user => {
      if (!cancelled) setUser(user); // chỉ update nếu effect này vẫn valid
    });

    return () => { cancelled = true; }; // cleanup khi userId thay đổi
  }, [userId]);
}
```

---

### 8.5 Unhandled Promise rejection

```js
// BAD: reject không được handle → UnhandledPromiseRejectionWarning
const promise = fetch('/api/data');
// Không .catch() hay try/catch → lỗi im lặng

// GOOD: luôn handle rejection
fetch('/api/data')
  .then(r => r.json())
  .catch(err => console.error(err));

// hoặc
try {
  const data = await fetch('/api/data').then(r => r.json());
} catch (err) {
  console.error(err);
}

// Node.js: lắng nghe unhandled rejections
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled rejection:', reason);
  process.exit(1); // crash thay vì silent fail
});
```

---

## PHẦN 9 — MENTAL MODEL — CÁCH NHỚ NHANH

### Quy tắc 5 chữ: **S-M-R-1-M**

```
S — Synchronous code trước hết (làm trống Call Stack)
M — Microtask queue tiếp theo (làm TRỐNG hoàn toàn)
R — Render (browser repaint nếu cần)
1 — 1 Macrotask duy nhất
M — Microtask queue lại (sau macrotask đó)
→ Lặp lại M-R-1-M mãi mãi
```

### Phân loại nhanh: "Cái này vào queue nào?"

```
Promise .then / .catch / .finally  → MICROTASK
await (code sau await)             → MICROTASK
queueMicrotask()                   → MICROTASK
process.nextTick() [Node.js]       → MICROTASK (ưu tiên nhất)

setTimeout / setInterval           → MACROTASK
DOM events (click, keydown...)     → MACROTASK
fetch / XHR callbacks              → MACROTASK (sau khi vào queue qua Web APIs)
setImmediate() [Node.js]           → MACROTASK (check phase)
MessageChannel                     → MACROTASK
```

---

## PHẦN 10 — BÀI TẬP TỰ KIỂM TRA

### Bài 1 — Dự đoán output

```js
// Dự đoán output TRƯỚC KHI chạy
async function test() {
  console.log('1');
  await Promise.resolve();
  console.log('2');
  await Promise.resolve();
  console.log('3');
}

console.log('A');
test();
console.log('B');

// Đáp án: A → 1 → B → 2 → 3
```

---

### Bài 2 — Output phức tạp hơn

```js
setTimeout(() => console.log('t1'), 0);

async function async1() {
  console.log('async1 start');
  await async2();
  console.log('async1 end');
}

async function async2() {
  console.log('async2');
}

console.log('start');
async1();
setTimeout(() => console.log('t2'), 0);
console.log('end');

/*
Đáp án:
start
async1 start
async2
end
async1 end
t1
t2

Giải thích:
1. "start" — sync
2. async1() gọi → "async1 start"
3. await async2() → async2() chạy → "async2"
4. async2 return → resolved → phần sau await vào microtask
5. async1 tạm dừng, trả control → setTimeout t2 vào macrotask queue
6. "end" — sync
7. Stack rỗng → microtask: "async1 end"
8. Macrotask: "t1", sau đó "t2"
*/
```

---

### Bài 3 — Tìm lỗi

```js
// Code này có bug gì?
async function loadPosts() {
  const userId = await getCurrentUser().id;  // BUG!
  return fetchPosts(userId);
}

// Sửa:
async function loadPosts() {
  const user = await getCurrentUser();
  return fetchPosts(user.id);
}

// Giải thích bug:
// await getCurrentUser() → resolve → trả về User object
// Nhưng .id truy cập trên Promise (chưa await) → undefined
// getCurrentUser().id = undefined (Promise không có .id)
// await undefined → undefined
```

---

### Bài 4 — Tối ưu code

```js
// Code dưới đây chạy được nhưng chậm. Tại sao và fix thế nào?

async function getProfile(userId) {
  const user = await db.findUser(userId);        // 100ms
  const posts = await db.findPosts(userId);       // 200ms
  const followers = await db.findFollowers(userId); // 150ms
  return { user, posts, followers };
  // Tổng: 450ms (sequential)
}

// Fix: chạy song song
async function getProfileFast(userId) {
  const [user, posts, followers] = await Promise.all([
    db.findUser(userId),
    db.findPosts(userId),
    db.findFollowers(userId),
  ]);
  return { user, posts, followers };
  // Tổng: 200ms (max của 3 requests)
}
```

---

## TỔNG KẾT — 10 Điều Cần Nhớ

```
1. JS single-threaded — 1 Call Stack, 1 thứ tại 1 thời điểm

2. Async KHÔNG nghĩa là multi-thread — là non-blocking I/O
   Công việc nặng giao cho Web APIs / OS, JS tiếp tục chạy

3. MICROTASK ưu tiên hơn MACROTASK — luôn luôn

4. Sau mỗi task: làm trống TOÀN BỘ microtask queue trước khi lấy task tiếp

5. Mỗi vòng Event Loop chỉ lấy 1 macrotask

6. Microtask sinh ra microtask vẫn được xử lý trong cùng lượt

7. setTimeout(fn, 0) ≠ "ngay lập tức" — ít nhất phải đợi microtask xong

8. await = .then() dưới dạng syntax sugar — mỗi await = 1 microtask tick

9. Promise.all = song song, Promise.allSettled = đợi tất cả, Promise.any = resolve đầu tiên

10. process.nextTick [Node.js] ưu tiên CAO HƠN Promise.then trong microtask
```
