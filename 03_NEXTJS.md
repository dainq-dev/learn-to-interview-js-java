# Next.js — Lý thuyết sâu + Ví dụ 3 cấp độ

---

## 1. RENDERING STRATEGIES

### Lý thuyết & Bản chất

**Câu hỏi cốt lõi:** "HTML này được tạo ra ở đâu và khi nào?"

| Strategy | HTML tạo ra khi | Tạo ở đâu | Data |
|----------|-----------------|-----------|------|
| **CSR** | Client request (sau khi JS load) | Browser | Mỗi visit |
| **SSR** | Mỗi HTTP request | Server | Mỗi request |
| **SSG** | Build time | Server | Tại build |
| **ISR** | Build + background revalidate | Server | Build + update định kỳ |

**TTFB (Time To First Byte):** SSG < ISR < SSR < CSR  
**Data freshness:** CSR > SSR > ISR > SSG  
**Server load:** SSG = 0 < ISR << SSR  

---

### Ví dụ 1 — Cơ bản: 4 strategies trong cùng app
```
E-commerce app cần:
- /           → SSG (landing page, ít thay đổi)
- /products   → ISR (product list, cập nhật hàng giờ)
- /products/[id] → ISR (product detail, cập nhật khi có stock change)
- /cart       → CSR (cần auth, personal data)
- /admin      → SSR (real-time, cần auth check mỗi request)
```

```tsx
// Pages Router

// SSG: getStaticProps (không nhận request, chạy build time)
export async function getStaticProps() {
  const featured = await db.products.findFeatured();
  return { props: { featured } };
  // Không có revalidate → không bao giờ thay đổi sau build
}

// ISR: getStaticProps + revalidate
export async function getStaticProps() {
  const products = await db.products.findAll();
  return {
    props: { products },
    revalidate: 3600, // revalidate sau 3600 giây (1 giờ)
    // Request đầu tiên sau 1h → serve stale, trigger background regen
    // Request sau khi regen xong → serve fresh
  };
}

// SSR: getServerSideProps (nhận request, chạy MỖI request)
export async function getServerSideProps(context) {
  const { req, res, params, query } = context;
  const session = await getSession(req);
  if (!session) return { redirect: { destination: '/login', permanent: false } };
  
  const data = await db.adminStats.findByDate(new Date());
  return { props: { data } };
}

// CSR: chỉ fetch ở client (useEffect / React Query)
function CartPage() {
  const { data: cart } = useQuery({
    queryKey: ['cart'],
    queryFn: fetchCart,
    // không cần disabled — chỉ chạy ở client
  });
  return <CartView cart={cart} />;
}
```

---

### Ví dụ 2 — Trung cấp: App Router — Server Components + Streaming
```tsx
// App Router (Next.js 13+)

// Server Component (default) — chạy trên server
// KHÔNG thể dùng: useState, useEffect, event handlers, browser APIs
// CÓ THỂ: async/await trực tiếp, truy cập DB, đọc env vars, headers/cookies

// app/products/page.tsx
async function ProductsPage({ searchParams }) {
  // Trực tiếp fetch từ DB — không cần API route!
  const products = await db.product.findMany({
    where: searchParams.category
      ? { category: searchParams.category }
      : undefined,
    orderBy: { createdAt: 'desc' },
  });

  return (
    <div>
      <h1>Products</h1>
      {/* Suspense streaming: show skeleton ngay, load data sau */}
      <React.Suspense fallback={<ProductsSkeleton />}>
        <ProductList products={products} />
      </React.Suspense>
    </div>
  );
}

// Parallel data fetching trong Server Component
async function DashboardPage() {
  // Khởi động song song, không await từng cái
  const usersPromise = db.user.count();
  const ordersPromise = db.order.count({ where: { status: 'pending' } });
  const revenuePromise = db.order.aggregate({ _sum: { total: true } });

  // Await tất cả cùng lúc
  const [users, orders, revenue] = await Promise.all([
    usersPromise, ordersPromise, revenuePromise,
  ]);

  return <Dashboard users={users} orders={orders} revenue={revenue.\_sum.total} />;
}
```

---

