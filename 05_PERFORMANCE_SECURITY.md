# Performance, Security & Authentication — Lý thuyết sâu + Ví dụ 3 cấp độ

---

## 1. CORE WEB VITALS & PERFORMANCE

### Lý thuyết & Bản chất
**Core Web Vitals** là 3 chỉ số Google dùng để đo **user experience**:

| Metric | Đo gì | Tốt | Nguyên nhân phổ biến |
|--------|-------|-----|---------------------|
| **LCP** (Largest Contentful Paint) | Thời gian render element lớn nhất | < 2.5s | Ảnh lớn không optimize, font chặn render, TTFB cao |
| **INP** (Interaction to Next Paint) | Độ trễ response với user input | < 200ms | Long JavaScript tasks, blocking main thread |
| **CLS** (Cumulative Layout Shift) | Layout bị nhảy ngoài ý muốn | < 0.1 | Ảnh không có kích thước, nội dung inject động, font swap |

**Critical Rendering Path:**
```
HTML parse → DOM → CSSOM → Render Tree → Layout → Paint → Composite
            ↑                ↑
    JS có thể block HTML    CSS block Render Tree
```

---

### Ví dụ 1 — Cơ bản: Quick wins
```tsx
// 1. LCP: Preload hero image
// app/layout.tsx
export default function RootLayout({ children }) {
  return (
    <html>
      <head>
        {/* Preload → browser biết sớm cần load ảnh này */}
        <link rel="preload" href="/hero.webp" as="image" />
        {/* Preconnect → thiết lập connection trước khi cần */}
        <link rel="preconnect" href="https://fonts.googleapis.com" />
      </head>
      <body>{children}</body>
    </html>
  );
}

// 2. LCP: Dùng Next.js Image component
import Image from 'next/image';

function HeroSection() {
  return (
    <Image
      src="/hero.webp"
      alt="Hero"
      width={1200}
      height={600}
      priority          // → preload, không lazy load
      placeholder="blur" // → hiện blur trước khi load xong
      blurDataURL="data:image/jpeg;base64,..."
    />
  );
}

// 3. CLS: Luôn đặt kích thước cho ảnh và dynamic content
// BAD: không biết kích thước → layout shift khi load xong
<img src="/photo.jpg" />

// GOOD: browser biết trước reserved space
<img src="/photo.jpg" width={400} height={300} />
// hoặc dùng aspect-ratio CSS
<div style={{ aspectRatio: '4/3' }}>
  <img src="/photo.jpg" style={{ width: '100%', height: '100%', objectFit: 'cover' }} />
</div>
```

---

### Ví dụ 2 — Trung cấp: Code splitting + Bundle optimization
```tsx
// Code splitting với dynamic import
import dynamic from 'next/dynamic';

// Không load cho đến khi cần — giảm initial bundle
const HeavyChart = dynamic(() => import('@/components/Chart'), {
  loading: () => <div className="h-64 bg-gray-100 animate-pulse rounded" />,
  ssr: false, // Nếu dùng browser-only APIs (window, document...)
});

const RichTextEditor = dynamic(() => import('@/components/Editor'), {
  ssr: false,
});

// Conditional import — component đặc biệt không luôn dùng
function ProductPage({ hasVariants }) {
  const VariantSelector = hasVariants
    ? dynamic(() => import('./VariantSelector'))
    : null;

  return (
    <div>
      <ProductInfo />
      {VariantSelector && <VariantSelector />}
    </div>
  );
}

// Bundle analysis — thêm vào next.config.js
// const withBundleAnalyzer = require('@next/bundle-analyzer')({ enabled: process.env.ANALYZE === 'true' });
// npm run build (với ANALYZE=true) → visual map của bundle
```

---

### Ví dụ 3 — Nâng cao: Virtualization + Web Workers
```tsx
// Virtualization: chỉ render items đang visible trong viewport
// Dùng khi list > 100 items
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }) {
  const parentRef = React.useRef(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 60, // estimated row height
    overscan: 5, // render thêm N rows ngoài viewport
  });

  return (
    <div ref={parentRef} style={{ height: '500px', overflow: 'auto' }}>
      {/* Container với tổng height → scrollbar đúng */}
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.index}
            style={{
              position: 'absolute',
              top: virtualRow.start,
              height: virtualRow.size,
              width: '100%',
            }}
          >
            <ListItem item={items[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}

// Web Worker: heavy computation không block UI thread
// worker.ts
self.onmessage = function(e) {
  const result = heavyComputation(e.data);
  self.postMessage(result);
};

// Component
function useWorker(workerFn) {
  const workerRef = React.useRef(null);
  const [result, setResult] = React.useState(null);

  React.useEffect(() => {
    workerRef.current = new Worker(new URL('./worker.ts', import.meta.url));
    workerRef.current.onmessage = (e) => setResult(e.data);
    return () => workerRef.current.terminate();
  }, []);

  const runTask = React.useCallback((data) => {
    workerRef.current?.postMessage(data);
  }, []);

  return { result, runTask };
}
```

