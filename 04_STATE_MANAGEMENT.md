# State Management — Lý thuyết sâu + Ví dụ 3 cấp độ

---

## 1. REDUX TOOLKIT

### Lý thuyết & Bản chất
Redux là thư viện quản lý **global state** theo pattern **Flux**: state luôn đi theo một chiều duy nhất.

**3 nguyên tắc:**
1. **Single source of truth**: Toàn bộ state của app trong 1 store
2. **State is read-only**: Chỉ thay đổi state bằng cách dispatch **action** (plain object mô tả điều gì đó đã xảy ra)
3. **Changes made with pure functions**: Reducer nhận state cũ + action → trả state mới (KHÔNG mutate!)

**Data flow:**
```
UI Event → dispatch(action) → Reducer(state, action) → New State → UI re-render
```

**Redux Toolkit (RTK)** = Redux "official" với boilerplate giảm thiểu:
- `createSlice`: tạo actions + reducer cùng lúc
- `createAsyncThunk`: xử lý async với pending/fulfilled/rejected states
- **Immer** tích hợp sẵn: cho phép "mutate" trong reducer (thực ra tạo state mới)
- **RTK Query**: data fetching + caching tích hợp vào Redux

---

### Ví dụ 1 — Cơ bản: Slice + Store setup
```tsx
// store/slices/authSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface User { id: string; name: string; email: string; role: string; }
interface AuthState { user: User | null; token: string | null; }

const authSlice = createSlice({
  name: 'auth',
  initialState: { user: null, token: null } as AuthState,
  reducers: {
    // Immer cho phép "mutate" trực tiếp
    setCredentials(state, action: PayloadAction<{ user: User; token: string }>) {
      state.user = action.payload.user;
      state.token = action.payload.token;
    },
    logout(state) {
      state.user = null;
      state.token = null;
    },
  },
});

export const { setCredentials, logout } = authSlice.actions;
export default authSlice.reducer;

// Selectors (nên đặt cùng slice)
export const selectCurrentUser = (state: RootState) => state.auth.user;
export const selectIsAuthenticated = (state: RootState) => !!state.auth.token;

// store/index.ts
import { configureStore } from '@reduxjs/toolkit';

export const store = configureStore({
  reducer: {
    auth: authReducer,
    cart: cartReducer,
    // ...
  },
  // RTK tự thêm redux-thunk và redux-devtools-extension
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

---

### Ví dụ 2 — Trung cấp: createAsyncThunk + extraReducers
```tsx
// store/slices/usersSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

// createAsyncThunk tự động dispatch:
// users/fetchById/pending
// users/fetchById/fulfilled (với kết quả)
// users/fetchById/rejected (với error)
export const fetchUserById = createAsyncThunk(
  'users/fetchById',
  async (userId: string, { rejectWithValue, getState }) => {
    try {
      const token = (getState() as RootState).auth.token;
      const response = await fetch(`/api/users/${userId}`, {
        headers: { Authorization: `Bearer ${token}` },
      });
      if (!response.ok) throw new Error('Failed');
      return await response.json();
    } catch (error) {
      // rejectWithValue cho phép truyền error payload tùy chỉnh
      return rejectWithValue(error.message);
    }
  }
);

interface UsersState {
  entities: Record<string, User>;
  loading: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
}

const usersSlice = createSlice({
  name: 'users',
  initialState: { entities: {}, loading: 'idle', error: null } as UsersState,
  reducers: {
    userUpdated(state, action: PayloadAction<User>) {
      state.entities[action.payload.id] = action.payload;
    },
  },
  extraReducers(builder) {
    builder
      .addCase(fetchUserById.pending, (state) => {
        state.loading = 'loading';
        state.error = null;
      })
      .addCase(fetchUserById.fulfilled, (state, action) => {
        state.loading = 'succeeded';
        state.entities[action.payload.id] = action.payload;
      })
      .addCase(fetchUserById.rejected, (state, action) => {
        state.loading = 'failed';
        state.error = action.payload as string;
      });
  },
});
```

---

### Ví dụ 3 — Nâng cao: RTK Query
```tsx
// RTK Query: data fetching layer được tích hợp vào Redux
// Tự động handle: caching, deduplication, loading/error states, background refresh

