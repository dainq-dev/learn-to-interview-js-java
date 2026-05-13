# Event Loop: Browser vs Node.js — So sánh tận gốc

---

## PHẦN 1 — BROWSER: 3 TẦNG RÕ RÀNG

Trước khi sang Node.js, cần nắm chắc Browser model để so sánh.

```
BROWSER EVENT LOOP — 3 TẦNG

┌──────────────────────────────────────────┐
│  TẦNG 1: CALL STACK                      │  ← Code đang chạy
│  (synchronous execution)                 │
└──────────────────────────────────────────┘
              ↓ khi rỗng
┌──────────────────────────────────────────┐
│  TẦNG 2: MICROTASK QUEUE                 │  ← Ưu tiên cao
│                                          │
│  • Promise .then / .catch / .finally     │
│  • await (phần tiếp theo sau await)      │
│  • queueMicrotask()                      │
│  • MutationObserver                      │
└──────────────────────────────────────────┘
              ↓ khi microtask rỗng
┌──────────────────────────────────────────┐
│  TẦNG 3: MACROTASK QUEUE                 │  ← Lấy 1 cái mỗi vòng
│                                          │
│  • setTimeout / setInterval              │
│  • DOM event handlers                    │
│  • MessageChannel                        │
│  • requestAnimationFrame (đặc biệt)      │
└──────────────────────────────────────────┘
```

**Quy trình Browser (đơn giản):**
```
1. Chạy sync code → stack rỗng
2. Làm trống microtask queue (kể cả microtask sinh microtask mới)
3. Render frame nếu cần (60fps ~ mỗi 16ms)
4. Lấy 1 macrotask → chạy → quay lại bước 2
```

---

## PHẦN 2 — NODE.JS: KHÔNG PHẢI 3 TẦNG, MÀ LÀ 6 PHASES

Đây là điểm khác biệt lớn nhất. Node.js không có 1 "Macrotask Queue" đơn giản — nó có **6 phases** riêng biệt chạy theo thứ tự vòng tròn.

```
NODE.JS EVENT LOOP — CẤU TRÚC ĐẦY ĐỦ

┌──────────────────────────────────────────────────────────────┐
│  TẦNG 1: CALL STACK (giống Browser)                         │
│  synchronous code                                            │
└──────────────────────────────────────────────────────────────┘
              ↓ khi rỗng
┌──────────────────────────────────────────────────────────────┐
│  TẦNG 2: MICROTASK QUEUE (chia làm 2 hàng đợi)              │
│                                                              │
│  [2a] process.nextTick queue  ← CHẠY TRƯỚC                  │
│       • process.nextTick(fn)                                 │
│                                                              │
│  [2b] Promise microtask queue ← CHẠY SAU 2a                 │
│       • Promise .then / .catch                               │
│       • await (tiếp theo)                                    │
│       • queueMicrotask()                                     │
└──────────────────────────────────────────────────────────────┘
              ↓ khi cả 2a và 2b đều rỗng
┌──────────────────────────────────────────────────────────────┐
│  TẦNG 3: MACROTASK — 6 PHASES (chạy tuần tự theo vòng)      │
│                                                              │
│  ┌─────────────┐  ┌────────────┐  ┌───────────────────────┐ │
│  │  1. TIMERS  │→ │2. PENDING  │→ │  3. IDLE/PREPARE      │ │
│  │             │  │ CALLBACKS  │  │  (internal only)      │ │
│  │ setTimeout  │  │I/O errors  │  └───────────────────────┘ │
│  │ setInterval │  │từ vòng trước│             ↓             │
│  └─────────────┘  └────────────┘  ┌───────────────────────┐ │
│         ↑                         │      4. POLL          │ │
│         │                         │  Chờ & xử lý I/O      │ │
│  ┌──────┴──────┐  ┌────────────┐  │  (đọc file, network)  │ │
│  │ 6. CLOSE    │← │  5. CHECK  │← └───────────────────────┘ │
│  │  CALLBACKS  │  │            │                            │
│  │socket.close │  │setImmediate│                            │
│  └─────────────┘  └────────────┘                            │
│                                                              │
│  *** Sau MỖI PHASE → làm trống Microtask Queue (2a + 2b) ***│
└──────────────────────────────────────────────────────────────┘
```