---

## 2. SECURITY — XSS, CSRF, SECURE STORAGE

### Lý thuyết & Bản chất

**XSS (Cross-Site Scripting):** Attacker inject script vào website → script chạy với quyền của website → đọc cookies, localStorage, thực hiện actions thay user.

**3 loại XSS:**
- **Stored XSS**: Script lưu vào DB (comment, username...) → hiện lên mọi user xem trang đó
- **Reflected XSS**: Script trong URL → server echo ra HTML → nạn nhân click link
- **DOM-based XSS**: Script không qua server, xử lý hoàn toàn ở client-side JS

**CSRF (Cross-Site Request Forgery):** Attacker trick browser của nạn nhân gửi request đến site đang đăng nhập → server nhận thấy cookie hợp lệ → thực hiện action.

---

### Ví dụ 1 — Cơ bản: XSS prevention
```tsx
// React tự động escape output → an toàn
const userInput = '<script>alert("xss")</script>';
return <div>{userInput}</div>; // Render: &lt;script&gt;... (escaped HTML)

// NGUY HIỂM: dangerouslySetInnerHTML với untrusted data
function Comment({ content }) {
  // BAD: nếu content chứa script → XSS
  return <div dangerouslySetInnerHTML={{ __html: content }} />;

  // GOOD: sanitize trước khi render (dùng DOMPurify)
  const sanitized = DOMPurify.sanitize(content, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p'],
    ALLOWED_ATTR: ['href'],
  });
  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />;
}

// Content Security Policy (CSP) — ngăn inline scripts
// next.config.js
const securityHeaders = [
  {
    key: 'Content-Security-Policy',
    value: [
      "default-src 'self'",
      "script-src 'self' 'nonce-{NONCE}'", // chỉ allow scripts có nonce
      "style-src 'self' 'unsafe-inline'",
      "img-src 'self' data: https:",
      "connect-src 'self' https://api.example.com",
    ].join('; '),
  },
];
```

---

### Ví dụ 2 — Trung cấp: CSRF Protection
```tsx
// CSRF attack scenario:
// 1. User đăng nhập bank.com → cookie set
// 2. User visit evil.com
// 3. evil.com có form ẩn submit đến bank.com/transfer
// 4. Browser tự gửi cookie của bank.com → transfer thực hiện!

// Phòng chống 1: SameSite Cookie
// response.cookie('session', token, {
//   sameSite: 'Strict', // cookie KHÔNG gửi trong cross-site requests
//   httpOnly: true,
//   secure: true,
// });
// 'Lax': gửi khi navigate (link click) nhưng không khi form POST, fetch

// Phòng chống 2: CSRF Token
// app/api/csrf/route.ts
import { randomBytes } from 'crypto';

export async function GET() {
  const token = randomBytes(32).toString('hex');
  // Lưu token trong session
  return Response.json({ token }, {
    headers: {
      'Set-Cookie': `csrf-token=${token}; SameSite=Strict; HttpOnly`,
    },
  });
}

// Client gửi token trong header (form không thể set custom headers cross-site)
async function transferMoney(data) {
  const { token } = await fetch('/api/csrf').then(r => r.json());
  await fetch('/api/transfer', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-CSRF-Token': token, // server verify token này
    },
    body: JSON.stringify(data),
  });
}

// Server verify:
// const headerToken = req.headers['x-csrf-token'];
// const cookieToken = req.cookies['csrf-token'];
// if (headerToken !== cookieToken) return 403;
```

---

### Ví dụ 3 — Nâng cao: Secure token storage + Auth flow
```
Câu hỏi phổ biến: "Lưu JWT ở đâu?"

Phân tích:
┌──────────────┬─────────────────────────┬──────────────────────────────┐
│ Storage      │ XSS                     │ CSRF                         │
├──────────────┼─────────────────────────┼──────────────────────────────┤
│ localStorage │ Vulnerable (JS đọc được)│ Safe (phải set header)       │
│ sessionStorage│ Vulnerable             │ Safe                         │
│ Memory (JS)  │ Vulnerable (page reload mất)│ Safe                    │
│ HttpOnly     │ SAFE (JS không đọc được)│ Vulnerable (cần SameSite)    │
│ Cookie       │                         │                              │
└──────────────┴─────────────────────────┴──────────────────────────────┘

Best practice: HttpOnly Cookie + SameSite=Strict/Lax + CSRF double-submit
```