### Ví dụ 3 — Nâng cao: ISR On-Demand + Cache Tags
```tsx
// On-demand ISR: revalidate khi có event (thay vì theo thời gian)

// app/api/revalidate/route.ts
import { revalidateTag, revalidatePath } from 'next/cache';
import { NextRequest } from 'next/server';

export async function POST(req: NextRequest) {
  const secret = req.nextUrl.searchParams.get('secret');
  if (secret !== process.env.REVALIDATION_SECRET) {
    return Response.json({ error: 'Invalid secret' }, { status: 401 });
  }

  const { tag, path } = await req.json();
  
  if (tag) revalidateTag(tag);        // revalidate tất cả fetch() với tag này
  if (path) revalidatePath(path);     // revalidate một path cụ thể
  
  return Response.json({ revalidated: true });
}

// app/products/page.tsx — gắn tag vào fetch
async function ProductsPage() {
  const products = await fetch('/api/products', {
    next: { tags: ['products'] }, // gắn tag
  }).then(r => r.json());
  
  return <ProductList products={products} />;
}

// Khi product được tạo/cập nhật → gọi webhook → revalidateTag('products')
// → Next.js tự động regenerate trang trong background

// Streaming với nhiều Suspense boundaries
async function Dashboard() {
  return (
    <div className="grid">
      {/* Mỗi section load độc lập — không block nhau */}
      <Suspense fallback={<StatsSkeleton />}>
        <Stats /> {/* Server Component async */}
      </Suspense>

      <Suspense fallback={<ChartSkeleton />}>
        <RevenueChart /> {/* Load sau khi Stats xong không cần thiết */}
      </Suspense>

      <Suspense fallback={<TableSkeleton />}>
        <RecentOrders /> {/* Load độc lập */}
      </Suspense>
    </div>
  );
}
// Browser nhận HTML dần dần (streaming) thay vì đợi tất cả xong
```

---

## 2. APP ROUTER — ROUTING & LAYOUTS

### Lý thuyết & Bản chất
App Router dùng **file-system routing** trong thư mục `app/`. Mỗi segment URL tương ứng với 1 folder. Special files:

| File | Mục đích |
|------|----------|
| `page.tsx` | UI của route, publicly accessible |
| `layout.tsx` | Shared UI bao quanh page, **persist state khi navigate** |
| `loading.tsx` | Auto wrap trong `<Suspense>`, hiện khi loading |
| `error.tsx` | Error boundary cho segment, phải là Client Component |
| `not-found.tsx` | UI khi `notFound()` được throw |
| `route.ts` | API endpoint (thay thế `pages/api`) |
| `middleware.ts` | Chạy trước request (ở root hoặc `src/`) |

---

### Ví dụ 1 — Cơ bản: Cấu trúc routing cơ bản
```
app/
├── layout.tsx           → Root layout (bắt buộc)
├── page.tsx             → /
├── about/
│   └── page.tsx         → /about
├── blog/
│   ├── layout.tsx       → Shared layout cho /blog và /blog/[slug]
│   ├── page.tsx         → /blog
│   └── [slug]/
│       └── page.tsx     → /blog/anything
└── (marketing)/         → Route group — KHÔNG ảnh hưởng URL
    ├── layout.tsx        → Layout CHỈ cho các routes trong group
    └── contact/
        └── page.tsx     → /contact (không phải /marketing/contact)
```

```tsx
// app/layout.tsx — root layout
export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <Header />
        <main>{children}</main>
        <Footer />
      </body>
    </html>
  );
}

// app/blog/layout.tsx — nested layout
// Tồn tại khi navigate giữa /blog và /blog/[slug]
export default function BlogLayout({ children }) {
  return (
    <div className="flex">
      <BlogSidebar /> {/* persist khi navigate! state không bị reset */}
      <article>{children}</article>
    </div>
  );
}

// app/blog/[slug]/page.tsx
export default async function BlogPost({ params }) {
  const post = await getPost(params.slug);
  if (!post) notFound(); // throws → not-found.tsx
  return <PostContent post={post} />;
}
```

---

### Ví dụ 2 — Trung cấp: Dynamic Routes + generateStaticParams
```tsx
// Static paths cho SSG
export async function generateStaticParams() {
  const posts = await db.post.findMany({ select: { slug: true } });
  return posts.map(p => ({ slug: p.slug }));
  // Next.js pre-render tất cả paths này tại build time
}

// Catch-all routes
// app/docs/[...slug]/page.tsx → /docs/a, /docs/a/b, /docs/a/b/c
// app/docs/[[...slug]]/page.tsx → cũng match /docs

// Parallel Routes (@ convention)
// app/
//   layout.tsx
//   @modal/
//     (.)photo/[id]/   → intercepting route
//       page.tsx
//   photo/[id]/
//     page.tsx

// Intercepting Routes: khi navigate từ cùng layout → show modal
// Khi reload trang → show full page
// Dùng cho: photo gallery (click → modal), shopping cart slide-over
```