// store/api/productsApi.ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const productsApi = createApi({
  reducerPath: 'productsApi',
  baseQuery: fetchBaseQuery({
    baseUrl: '/api',
    // Auto thêm auth header
    prepareHeaders(headers, { getState }) {
      const token = (getState() as RootState).auth.token;
      if (token) headers.set('Authorization', `Bearer ${token}`);
      return headers;
    },
  }),
  tagTypes: ['Product', 'ProductList'],
  endpoints: (build) => ({
    getProducts: build.query<Product[], { category?: string }>({
      query: ({ category } = {}) => ({
        url: '/products',
        params: category ? { category } : {},
      }),
      providesTags: (result) =>
        result
          ? [...result.map(({ id }) => ({ type: 'Product' as const, id })), 'ProductList']
          : ['ProductList'],
    }),

    getProductById: build.query<Product, string>({
      query: (id) => `/products/${id}`,
      providesTags: (_, __, id) => [{ type: 'Product', id }],
    }),

    createProduct: build.mutation<Product, Omit<Product, 'id'>>({
      query: (body) => ({ url: '/products', method: 'POST', body }),
      invalidatesTags: ['ProductList'], // auto refetch list sau khi tạo mới
    }),

    updateProduct: build.mutation<Product, { id: string; patch: Partial<Product> }>({
      query: ({ id, patch }) => ({ url: `/products/${id}`, method: 'PATCH', body: patch }),
      // Optimistic update
      async onQueryStarted({ id, patch }, { dispatch, queryFulfilled }) {
        const patchResult = dispatch(
          productsApi.util.updateQueryData('getProductById', id, (draft) => {
            Object.assign(draft, patch);
          })
        );
        try {
          await queryFulfilled;
        } catch {
          patchResult.undo(); // rollback nếu server error
        }
      },
      invalidatesTags: (_, __, { id }) => [{ type: 'Product', id }],
    }),
  }),
});

export const {
  useGetProductsQuery,
  useGetProductByIdQuery,
  useCreateProductMutation,
  useUpdateProductMutation,
} = productsApi;

// Component sử dụng
function ProductsList({ category }) {
  const { data: products, isLoading, isFetching } = useGetProductsQuery({ category });
  const [createProduct, { isLoading: isCreating }] = useCreateProductMutation();

  if (isLoading) return <Skeleton />;
  return (
    <div>
      {isFetching && <RefreshIndicator />} {/* background refresh */}
      {products?.map(p => <ProductCard key={p.id} product={p} />)}
    </div>
  );
}
```

---

## 2. ZUSTAND

### Lý thuyết & Bản chất
Zustand là state management nhỏ gọn (~1KB). Không có boilerplate Redux, không cần Provider wrapper.

**Bản chất:** Store là một closure. `create` tạo ra một React hook. Khi state thay đổi, chỉ những component **subscribe** vào **phần state đó** mới re-render (selector-based subscription).

**Khi nào dùng Zustand thay Redux?**
- App nhỏ đến trung bình
- Không cần devtools phức tạp
- Team ít người, ít boilerplate hơn
- Không cần RTK Query (đã dùng React Query)

---

### Ví dụ 1 — Cơ bản: Counter + Todo store
```tsx
import { create } from 'zustand';

// Store đơn giản
interface CounterStore {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
  incrementBy: (amount: number) => void;
}

const useCounterStore = create<CounterStore>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
  incrementBy: (amount) => set((state) => ({ count: state.count + amount })),
}));

// Component — chỉ subscribe vào `count`, không re-render khi actions thay đổi
function Counter() {
  const count = useCounterStore((state) => state.count); // selector!
  const { increment, decrement } = useCounterStore();

  return (
    <div>
      <button onClick={decrement}>-</button>
      <span>{count}</span>
      <button onClick={increment}>+</button>
    </div>
  );
}

// QUAN TRỌNG: Nếu không dùng selector → re-render khi BẤT KỲ state nào thay đổi
const everythingWrong = useCounterStore(); // BAD: re-render khi bất kỳ field nào đổi
const countOnly = useCounterStore(s => s.count); // GOOD: chỉ re-render khi count đổi
```

---

### Ví dụ 2 — Trung cấp: Persist + DevTools
```tsx
import { create } from 'zustand';
import { persist, devtools } from 'zustand/middleware';