---

## PHẦN 3 — GIẢI THÍCH TỪNG PHASE

### Phase 1: TIMERS
Chạy callbacks của `setTimeout` và `setInterval` **đã hết hạn**.

```
Lưu ý: "đã hết hạn" = thời gian delay đã qua.
Nếu setTimeout(fn, 100) nhưng poll phase mất 200ms → fn vẫn chờ đến timers phase.
Delay là MINIMUM, không phải EXACT.
```

### Phase 2: PENDING CALLBACKS
Chạy I/O callbacks bị defer từ vòng trước (ví dụ: TCP error callbacks).
Ít gặp trong code thông thường.

### Phase 3: IDLE, PREPARE
Internal only — Node.js dùng nội bộ, không liên quan đến code người dùng.

### Phase 4: POLL ← QUAN TRỌNG NHẤT
Đây là "trái tim" của Node.js Event Loop:

```
POLL phase làm 2 việc:

1. Tính toán bao lâu cần block để chờ I/O
2. Xử lý các I/O events trong poll queue

Logic:
if (poll queue có việc):
    → Chạy hết tất cả callbacks trong poll queue
    → Dừng khi queue rỗng HOẶC đạt system-dependent limit

else if (poll queue rỗng):
    if (có setImmediate):
        → Kết thúc poll phase, chuyển sang CHECK phase
    else if (có timer hết hạn):
        → Quay lại TIMERS phase
    else:
        → BLOCK tại đây, chờ I/O mới
```

Đây là lý do Node.js "không cần tạo thread" cho I/O — nó ngủ tại poll phase, OS đánh thức khi có data.

### Phase 5: CHECK
Chạy `setImmediate` callbacks. `setImmediate` được thiết kế để chạy **sau poll phase hiện tại**.

### Phase 6: CLOSE CALLBACKS
Chạy close events: `socket.on('close', ...)`, `fs.ReadStream.on('close', ...)`.

---

## PHẦN 4 — SO SÁNH TRỰC TIẾP: BROWSER vs NODE.JS

```
┌────────────────────┬──────────────────────────┬──────────────────────────────┐
│                    │       BROWSER             │          NODE.JS             │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ Call Stack         │ ✅ Giống nhau             │ ✅ Giống nhau                │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ Microtask          │ 1 hàng đợi duy nhất:      │ 2 hàng đợi:                  │
│                    │ • Promise .then           │ [1] process.nextTick (trước) │
│                    │ • queueMicrotask          │ [2] Promise .then (sau)      │
│                    │ • MutationObserver        │ • queueMicrotask             │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ Macrotask          │ 1 hàng đợi:               │ 6 phases vòng tròn           │
│                    │ • setTimeout              │ Timer → Pending → Idle →     │
│                    │ • setInterval             │ Poll → Check → Close         │
│                    │ • DOM events              │                              │
│                    │ • MessageChannel          │                              │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ setTimeout(fn,0)   │ Delay tối thiểu ≥ 4ms    │ Delay tối thiểu ~1ms         │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ setImmediate       │ ❌ Không có               │ ✅ CHECK phase               │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ process.nextTick   │ ❌ Không có               │ ✅ Microtask, ưu tiên cao nhất│
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ requestAnimationFrame│ ✅ Trước render frame  │ ❌ Không có                  │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ Render             │ ✅ Sau microtask          │ ❌ Không có (no UI)          │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ Microtask sau      │ Sau mỗi macrotask         │ Sau MỖI PHASE                │
│ macrotask          │                           │ (chi tiết hơn nhiều)         │
└────────────────────┴──────────────────────────┴──────────────────────────────┘
```

---

## PHẦN 5 — THỨ TỰ ƯU TIÊN TRONG NODE.JS