---

### Ví dụ 3 — Nâng cao: Server Actions
```tsx
// Server Actions: function async chạy trên server, gọi từ client
// Không cần API route!

// app/actions/user.ts
'use server'; // file-level directive

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

export async function updateProfile(formData: FormData) {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;

  // Validation
  if (!name || name.length < 2) {
    return { error: 'Name must be at least 2 characters' };
  }

  // Auth check
  const session = await getSession();
  if (!session) redirect('/login');

  // DB update
  await db.user.update({
    where: { id: session.userId },
    data: { name, email },
  });

  revalidatePath('/profile');
  return { success: true };
}

// Client Component dùng Server Action
'use client';
import { updateProfile } from '@/actions/user';
import { useFormState } from 'react-dom';

function ProfileForm({ user }) {
  const [state, formAction] = useFormState(updateProfile, null);

  return (
    <form action={formAction}>
      <input name="name" defaultValue={user.name} />
      <input name="email" defaultValue={user.email} />
      {state?.error && <p className="text-red-500">{state.error}</p>}
      {state?.success && <p className="text-green-500">Updated!</p>}
      <button type="submit">Save</button>
    </form>
  );
}
// Lợi ích: progressive enhancement (form hoạt động không có JS!),
// không expose API endpoint, automatic CSRF protection
```

---

## 3. CACHING TRONG NEXT.JS

### Lý thuyết & Bản chất
Next.js có **4 tầng caching** xếp chồng lên nhau:

```
Request → Router Cache (memory, client) →
         Full Route Cache (disk, server) →
         Data Cache (disk, server) →
         Origin (DB/API)
```

**Data Cache:** `fetch()` trong Next.js được **extend** để cache response. Default: cache mãi mãi (như SSG). Phải opt-out nếu muốn SSR behavior.

---

### Ví dụ 1 — Cơ bản: Fetch caching options
```tsx
// Cache mãi mãi (default, như SSG)
const data = await fetch('/api/products');

// Không cache (như SSR — chạy mỗi request)
const data = await fetch('/api/products', {
  cache: 'no-store'
});

// Cache với revalidation (như ISR)
const data = await fetch('/api/products', {
  next: { revalidate: 3600 } // 1 giờ
});

// Cache với tags (on-demand revalidation)
const data = await fetch('/api/products', {
  next: { tags: ['products', `product-${id}`] }
});
```

---

### Ví dụ 2 — Trung cấp: Route Segment Config + opting out
```tsx
// app/dashboard/page.tsx
// Force dynamic rendering (opt-out Full Route Cache)
export const dynamic = 'force-dynamic'; // tương đương getServerSideProps

// Revalidate toàn bộ route
export const revalidate = 3600; // ISR cho toàn route

// Chỉ generate các params này tại build
export const dynamicParams = false; // 404 cho params không có trong generateStaticParams

async function DashboardPage() {
  // Vì dynamic = 'force-dynamic':
  // - headers() và cookies() available
  // - không cache kết quả
  const user = await getCurrentUser();

  return <Dashboard user={user} />;
}
```

---

### Ví dụ 3 — Nâng cao: Unstable_cache + React cache deduplication
```tsx
import { unstable_cache } from 'next/cache';
import { cache } from 'react';

// unstable_cache: cache function result ở Data Cache layer
// Dùng khi không thể dùng fetch() (e.g., DB calls)
const getCachedUser = unstable_cache(
  async (userId: string) => {
    return db.user.findUnique({ where: { id: userId } });
  },
  ['user'], // cache key
  {
    tags: ['users', `user-${userId}`],
    revalidate: 60, // 1 phút
  }
);

// React cache: deduplicate trong 1 render request
// Hai component cùng gọi getUser(1) → chỉ query DB 1 lần
const getUser = cache(async (id: string) => {
  return db.user.findUnique({ where: { id } });
});

// app/layout.tsx và app/page.tsx cùng gọi getUser(session.userId)
// → chỉ query 1 lần trong 1 request cycle
async function Header() {
  const user = await getUser(session.userId); // query
  return <nav>{user.name}</nav>;
}

async function ProfileCard() {
  const user = await getUser(session.userId); // cache hit — không query lại!
  return <Card>{user.email}</Card>;
}
```

---

