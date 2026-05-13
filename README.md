# 🎯 Interview Prep — Middle/Senior Fullstack Engineer

> Stack: Next.js · React · TypeScript · Java Spring Boot  
> Tổng thời gian: **20 giờ** · Focus: 70% Frontend · 30% Backend

---

## 📁 Danh sách tài liệu

| # | File | Nội dung chính | Thời gian |
|---|------|----------------|-----------|
| 1 | [01_JS_TYPESCRIPT.md](01_JS_TYPESCRIPT.md) | Closure, Event Loop, `this`, Prototype, Promise, Generic, Utility Types | 2h |
| 2 | [02_REACT_HOOKS.md](02_REACT_HOOKS.md) | Virtual DOM, Reconciliation, useState/useReducer, useEffect, useMemo/useCallback, useRef, Custom Hooks | 2h |
| 2b | [02b_REACT_LIFECYCLE_HOOKS_REACT19.md](02b_REACT_LIFECYCLE_HOOKS_REACT19.md) | Lifecycle class + functional, **19 hooks đầy đủ**, React 19 mới nhất | 2h |
| 3 | [03_NEXTJS.md](03_NEXTJS.md) | CSR/SSR/SSG/ISR, App Router, Server Components, Caching, Middleware, NextAuth | 2.5h |
| 4 | [04_STATE_MANAGEMENT.md](04_STATE_MANAGEMENT.md) | Redux Toolkit + RTK Query, Zustand, Context API | 1.5h |
| 5 | [05_PERFORMANCE_SECURITY.md](05_PERFORMANCE_SECURITY.md) | Core Web Vitals, Code Splitting, XSS/CSRF, JWT, React Query | 2h |
| 6 | [06_SPRING_BOOT.md](06_SPRING_BOOT.md) | IoC/DI, REST API, JPA, N+1, @Transactional, Spring Security JWT | 3h |
| 7 | [07_INTERVIEW_QA.md](07_INTERVIEW_QA.md) | 15+ Q&A chuẩn, Coding challenges, Behavioral questions | 3h |
| 8 | [08_JAVA_CORE.md](08_JAVA_CORE.md) | OOP, Generics, Collections, Streams, Concurrency, Modern Java 8→21, Design Patterns | 2h |

---

## 🗓 Lịch học 20 giờ

> Mỗi session **2–2.5h**. Sau mỗi session: 15 phút tự giải thích lại to thành lời (Feynman technique).

### Ngày 1 — Nền tảng JS & Java (4h)
| Thứ tự | File | Nội dung | Thời gian |
|--------|------|----------|-----------|
| 1 | `01_JS_TYPESCRIPT.md` | Closure → Event Loop → `this` → Promise → Generics | 2h |
| 2 | `08_JAVA_CORE.md` — Phần 1,2,3 | OOP · Generics · Collections | 2h |

**Mục tiêu cuối ngày:**
- [ ] Giải thích được Event Loop với output question
- [ ] Viết được closure + memoize từ đầu
- [ ] Phân biệt `interface` vs `abstract class` trong Java

---

### Ngày 2 — React Core (4h)
| Thứ tự | File | Nội dung | Thời gian |
|--------|------|----------|-----------|
| 3 | `02_REACT_HOOKS.md` | Virtual DOM · Hooks · Custom Hooks | 2h |
| 4 | `02b_REACT_LIFECYCLE_HOOKS_REACT19.md` | Lifecycle · Full hooks · React 19 | 2h |

**Mục tiêu cuối ngày:**
- [ ] Vẽ được lifecycle diagram class + functional
- [ ] Giải thích stale closure trong useEffect
- [ ] Nêu được 3 React 19 features mới nhất với ví dụ

---

### Ngày 3 — Next.js & State (4h)
| Thứ tự | File | Nội dung | Thời gian |
|--------|------|----------|-----------|
| 5 | `03_NEXTJS.md` | 4 Rendering Strategies · App Router · Caching | 2.5h |
| 6 | `04_STATE_MANAGEMENT.md` | Redux Toolkit · RTK Query · Zustand | 1.5h |

**Mục tiêu cuối ngày:**
- [ ] Quyết định đúng strategy cho từng loại trang không cần nhìn tài liệu
- [ ] Giải thích Server Component vs Client Component
- [ ] Viết được Redux slice với createAsyncThunk

---

### Ngày 4 — Backend + Performance (4h)
| Thứ tự | File | Nội dung | Thời gian |
|--------|------|----------|-----------|
| 7 | `05_PERFORMANCE_SECURITY.md` | Core Web Vitals · XSS/CSRF · JWT · React Query | 2h |
| 8 | `06_SPRING_BOOT.md` | IoC/DI · REST · JPA · Security · N+1 | 2h |

**Mục tiêu cuối ngày:**
- [ ] Giải thích 3 Core Web Vitals và cách tối ưu từng loại
- [ ] Mô tả JWT refresh token flow hoàn chỉnh
- [ ] Fix được N+1 query với JOIN FETCH / EntityGraph

---

### Ngày 5 — Java nâng cao + Mock Interview (4h)
| Thứ tự | File | Nội dung | Thời gian |
|--------|------|----------|-----------|
| 9 | `08_JAVA_CORE.md` — Phần 4,5,6,7,8 | Streams · Concurrency · Modern Java · Design Patterns | 1.5h |
| 10 | `07_INTERVIEW_QA.md` | Toàn bộ Q&A + Coding challenges | 2.5h |