```
Ưu tiên từ cao → thấp:

1. Synchronous code (Call Stack)
         │
         ▼
2. process.nextTick queue     ← CAO NHẤT trong async
         │
         ▼
3. Promise microtask queue    ← .then, await, queueMicrotask
         │
         ▼
4. setImmediate (CHECK phase) ← Trong I/O callback: trước setTimeout
         │
         ▼
5. setTimeout / setInterval (TIMERS phase)
         │
         ▼
6. I/O callbacks (POLL phase)
         │
         ▼
7. close callbacks (CLOSE phase)
```

**Điểm khác biệt lớn nhất so với Browser:**
> `process.nextTick` ưu tiên hơn `Promise.then` — điều này KHÔNG có trong Browser.

---

## PHẦN 6 — VÍ DỤ CÓ THỂ CHẠY ĐƯỢC

> Chạy bằng: `bun run <file>.ts` hoặc `node <file>.js`

### Ví dụ 1 — process.nextTick vs Promise (Node.js only)

```ts
// Chạy: bun run index.ts

console.log('=== Test: nextTick vs Promise ===');

setTimeout(() => console.log('[macrotask] setTimeout'), 0);

Promise.resolve().then(() => console.log('[microtask-2] Promise.then'));

process.nextTick(() => console.log('[microtask-1] process.nextTick'));

console.log('[sync] synchronous');

/*
OUTPUT (Node.js / Bun):
[sync] synchronous
[microtask-1] process.nextTick       ← nextTick trước Promise!
[microtask-2] Promise.then
[macrotask] setTimeout

So sánh với Browser (không có process.nextTick):
[sync] synchronous
[microtask] Promise.then
[macrotask] setTimeout
*/
```

---

### Ví dụ 2 — nextTick queue được ưu tiên hoàn toàn trước Promise

```ts
console.log('=== Test: nextTick queue drain ===');

Promise.resolve()
  .then(() => {
    console.log('[promise] 1');
    process.nextTick(() => console.log('[nextTick sinh bởi promise] 2'));
  })
  .then(() => {
    console.log('[promise] 3');
  });

process.nextTick(() => {
  console.log('[nextTick] A');
  process.nextTick(() => console.log('[nextTick sinh bởi nextTick] B'));
});

/*
OUTPUT:
[nextTick] A
[nextTick sinh bởi nextTick] B   ← nextTick queue drain trước khi sang Promise
[promise] 1
[nextTick sinh bởi promise] 2    ← nextTick được sinh ra → drain ngay trước Promise tiếp
[promise] 3

Giải thích:
- nextTick queue drain hoàn toàn (A, B) trước khi Promise queue chạy
- Sau mỗi Promise callback, nextTick queue lại được check trước Promise tiếp theo
*/
```

---

### Ví dụ 3 — setImmediate vs setTimeout TRONG I/O callback

```ts
import { readFile } from 'fs';

console.log('=== Test: setImmediate vs setTimeout trong I/O ===');

// Ngoài I/O: thứ tự không đảm bảo (phụ thuộc OS timing)
setTimeout(() => console.log('[ngoài I/O] setTimeout'), 0);
setImmediate(() => console.log('[ngoài I/O] setImmediate'));
// Output: KHÔNG xác định — có thể là setTimeout trước hoặc setImmediate trước

// Trong I/O callback: setImmediate LUÔN LUÔN trước setTimeout
readFile(__filename, () => {
  setTimeout(() => console.log('[trong I/O] setTimeout'), 0);
  setImmediate(() => console.log('[trong I/O] setImmediate'));
  // Output: LUÔN LUÔN setImmediate trước setTimeout
});

/*
Lý do:
Khi trong I/O callback, ta đang ở POLL phase.
Sau khi I/O callback xong:
- Event Loop thấy có setImmediate → chuyển sang CHECK phase trước
- Sau CHECK phase mới quay vòng về TIMERS phase
→ setImmediate luôn chạy trước setTimeout trong I/O context
*/
```

---

### Ví dụ 4 — Microtask drain sau MỖI phase (khác Browser)