```tsx
// Secure auth flow implementation
// lib/auth-client.ts

class AuthClient {
  // Access token ngắn hạn → lưu trong memory (không persist, mất khi reload)
  private accessToken: string | null = null;

  async login(email: string, password: string) {
    const res = await fetch('/api/auth/login', {
      method: 'POST',
      body: JSON.stringify({ email, password }),
      headers: { 'Content-Type': 'application/json' },
    });

    const data = await res.json();
    // Server set refresh token trong HttpOnly cookie (không trả về body)
    // Access token ngắn hạn (15 min) trả về body → lưu in-memory
    this.accessToken = data.accessToken;
    return data.user;
  }

  async getAccessToken(): Promise<string | null> {
    if (this.accessToken && !this.isExpired(this.accessToken)) {
      return this.accessToken;
    }
    // Refresh: HttpOnly cookie tự động gửi
    try {
      const res = await fetch('/api/auth/refresh', { method: 'POST' });
      const data = await res.json();
      this.accessToken = data.accessToken;
      return this.accessToken;
    } catch {
      return null;
    }
  }

  async logout() {
    this.accessToken = null;
    await fetch('/api/auth/logout', { method: 'POST' }); // server clear cookie
  }

  private isExpired(token: string): boolean {
    const payload = JSON.parse(atob(token.split('.')[1]));
    return payload.exp * 1000 < Date.now();
  }
}

// Axios interceptor tự động refresh
axiosInstance.interceptors.response.use(
  response => response,
  async error => {
    const originalRequest = error.config;
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;
      const token = await authClient.getAccessToken(); // try refresh
      if (token) {
        originalRequest.headers.Authorization = `Bearer ${token}`;
        return axiosInstance(originalRequest);
      }
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

---

## 3. REACT QUERY / TANSTACK QUERY

### Lý thuyết & Bản chất
React Query là **server state management library**. "Server state" khác client state ở chỗ:
- Lưu ở xa (DB)
- Cần async API để đọc/ghi
- Có thể thay đổi bởi người khác (stale)
- Cần cache, background refresh, retry

**Cơ chế:**
- Query được identify bởi `queryKey` (array, deeply compared)
- `staleTime`: thời gian data được coi là "tươi" (không fetch lại)
- `gcTime` (cacheTime): thời gian data được giữ trong cache sau khi không ai dùng
- Background refetch: khi window focus lại, network reconnect, `staleTime` hết

---

### Ví dụ 1 — Cơ bản: useQuery + useMutation
```tsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// Setup
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 phút
      retry: 3,                  // retry 3 lần khi lỗi
      retryDelay: attemptIndex => Math.min(1000 * 2 ** attemptIndex, 30000), // exponential backoff
    },
  },
});

// Read data
function UserProfile({ userId }) {
  const {
    data: user,
    isLoading,    // true lần đầu, không có cached data
    isFetching,   // true bất kỳ khi nào đang fetch (bao gồm background)
    isError,
    error,
    refetch,
  } = useQuery({
    queryKey: ['user', userId],   // unique key — array, reactive
    queryFn: () => fetchUser(userId),
    staleTime: 60_000, // 1 phút — không refetch nếu data < 1 phút tuổi
    enabled: !!userId, // chỉ fetch khi userId tồn tại
  });

  if (isLoading) return <Skeleton />;
  if (isError) return <div>Error: {error.message}</div>;
  return <div>{user.name}</div>;
}

// Write data
function UpdateProfileForm({ user }) {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: (data) => updateUser(user.id, data),
    onSuccess: (updatedUser) => {
      // Cập nhật cache ngay thay vì refetch
      queryClient.setQueryData(['user', user.id], updatedUser);
      // Hoặc invalidate để refetch fresh data
      queryClient.invalidateQueries({ queryKey: ['user', user.id] });
    },
    onError: (error) => {
      toast.error(error.message);
    },
  });

  return (
    <form onSubmit={e => {
      e.preventDefault();
      mutation.mutate(Object.fromEntries(new FormData(e.target)));
    }}>
      <input name="name" defaultValue={user.name} />
      <button disabled={mutation.isPending}>
        {mutation.isPending ? 'Saving...' : 'Save'}
      </button>
    </form>
  );
}
```

---

### Ví dụ 2 — Trung cấp: Pagination + Infinite scroll
```tsx
// Pagination
function PaginatedList({ pageSize = 20 }) {
  const [page, setPage] = React.useState(1);

  const { data, isPlaceholderData } = useQuery({
    queryKey: ['products', page, pageSize],
    queryFn: () => fetchProducts({ page, pageSize }),
    placeholderData: keepPreviousData, // giữ data cũ khi page thay đổi → không flicker
  });

  return (
    <div>
      <div style={{ opacity: isPlaceholderData ? 0.7 : 1 }}>
        {data?.products.map(p => <ProductCard key={p.id} product={p} />)}
      </div>
      <div>
        <button disabled={page === 1} onClick={() => setPage(p => p - 1)}>Prev</button>
        <span>Page {page} of {data?.totalPages}</span>
        <button disabled={isPlaceholderData || page === data?.totalPages}
                onClick={() => setPage(p => p + 1)}>Next</button>
      </div>
    </div>
  );
}