## 4. MIDDLEWARE & AUTHENTICATION

### Lý thuyết & Bản chất
**Middleware** trong Next.js chạy **trước khi request được xử lý** ở edge runtime (nhanh hơn, gần user hơn serverless functions). Middleware có thể:
- Redirect/rewrite request
- Modify request/response headers
- Kiểm tra auth

---

### Ví dụ 1 — Cơ bản: Route protection
```ts
// middleware.ts (root level)
import { NextRequest, NextResponse } from 'next/server';

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Lấy token từ cookie (HttpOnly cookie accessible ở middleware)
  const token = request.cookies.get('auth-token')?.value;

  // Protected routes
  const isProtected = pathname.startsWith('/dashboard') ||
                      pathname.startsWith('/profile');

  if (isProtected && !token) {
    const loginUrl = new URL('/login', request.url);
    loginUrl.searchParams.set('callbackUrl', pathname); // redirect sau login
    return NextResponse.redirect(loginUrl);
  }

  // Role-based access
  const isAdmin = pathname.startsWith('/admin');
  if (isAdmin) {
    const decoded = verifyToken(token); // JWT decode (nhẹ, edge runtime)
    if (decoded?.role !== 'admin') return NextResponse.redirect(new URL('/403', request.url));
  }

  return NextResponse.next();
}

// Chỉ apply middleware cho các routes này
export const config = {
  matcher: ['/dashboard/:path*', '/profile/:path*', '/admin/:path*'],
};
```

---

### Ví dụ 2 — Trung cấp: NextAuth.js setup
```ts
// app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import CredentialsProvider from 'next-auth/providers/credentials';
import GoogleProvider from 'next-auth/providers/google';

const handler = NextAuth({
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_ID!,
      clientSecret: process.env.GOOGLE_SECRET!,
    }),
    CredentialsProvider({
      name: 'credentials',
      credentials: { email: {}, password: {} },
      async authorize(credentials) {
        const user = await db.user.findUnique({ where: { email: credentials.email } });
        if (!user) return null;
        const isValid = await bcrypt.compare(credentials.password, user.password);
        if (!isValid) return null;
        return { id: user.id, email: user.email, name: user.name, role: user.role };
      },
    }),
  ],
  callbacks: {
    // jwt callback: chạy khi tạo/cập nhật token
    async jwt({ token, user }) {
      if (user) token.role = user.role; // thêm role vào token
      return token;
    },
    // session callback: chạy khi session được đọc
    async session({ session, token }) {
      session.user.role = token.role; // expose role trong session
      return session;
    },
  },
  pages: {
    signIn: '/login',
    error: '/auth/error',
  },
  session: { strategy: 'jwt' }, // stateless JWT, không cần DB session
});

export { handler as GET, handler as POST };
```

---

### Ví dụ 3 — Nâng cao: Auth với Server Components + Actions
```tsx
// lib/auth.ts — helper
import { getServerSession } from 'next-auth';
import { redirect } from 'next/navigation';

export async function requireAuth() {
  const session = await getServerSession(authOptions);
  if (!session) redirect('/login');
  return session;
}

export async function requireRole(role: string) {
  const session = await requireAuth();
  if (session.user.role !== role) redirect('/403');
  return session;
}

// app/admin/page.tsx
async function AdminPage() {
  const session = await requireRole('admin'); // redirect nếu không đủ quyền
  const stats = await db.admin.getStats();
  return <AdminDashboard stats={stats} user={session.user} />;
}

// Server Action với auth
'use server';
async function deletePost(postId: string) {
  const session = await requireAuth();
  
  const post = await db.post.findUnique({ where: { id: postId } });
  if (!post) throw new Error('Not found');
  if (post.authorId !== session.user.id && session.user.role !== 'admin') {
    throw new Error('Forbidden');
  }

  await db.post.delete({ where: { id: postId } });
  revalidatePath('/posts');
}
```

---

## TỔNG KẾT — Next.js Decision Tree

```
Cần render HTML?
├── Không (dashboard, app cần auth) → CSR hoặc SSR + auth check
└── Có (SEO quan trọng)
    ├── Data thay đổi theo từng user/request? → SSR (getServerSideProps hoặc Server Component + no-store)
    └── Data có thể share giữa users?
        ├── Ít thay đổi (blog, docs) → SSG
        ├── Thay đổi định kỳ (product list, news) → ISR với revalidate
        └── Thay đổi khi có event (CMS publish) → ISR + on-demand revalidation
```