```ts
console.log('=== Test: Microtask drain sau mỗi phase ===');

// Đây là điểm khác biệt lớn nhất so với Browser
// Browser: microtask drain sau mỗi MACROTASK
// Node.js: microtask drain sau mỗi PHASE (chi tiết hơn nhiều)

setImmediate(() => {
  console.log('[CHECK phase] setImmediate 1');
  process.nextTick(() => console.log('  [nextTick sinh bởi setImmediate 1]'));
  Promise.resolve().then(() => console.log('  [promise sinh bởi setImmediate 1]'));
});

setImmediate(() => {
  console.log('[CHECK phase] setImmediate 2');
});

/*
OUTPUT:
[CHECK phase] setImmediate 1
  [nextTick sinh bởi setImmediate 1]     ← microtask drain ngay!
  [promise sinh bởi setImmediate 1]      ← trước setImmediate 2
[CHECK phase] setImmediate 2

Nếu là Browser (giả sử):
setImmediate 1
setImmediate 2
nextTick...       ← sau khi cả 2 macrotask chạy xong (không có trong browser anyway)

Node.js drain microtask GIỮA các callbacks trong cùng phase!
*/
```

---

### Ví dụ 5 — Câu hỏi phỏng vấn kinh điển Node.js

```ts
console.log('=== Classic Node.js Interview Question ===');

async function main() {
  console.log('1: main start');

  await Promise.resolve();
  console.log('2: after first await');

  await new Promise(resolve => setImmediate(resolve));
  console.log('3: after setImmediate await');

  await new Promise(resolve => setTimeout(resolve, 0));
  console.log('4: after setTimeout await');
}

process.nextTick(() => console.log('5: nextTick'));
main();
console.log('6: sync after main()');

/*
OUTPUT:
1: main start         ← sync trong main()
6: sync after main()  ← sync sau khi main() tạm dừng tại await
5: nextTick           ← nextTick queue
2: after first await  ← Promise microtask (await Promise.resolve())
                      ← main tạm dừng tại setImmediate
3: after setImmediate await  ← CHECK phase
                      ← main tạm dừng tại setTimeout
4: after setTimeout await    ← TIMERS phase

Giải thích: await bọc trong Promise(resolve => setImmediate(resolve))
→ resume function khi setImmediate chạy → CHECK phase
→ tức là function "ngủ" đến CHECK phase mới tiếp tục
*/
```

---

## PHẦN 7 — GOTCHAS & PITFALLS (SO SÁNH BROWSER vs NODE)

### Gotcha 1 — process.nextTick có thể starve I/O

```ts
// BUG: recursive nextTick → I/O callbacks không bao giờ chạy
// (Giống microtask starvation nhưng tệ hơn vì nextTick ưu tiên cao hơn)

function recursiveNextTick() {
  process.nextTick(() => {
    console.log('nextTick chạy...');
    recursiveNextTick(); // tạo nextTick mới liên tục
  });
}

// setTimeout sẽ KHÔNG BAO GIỜ chạy nếu nextTick không dừng
setTimeout(() => console.log('Tôi sẽ không bao giờ in'), 0);
recursiveNextTick();

/*
So sánh:
Browser: không có nextTick, nhưng recursive Promise cũng starve macrotask
Node.js: nextTick starve cả Promise.then và toàn bộ macrotask phases

FIX: dùng setImmediate thay nextTick cho recursive tasks
function recursiveSafe() {
  setImmediate(() => {
    console.log('setImmediate chạy...');
    recursiveSafe(); // OK — mỗi vòng Event Loop 1 lần, không starve
  });
}
*/
```

---

### Gotcha 2 — setTimeout(fn, 0) KHÔNG phải "ngay sau sync code" trong Node.js

```ts
// Browser: setTimeout(fn, 0) delay tối thiểu ~4ms (HTML spec)
// Node.js: setTimeout(fn, 0) delay tối thiểu ~1ms, nhưng quan trọng hơn:
//          phụ thuộc vào thời điểm Event Loop vào TIMERS phase

// Tình huống: setTimeout(fn, 0) có thể chạy SAU setImmediate
// dù setImmediate được đăng ký SAU setTimeout

setTimeout(() => console.log('A: setTimeout'), 0);
setImmediate(() => console.log('B: setImmediate'));

/*
Output không đảm bảo (chạy nhiều lần có thể ra khác nhau):
- Lần 1: A: setTimeout → B: setImmediate
- Lần 2: B: setImmediate → A: setTimeout

Lý do: khi Node.js khởi động, nếu chưa vào TIMERS phase
→ setTimeout chưa "expired" từ góc nhìn của Event Loop
→ skip TIMERS, vào POLL (rỗng), vào CHECK (setImmediate)
→ setImmediate trước

FIX: Nếu cần đảm bảo thứ tự → đặt trong I/O callback (xem Ví dụ 3)
*/
```