interface CartStore {
  items: CartItem[];
  addItem: (product: Product) => void;
  removeItem: (id: string) => void;
  updateQuantity: (id: string, qty: number) => void;
  clearCart: () => void;
  total: number; // computed
}

const useCartStore = create<CartStore>()(
  devtools( // Redux DevTools support
    persist( // localStorage persistence
      (set, get) => ({
        items: [],
        total: 0,

        addItem: (product) => set((state) => {
          const existing = state.items.find(i => i.id === product.id);
          const items = existing
            ? state.items.map(i => i.id === product.id
                ? { ...i, qty: i.qty + 1 }
                : i)
            : [...state.items, { ...product, qty: 1 }];
          return { items, total: items.reduce((sum, i) => sum + i.price * i.qty, 0) };
        }),

        removeItem: (id) => set((state) => {
          const items = state.items.filter(i => i.id !== id);
          return { items, total: items.reduce((sum, i) => sum + i.price * i.qty, 0) };
        }),

        updateQuantity: (id, qty) => set((state) => {
          const items = qty <= 0
            ? state.items.filter(i => i.id !== id)
            : state.items.map(i => i.id === id ? { ...i, qty } : i);
          return { items, total: items.reduce((sum, i) => sum + i.price * i.qty, 0) };
        }),

        clearCart: () => set({ items: [], total: 0 }),
      }),
      {
        name: 'cart-storage', // localStorage key
        partialize: (state) => ({ items: state.items }), // chỉ persist items, không persist total
      }
    ),
    { name: 'CartStore' } // devtools name
  )
);
```

---

### Ví dụ 3 — Nâng cao: Slice pattern + Subscriptions
```tsx
// Chia store lớn thành slices
interface AuthSlice {
  user: User | null;
  setUser: (user: User | null) => void;
}

interface UISlice {
  theme: 'light' | 'dark';
  sidebarOpen: boolean;
  toggleTheme: () => void;
  toggleSidebar: () => void;
}

type AppStore = AuthSlice & UISlice;

const createAuthSlice = (set: any): AuthSlice => ({
  user: null,
  setUser: (user) => set({ user }, false, 'auth/setUser'),
});

const createUISlice = (set: any): UISlice => ({
  theme: 'light',
  sidebarOpen: true,
  toggleTheme: () => set(
    (s: AppStore) => ({ theme: s.theme === 'light' ? 'dark' : 'light' }),
    false, 'ui/toggleTheme'
  ),
  toggleSidebar: () => set(
    (s: AppStore) => ({ sidebarOpen: !s.sidebarOpen }),
    false, 'ui/toggleSidebar'
  ),
});

const useAppStore = create<AppStore>()(
  devtools((...args) => ({
    ...createAuthSlice(...args),
    ...createUISlice(...args),
  }))
);

// Subscribe NGOÀI React (cho analytics, sync với server, etc.)
const unsubscribe = useAppStore.subscribe(
  (state) => state.user, // selector
  (user, prevUser) => {
    if (user && !prevUser) analytics.track('login', { userId: user.id });
    if (!user && prevUser) analytics.track('logout');
  }
);
```

---

## 3. CONTEXT API

### Lý thuyết & Bản chất
Context API là cơ chế **prop drilling** giải pháp của React — truyền data từ component cha xuống con mà không cần pass qua mọi tầng.

**Bản chất:** `createContext` tạo một React context object. `Provider` đặt value vào context tree. `useContext` đọc value từ Provider gần nhất.

**Pitfall quan trọng nhất:** Mỗi khi `value` của Provider thay đổi (kể cả tạo object mới `{}`), **TẤT CẢ consumers re-render** — không phân biệt chúng có dùng phần đó không.

**Context tốt cho:** Theme, locale, auth user (ít thay đổi). **Không tốt cho:** Frequently changing state (dùng Zustand/Redux).

---

### Ví dụ 1 — Cơ bản: Theme context
```tsx
// contexts/ThemeContext.tsx
type Theme = 'light' | 'dark';

interface ThemeContextValue {
  theme: Theme;
  toggleTheme: () => void;
}