**Mục tiêu cuối ngày:**
- [ ] Làm được 3 coding challenges không nhìn đáp án
- [ ] Trả lời được toàn bộ 15 câu hỏi trong file Q&A
- [ ] Chuẩn bị 2–3 câu chuyện STAR cho behavioral questions

---

## 📖 Cấu trúc mỗi chủ đề trong tài liệu

Mỗi khái niệm đều có 4 phần:

```
┌─────────────────────────────────────────┐
│  Lý thuyết & Bản chất                  │  ← Giải thích cơ chế bên trong
├─────────────────────────────────────────┤
│  Ví dụ 1 — Cơ bản                      │  ← Hiểu khái niệm thuần túy
├─────────────────────────────────────────┤
│  Ví dụ 2 — Trung cấp                   │  ← Áp dụng thực tế
├─────────────────────────────────────────┤
│  Ví dụ 3 — Nâng cao                    │  ← Production patterns, pitfalls
└─────────────────────────────────────────┘
```

---

## ✅ Checklist tổng trước ngày interview

### Frontend — Bắt buộc nắm chắc
- [ ] Event Loop: microtask vs macrotask, thứ tự output
- [ ] Closure: viết module pattern, memoize không nhìn tài liệu
- [ ] `this` binding: 4 rules, arrow vs regular function
- [ ] TypeScript: Union, Intersection, Utility types, Generic constraints
- [ ] React rendering: khi nào re-render, cách prevent (memo/useCallback/useMemo)
- [ ] Hooks: useState pitfall (functional update), useEffect cleanup + stale closure
- [ ] React 19: `use()`, `useActionState`, `useOptimistic`, Actions
- [ ] Next.js: 4 strategies và khi nào dùng — **câu hỏi gần như 100% bị hỏi**
- [ ] Server Components vs Client Components — phân biệt và pattern
- [ ] Next.js caching: 4 tầng, revalidate, on-demand ISR
- [ ] Redux: dispatch → reducer → store → selector flow
- [ ] Zustand: tại sao cần selector, không thì sao
- [ ] React Query: queryKey, staleTime, invalidateQueries, optimistic update
- [ ] XSS: 3 loại, React escape tự động, khi nào vẫn nguy hiểm
- [ ] CSRF: SameSite cookie, double-submit pattern
- [ ] JWT: access + refresh token, lưu ở đâu và tại sao

### Backend — Nắm đủ để trả lời
- [ ] IoC/DI: 3 cách inject, tại sao constructor injection tốt nhất
- [ ] @Transactional: rollback rules, self-invocation pitfall, private method pitfall
- [ ] N+1: nhận ra trong code, fix bằng JOIN FETCH hoặc EntityGraph
- [ ] JWT filter chain: thứ tự filter trong Spring Security
- [ ] @Cacheable / @CacheEvict: dùng đúng chỗ
- [ ] Dockerfile multi-stage build

### Java Core
- [ ] abstract class vs interface: khi nào dùng gì
- [ ] equals/hashCode contract: tại sao cần override cả hai
- [ ] Stream: filter/map/collect, groupingBy, flatMap
- [ ] CompletableFuture: thenApply, thenCombine, exceptionally
- [ ] Collections: HashMap vs ConcurrentHashMap, khi nào dùng TreeMap

### Coding — Phải tự viết được
- [ ] `debounce(fn, delay)` — từ đầu
- [ ] `throttle(fn, limit)` — từ đầu
- [ ] `Promise.all` implementation
- [ ] `useDebounce` custom hook
- [ ] `useLocalStorage` custom hook
- [ ] Flatten nested array (không dùng `.flat()`)
- [ ] Deep clone (không dùng library)
- [ ] Group by array of objects

---

## 💡 Tips học hiệu quả

### Kỹ thuật Feynman — áp dụng sau mỗi session
Sau khi đọc xong 1 chủ đề, đóng tài liệu lại và **giải thích to thành lời** như thể đang dạy cho người chưa biết. Chỗ nào bị vấp → chỗ đó chưa thực sự hiểu → quay lại đọc phần đó.

### Spaced Repetition
Đừng đọc hết 1 lần rồi thôi. Mỗi ngày dành 10–15 phút review lại ngày hôm trước trước khi học mới.

### Active recall
Thay vì đọc lại → tự hỏi "Closure là gì? → suy nghĩ → kiểm tra". Nhớ lâu hơn đọc thụ động 5 lần.

### Khi bị hỏi câu khó trong interview
1. **Nói ra hướng suy nghĩ** trước — interviewer thường quan tâm process hơn đáp án
2. **Đưa ví dụ cụ thể** — "Điều này giống như khi tôi làm project X..."
3. **Trả lời từ cơ bản đến nâng cao** — bắt đầu đơn giản, mở rộng nếu được hỏi thêm
4. **Không biết thì nói thẳng** — "Tôi không chắc về chi tiết này, nhưng tôi hiểu nguyên lý là..."

---

## 🔗 Thứ tự dependencies khi học

```
JS/TS Core (01)
    ↓
React Hooks (02) ──────── Java Core (08)
    ↓                           ↓
React 19 (02b)          Spring Boot (06)
    ↓                           ↓
Next.js (03)            ────────┘
    ↓                           ↓
State Management (04)   Performance & Security (05)
    └──────────────────────────┘
                ↓
        Mock Interview (07)
```

---

*Chúc bạn interview thành công! 🚀*