---

### Gotcha 3 — await trong vòng lặp: sequential vs parallel

```ts
// BUG: sequential thay vì song song (cả Browser lẫn Node.js)
async function fetchAllBad(ids: string[]) {
  const results = [];
  for (const id of ids) {
    const data = await fetch(`/api/${id}`); // đợi từng cái
    results.push(await data.json());
  }
  return results;
  // 5 requests × 200ms = 1000ms
}

// GOOD: song song với Promise.all
async function fetchAllGood(ids: string[]) {
  const promises = ids.map(id =>
    fetch(`/api/${id}`).then(r => r.json())
  );
  return Promise.all(promises);
  // Tất cả chạy cùng lúc → 200ms tổng
}

// GOOD: song song với giới hạn concurrency (không spam server)
async function fetchWithLimit(ids: string[], limit = 5) {
  const results: unknown[] = [];
  for (let i = 0; i < ids.length; i += limit) {
    const batch = ids.slice(i, i + limit);
    const batchResults = await Promise.all(
      batch.map(id => fetch(`/api/${id}`).then(r => r.json()))
    );
    results.push(...batchResults);
  }
  return results;
}
```

---

### Gotcha 4 — Unhandled rejection: hành vi KHÁC nhau

```ts
// Browser: console.error + window.onunhandledrejection event, KHÔNG crash

// Node.js < v15: chỉ warning, KHÔNG crash
// Node.js >= v15: CRASH process với exit code 1!

// BAD:
async function riskyOperation() {
  throw new Error('Something went wrong');
}
riskyOperation(); // không await, không catch → crash Node v15+

// GOOD 1: always catch
riskyOperation().catch(err => console.error(err));

// GOOD 2: await với try/catch
try {
  await riskyOperation();
} catch (err) {
  console.error(err);
}

// GOOD 3: global handler (last resort)
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection:', reason);
  // Có thể log rồi graceful shutdown
  process.exit(1);
});

/*
So sánh hành vi:
┌─────────────────────────┬──────────────────┬──────────────────────────────┐
│                         │    BROWSER       │    NODE.JS                   │
├─────────────────────────┼──────────────────┼──────────────────────────────┤
│ Unhandled rejection     │ console.error    │ v14-: warning                │
│                         │ + event fired    │ v15+: CRASH (exit code 1)    │
├─────────────────────────┼──────────────────┼──────────────────────────────┤
│ Global handler          │ window           │ process                      │
│                         │ .onunhandled     │ .on('unhandledRejection')    │
│                         │ Rejection        │                              │
└─────────────────────────┴──────────────────┴──────────────────────────────┘
*/
```

---

### Gotcha 5 — setTimeout delay KHÔNG chính xác khi có heavy microtask

```ts
// Browser lẫn Node.js đều bị ảnh hưởng
// Dù setTimeout(fn, 100), fn sẽ chạy MUỘN HƠN 100ms
// nếu microtask queue hoặc sync code block Event Loop

console.time('actual delay');

setTimeout(() => {
  console.timeEnd('actual delay'); // Sẽ > 100ms!
}, 100);

// Block bằng heavy microtask chain
function spawnMicrotasks(n: number): Promise<void> {
  if (n === 0) return Promise.resolve();
  return Promise.resolve().then(() => spawnMicrotasks(n - 1));
}

await spawnMicrotasks(1_000_000); // block Event Loop 200ms+

/*
Output: actual delay: ~300ms (100ms delay + 200ms microtask block)

Bài học: setTimeout là "tối thiểu", không phải "chính xác"
→ Không dùng setTimeout cho timing-critical code
→ Dùng performance.now() để đo thời gian thực tế
*/
```