const ThemeContext = React.createContext<ThemeContextValue | null>(null);

// Custom hook để dùng context với type safety + better error message
export function useTheme() {
  const ctx = React.useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be used within ThemeProvider');
  return ctx;
}

export function ThemeProvider({ children }) {
  const [theme, setTheme] = React.useState<Theme>('light');

  // Memoize value để tránh re-render khi Provider re-render vì lý do khác
  const value = React.useMemo(
    () => ({ theme, toggleTheme: () => setTheme(t => t === 'light' ? 'dark' : 'light') }),
    [theme]
  );

  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}

// Dùng
function Button() {
  const { theme, toggleTheme } = useTheme();
  return (
    <button
      className={theme === 'dark' ? 'bg-gray-800 text-white' : 'bg-white text-black'}
      onClick={toggleTheme}
    >
      Toggle
    </button>
  );
}
```

---

### Ví dụ 2 — Trung cấp: Split context để tối ưu re-render
```tsx
// Vấn đề: cả state và dispatch trong cùng 1 context
// → component chỉ cần dispatch vẫn re-render khi state đổi

// Giải pháp: tách thành 2 context
const UserStateContext = React.createContext<User | null>(null);
const UserDispatchContext = React.createContext<React.Dispatch<UserAction> | null>(null);

export function UserProvider({ children }) {
  const [user, dispatch] = React.useReducer(userReducer, null);

  return (
    <UserStateContext.Provider value={user}>
      {/* Dispatch context không thay đổi → consumers không re-render */}
      <UserDispatchContext.Provider value={dispatch}>
        {children}
      </UserDispatchContext.Provider>
    </UserStateContext.Provider>
  );
}

// Component chỉ đọc user → re-render khi user thay đổi
export function useUser() {
  const ctx = React.useContext(UserStateContext);
  if (ctx === undefined) throw new Error('useUser must be inside UserProvider');
  return ctx;
}

// Component chỉ dispatch → KHÔNG re-render khi user thay đổi!
export function useUserDispatch() {
  const ctx = React.useContext(UserDispatchContext);
  if (!ctx) throw new Error('useUserDispatch must be inside UserProvider');
  return ctx;
}
```

---

### Ví dụ 3 — Nâng cao: Context với external store + subscription
```tsx
// Sync context với external state (như URL params, localStorage)
function createExternalStoreContext<T>(getSnapshot: () => T, subscribe: (cb: () => void) => () => void) {
  const Context = React.createContext<T>(getSnapshot());

  function Provider({ children }) {
    const value = React.useSyncExternalStore(subscribe, getSnapshot);
    return <Context.Provider value={value}>{children}</Context.Provider>;
  }

  return { Context, Provider, useValue: () => React.useContext(Context) };
}

// URL search params context
const { Provider: SearchParamsProvider, useValue: useSearchParams } =
  createExternalStoreContext(
    () => new URLSearchParams(window.location.search),
    (callback) => {
      window.addEventListener('popstate', callback);
      return () => window.removeEventListener('popstate', callback);
    }
  );
```

---

## TỔNG KẾT — State Management Decision

```
Global state cần shared giữa nhiều component?
├── Không → useState / useReducer trong component hoặc parent
└── Có
    ├── Server data (cần cache, sync, background refresh)?
    │   └── React Query / RTK Query
    └── Client state (UI, auth, cart...)
        ├── Ít thay đổi, simple (theme, locale, auth user)?
        │   └── Context API (với split để tối ưu)
        ├── Moderate complexity, muốn nhẹ?
        │   └── Zustand (selector pattern)
        └── Complex, cần devtools, middleware, large team?
            └── Redux Toolkit (+ RTK Query cho server state)
```

| | Context | Zustand | Redux Toolkit |
|---|---|---|---|
| Bundle size | 0 (built-in) | ~1KB | ~10KB |
| Boilerplate | Ít | Rất ít | Trung bình (có RTK) |
| DevTools | ❌ | ✅ (qua middleware) | ✅ (xuất sắc) |
| Performance | Cần optimize thủ công | Tự động (selector) | Tự động (selector) |
| Async | useEffect + useReducer | Get/set trong actions | createAsyncThunk |
| Persistence | Tự code | Middleware `persist` | Middleware |
