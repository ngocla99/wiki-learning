# Performance & Optimization — Interview Questions

> Sources: Nadia Makarevich, MDN, web.dev, personal experience
> Updated: 2026-04-21

## Overview

Interview questions covering browser performance patterns, image optimization, memory management, React-specific optimizations, API efficiency, and Next.js production optimizations.

---

## Debouncing vs Throttling

### What is the difference between debouncing and throttling?

Both techniques limit how often a function runs in response to rapid events (typing, scrolling, resizing), but their timing strategy is fundamentally different.

| | Debounce | Throttle |
|---|----------|----------|
| **When it fires** | After the last call + delay has passed | At most once per interval |
| **Timer behavior** | Resets on every new call | Ignores calls during the interval |
| **Use case** | Search input, form validation | Scroll handlers, auto-save, resize |
| **Risk** | May never fire if user never pauses | Might miss the final call |

```js
// Debounce — fires 500ms after the last keystroke
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

// Throttle — fires at most once every 200ms
function throttle(fn, interval) {
  let lastTime = 0;
  return (...args) => {
    const now = Date.now();
    if (now - lastTime >= interval) {
      lastTime = now;
      fn(...args);
    }
  };
}
```

**Decision rule:** If you only care about the *final* value (search query after typing stops) → debounce. If you need *regular updates* during continuous action (scroll position, auto-save every N seconds) → throttle.

### How do you implement these correctly in React?

Naive implementations break in React because the component re-renders re-create the function, destroying the internal timer:

```jsx
// BROKEN: new debounced function created on every render
const Input = ({ onSearch }) => {
  const handleChange = debounce(onSearch, 500); // new instance each render
  return <input onChange={handleChange} />;
};

// CORRECT: stable ref persists across renders
function useDebounce(fn, delay) {
  const fnRef = useRef(fn);
  useEffect(() => { fnRef.current = fn; });

  return useCallback(
    debounce((...args) => fnRef.current(...args), delay),
    [delay] // only recreate if delay changes
  );
}
```

---

## React Optimization — Overview Table

| # | Kỹ thuật | Mô tả ngắn |
| - | -------- | ---------- |
| **🔁 Render Optimization** | | |
| 1 | `React.memo` | Bỏ qua re-render khi props không đổi (shallow compare) |
| 2 | `useMemo` | Cache kết quả tính toán nặng giữa các render |
| 3 | `useCallback` | Tạo stable function reference để không phá vỡ `React.memo` |
| 4 | State colocation | Giữ state gần nơi dùng nhất — tránh re-render cả cây |
| 5 | Children as props | Tách component hay thay đổi ra khỏi scope component ổn định |
| 6 | Context splitting | Tách context theo tần suất thay đổi — tránh re-render toàn bộ consumer |
| **📦 Bundle Size** | | |
| 7 | `React.lazy` + `Suspense` | Code splitting — chỉ tải component khi cần |
| 8 | Tree shaking | Import named, dùng ESM để bundler loại bỏ code không dùng |
| 9 | Dependency audit | Thay thư viện nặng (`moment` → `date-fns`, `lodash` → native) |
| 10 | Bundle analyzer | Phát hiện chunk nặng không cần thiết |
| **🖼️ Load Performance** | | |
| 11 | Lazy loading component | Modal, drawer, below-the-fold chỉ tải khi mount lần đầu |
| 12 | Virtualization | `react-window` / `react-virtual` — chỉ render row đang visible |
| 13 | Image optimization | WebP/AVIF, `srcset`, `loading="lazy"`, `next/image` |
| **⚡ Data Fetching** | | |
| 14 | TanStack Query / SWR | Cache, deduplication, background refetch tự động |
| 15 | Debounce search | Tránh gọi API mỗi keystroke |
| 16 | Pagination / Infinite scroll | Không load toàn bộ data cùng lúc |
| 17 | `AbortController` | Hủy request cũ khi có request mới |
| **🚀 Advanced** | | |
| 18 | `useTransition` | Đánh dấu update không urgent — giữ UI responsive |
| 19 | React Compiler | Tự động memo hóa, không cần viết `useMemo`/`useCallback` thủ công |
| 20 | Server Components | Chạy trên server — zero JS bundle gửi về client |
| 21 | SSG / ISR | Pre-render tĩnh thay vì SSR mỗi request |

---