---

### Gotcha 6 — Error trong Promise chain bị "nuốt" im lặng

```ts
// Browser lẫn Node.js
// BUG: error bị nuốt hoàn toàn, không có log nào

async function processData(data: unknown[]) {
  return data.map(async item => {
    // Nếu processItem throw → Promise.reject → KHÔNG ai catch!
    return await processItem(item);
  });
  // map trả về Array<Promise>, không phải Promise<Array>
  // Không có await ở đây → errors bị nuốt
}

// FIX:
async function processDataFixed(data: unknown[]) {
  return Promise.all(
    data.map(item =>
      processItem(item).catch(err => {
        console.error(`Failed for item:`, err);
        return null; // hoặc throw để fail toàn bộ
      })
    )
  );
}

// Dùng Promise.allSettled nếu muốn biết cái nào lỗi
async function processDataResilient(data: unknown[]) {
  const results = await Promise.allSettled(data.map(processItem));
  const succeeded = results
    .filter((r): r is PromiseFulfilledResult<unknown> => r.status === 'fulfilled')
    .map(r => r.value);
  const failed = results
    .filter((r): r is PromiseRejectedResult => r.status === 'rejected')
    .map(r => r.reason);

  if (failed.length) console.error(`${failed.length} items failed`);
  return succeeded;
}
```

---

### Gotcha 7 — setInterval drift: khoảng cách không đều

```ts
// setInterval KHÔNG đảm bảo interval đều nhau
// Nếu callback mất 80ms và interval là 100ms → gap thực tế là 100+80=180ms

// BUG: drift tích lũy theo thời gian
let count = 0;
const start = Date.now();
setInterval(() => {
  count++;
  const expected = count * 1000;
  const actual = Date.now() - start;
  console.log(`Expected: ${expected}ms, Actual: ${actual}ms, Drift: ${actual - expected}ms`);
  // Drift ngày càng tăng!
  heavyWork(); // block 200ms
}, 1000);

// FIX: self-correcting timer
function setAccurateInterval(fn: () => void, interval: number) {
  let expected = Date.now() + interval;

  function tick() {
    const drift = Date.now() - expected;
    fn();
    expected += interval;
    // Điều chỉnh delay theo drift
    setTimeout(tick, Math.max(0, interval - drift));
  }

  setTimeout(tick, interval);
}
```

---

## PHẦN 8 — BẢNG TỔNG HỢP: CÁI GÌ CHẠY KHI NÀO

### Node.js — Thứ tự ưu tiên đầy đủ

```
Khi Call Stack rỗng, Node.js làm theo thứ tự:

1. process.nextTick queue          [MICROTASK - cao nhất]
   → Làm trống hoàn toàn (kể cả nextTick mới)

2. Promise microtask queue         [MICROTASK]
   → Làm trống hoàn toàn
   → Sau mỗi Promise callback, check nextTick queue trước

3. TIMERS phase                    [MACROTASK]
   → setTimeout / setInterval đã hết hạn
   → Sau mỗi callback: drain nextTick + Promise

4. PENDING CALLBACKS phase
   → I/O errors từ vòng trước
   → Sau mỗi callback: drain nextTick + Promise

5. POLL phase                      [MACROTASK - core]
   → I/O callbacks (fs, network, child_process)
   → Block chờ nếu queue rỗng và không có pending timers
   → Sau mỗi callback: drain nextTick + Promise

6. CHECK phase                     [MACROTASK]
   → setImmediate callbacks
   → Sau mỗi callback: drain nextTick + Promise

7. CLOSE CALLBACKS phase
   → socket.on('close'), stream close events
```

### Browser — Thứ tự ưu tiên

```
Khi Call Stack rỗng:

1. Microtask queue
   → Promise .then, await, queueMicrotask, MutationObserver
   → Làm trống hoàn toàn

2. Render (nếu cần, ~60fps)
   → requestAnimationFrame
   → Layout, Paint, Composite

3. 1 Macrotask
   → setTimeout, setInterval, DOM events, MessageChannel
   → Sau khi xong: quay lại bước 1
```