// Infinite scroll
import { useInfiniteQuery } from '@tanstack/react-query';

function InfiniteList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['products', 'infinite'],
    queryFn: ({ pageParam }) => fetchProducts({ cursor: pageParam, limit: 20 }),
    initialPageParam: undefined,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    // nextCursor: null nếu là page cuối
  });

  const allProducts = data?.pages.flatMap(page => page.products) ?? [];

  // Dùng IntersectionObserver để auto-load khi scroll đến cuối
  const [ref, isVisible] = useIntersectionObserver({ threshold: 0.5 });
  React.useEffect(() => {
    if (isVisible && hasNextPage && !isFetchingNextPage) fetchNextPage();
  }, [isVisible, hasNextPage]);

  return (
    <div>
      {allProducts.map(p => <ProductCard key={p.id} product={p} />)}
      <div ref={ref}>
        {isFetchingNextPage ? <Spinner /> : hasNextPage ? 'Load more' : 'No more'}
      </div>
    </div>
  );
}
```

---

### Ví dụ 3 — Nâng cao: Optimistic updates + Prefetching
```tsx
// Optimistic updates: cập nhật UI ngay mà không đợi server
function TodoItem({ todo }) {
  const queryClient = useQueryClient();

  const toggleMutation = useMutation({
    mutationFn: () => toggleTodo(todo.id),
    onMutate: async () => {
      // Cancel in-flight queries để tránh race condition
      await queryClient.cancelQueries({ queryKey: ['todos'] });

      // Snapshot giá trị hiện tại để rollback nếu cần
      const previousTodos = queryClient.getQueryData(['todos']);

      // Optimistic update
      queryClient.setQueryData(['todos'], (old) =>
        old.map(t => t.id === todo.id ? { ...t, completed: !t.completed } : t)
      );

      return { previousTodos }; // trả về context cho onError
    },
    onError: (error, _, context) => {
      // Rollback về state trước
      queryClient.setQueryData(['todos'], context.previousTodos);
      toast.error('Failed to update');
    },
    onSettled: () => {
      // Luôn refetch để sync với server
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });

  return (
    <li onClick={() => toggleMutation.mutate()}>
      <input type="checkbox" checked={todo.completed} readOnly />
      {todo.title}
    </li>
  );
}

// Prefetching: load data trước khi user navigate
function ProductList({ products }) {
  const queryClient = useQueryClient();

  return (
    <ul>
      {products.map(product => (
        <li key={product.id}>
          <Link
            href={`/products/${product.id}`}
            // Prefetch khi hover → instant navigation
            onMouseEnter={() => {
              queryClient.prefetchQuery({
                queryKey: ['product', product.id],
                queryFn: () => fetchProduct(product.id),
                staleTime: 60_000,
              });
            }}
          >
            {product.name}
          </Link>
        </li>
      ))}
    </ul>
  );
}
```

---

## TỔNG KẾT

### Performance Checklist
```
LCP (< 2.5s):
☐ Preload hero image (rel="preload")
☐ Dùng Next.js Image với priority={true}
☐ SSR/SSG thay vì CSR cho content quan trọng
☐ Giảm TTFB (CDN, caching, server optimization)
☐ Font loading: font-display: swap

INP (< 200ms):
☐ Code splitting (dynamic import)
☐ Tránh long tasks (chia nhỏ với scheduler)
☐ Virtualize long lists
☐ Debounce/throttle event handlers

CLS (< 0.1):
☐ Khai báo kích thước ảnh (width/height hoặc aspect-ratio)
☐ Không inject content động phía trên existing content
☐ Font: size-adjust, font-display
```

### Security Checklist
```
XSS:
☐ Không dùng dangerouslySetInnerHTML với untrusted data
☐ DOMPurify nếu cần render HTML
☐ Content Security Policy header

CSRF:
☐ SameSite=Strict hoặc Lax cho cookies
☐ CSRF token cho state-changing requests

Token Storage:
☐ Refresh token: HttpOnly cookie
☐ Access token: in-memory (ngắn hạn)
☐ Không lưu sensitive data trong localStorage

General:
☐ HTTPS everywhere
☐ Input validation (client + server)
☐ Rate limiting trên auth endpoints
```