---

## PHẦN 9 — KHI NÀO DÙNG GÌ TRONG NODE.JS

```
Muốn chạy "ngay sau sync code hiện tại, trước tất cả I/O":
→ process.nextTick(fn)
→ Dùng cho: emit events, error callbacks, cleanup sau sync setup
→ CẢNH BÁO: không dùng đệ quy

Muốn chạy "sau microtask, trong cùng iteration của Event Loop":
→ Promise.resolve().then(fn) hoặc queueMicrotask(fn)
→ Dùng cho: defer work nhỏ, batching updates

Muốn chạy "sau tất cả I/O callbacks hiện tại":
→ setImmediate(fn)
→ Dùng cho: break up CPU-intensive tasks, sau I/O
→ GẦN GIỐNG requestAnimationFrame trong browser

Muốn chạy "sau N milliseconds":
→ setTimeout(fn, N)
→ Delay là MINIMUM, không phải exact
→ setTimeout(fn, 0) ≈ setImmediate(fn) nhưng không đảm bảo thứ tự

Muốn chạy SONG SONG:
→ Promise.all([...]) — không giới hạn
→ Chạy theo batch nếu cần giới hạn concurrency
```

---

## PHẦN 10 — BÀI TẬP TỰ KIỂM TRA (CÓ ĐÁP ÁN)

### Bài 1 — Output là gì? (Node.js)

```ts
process.nextTick(() => console.log('A'));
Promise.resolve().then(() => console.log('B'));
setImmediate(() => console.log('C'));
setTimeout(() => console.log('D'), 0);
console.log('E');

// Đáp án: E → A → B → D hoặc C → C hoặc D
// (D và C không đảm bảo thứ tự ngoài I/O callback)
// Chắc chắn: E trước, A trước B, A+B trước C+D
```

---

### Bài 2 — Trong I/O callback, output là gì?

```ts
import { readFile } from 'fs';

readFile('/etc/hosts', () => {
  process.nextTick(() => console.log('nextTick'));
  Promise.resolve().then(() => console.log('Promise'));
  setImmediate(() => console.log('setImmediate'));
  setTimeout(() => console.log('setTimeout'), 0);
});

// Đáp án: nextTick → Promise → setImmediate → setTimeout
// Trong I/O callback: setImmediate LUÔN trước setTimeout
```

---

### Bài 3 — Tìm bug và fix

```ts
// Code này có vấn đề gì?
async function processItems(items: string[]) {
  const results: string[] = [];
  items.forEach(async (item) => {  // BUG!
    const result = await processItem(item);
    results.push(result);
  });
  return results; // Luôn trả về [] rỗng!
}

// Tại sao bug?
// forEach không await Promise → hàm return [] trước khi bất kỳ item nào xong

// Fix:
async function processItemsFixed(items: string[]) {
  return Promise.all(items.map(item => processItem(item)));
}

// Hoặc nếu cần sequential:
async function processItemsSequential(items: string[]) {
  const results: string[] = [];
  for (const item of items) {
    results.push(await processItem(item)); // for...of await được
  }
  return results;
}
```

---

## TỔNG KẾT — 7 Điều Phân Biệt Node.js vs Browser

```
1. Node.js có process.nextTick — ưu tiên CAO HƠN Promise.then
   Browser không có

2. Node.js macrotask = 6 phases vòng tròn
   Browser macrotask = 1 queue đơn giản

3. Node.js drain microtask sau MỖI PHASE (và giữa callbacks trong phase)
   Browser drain microtask sau mỗi macrotask

4. setImmediate chỉ có trong Node.js (CHECK phase)
   Trong I/O callback: setImmediate LUÔN trước setTimeout

5. setTimeout(fn, 0) ngoài I/O: thứ tự so với setImmediate KHÔNG đảm bảo
   Trong I/O: setImmediate luôn trước

6. Node.js v15+: unhandled rejection = CRASH
   Browser: chỉ warning

7. Node.js POLL phase block chờ I/O — đây là lý do Node.js efficient với I/O
   Browser không có khái niệm này
```
