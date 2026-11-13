# 06. Áp dụng cho Portfolio

Mục tiêu là dựng lại **cảm giác chuyển động** của dontboardme.com cho portfolio cá nhân. Lưu ý chỉ học kỹ thuật, không sao chép nội dung, hình ảnh hay thiết kế của họ.

## 1. Stack đề xuất (bản 2026)

| Mục | Lựa chọn | Lý do |
|---|---|---|
| Framework | Nuxt 3/4 (SSR hoặc `nuxi generate` ra site tĩnh) | giống site gốc; portfolio có thể build tĩnh rồi deploy lên Vercel hoặc Netlify |
| Animation | `gsap` ≥ 3.13 | mọi plugin (SplitText, DrawSVG...) giờ đã miễn phí |
| Cuộn mượt | `lenis` (package mới) | |
| Nội dung | Nuxt Content (file Markdown) thay cho Strapi | portfolio không cần CMS riêng |
| Slider | Swiper hoặc CSS marquee | |
| SEO | `useSeoMeta` và `@nuxtjs/sitemap` | sửa các lỗi đã nêu ở file 05 |

```bash
npx nuxi@latest init portfolio
npm i gsap lenis
```

## 2. Plugin GSAP và Lenis

`plugins/gsap.client.ts`
```ts
import gsap from "gsap"
import { ScrollTrigger } from "gsap/ScrollTrigger"
import { ScrollToPlugin } from "gsap/ScrollToPlugin"
import { SplitText } from "gsap/SplitText"
import { CustomEase } from "gsap/CustomEase"
import Lenis from "lenis"

export default defineNuxtPlugin(() => {
  gsap.registerPlugin(ScrollTrigger, ScrollToPlugin, SplitText, CustomEase)
  CustomEase.create("soft", "0.24, 1, 0.36, 1")
  gsap.defaults({ ease: "soft" })

  const lenis = new Lenis({ duration: 1.2, smoothWheel: true })
  lenis.on("scroll", ScrollTrigger.update)
  gsap.ticker.add(t => lenis.raf(t * 1000))
  gsap.ticker.lagSmoothing(0)

  return { provide: { gsap, ScrollTrigger, SplitText, lenis } }
})
```

## 3. Composable: chữ hiện theo từng dòng

`composables/useRevealLines.ts`
```ts
export function useRevealLines(selector: string, vars: gsap.TweenVars = {}) {
  const { $gsap, $SplitText } = useNuxtApp()
  let ctx: gsap.Context
  onMounted(() => {
    ctx = $gsap.context(() => {
      const el = document.querySelector(selector)
      if (!el) return
      // autoSplit + onSplit (GSAP 3.13+) tự split lại khi font hoặc resize thay đổi
      $SplitText.create(el, {
        type: "lines", mask: "lines", autoSplit: true,
        onSplit: (self) => $gsap.from(self.lines, {
          yPercent: 100, rotate: 30, transformOrigin: "left",
          stagger: 0.15, duration: 0.8, ease: "power4.out",
          scrollTrigger: { trigger: el, start: "top 90%" }, ...vars
        })
      })
    })
  })
  onUnmounted(() => ctx?.revert())   // dọn sạch khi chuyển trang
}
```

> SplitText 3.13 có tuỳ chọn `mask: "lines"` nên không cần tách hai lần như site gốc.

## 4. Chuyển trang bằng clip-path

`app.vue`
```vue
<template>
  <NuxtLayout><NuxtPage :transition="pageTransition" /></NuxtLayout>
</template>

<script setup lang="ts">
const { $gsap } = useNuxtApp()
const navDone = useState("navigation-state", () => true)

const pageTransition = {
  mode: "in-out" as const,
  css: false,
  onLeave(el: Element, done: () => void) {
    navDone.value = false
    $gsap.to(el, { duration: 1, onComplete: done })    // giữ trang cũ bên dưới
  },
  onEnter(el: Element, done: () => void) {
    $gsap.set(el, { position: "fixed", inset: 0, zIndex: 3 })
    $gsap.timeline({ onComplete() {
        $gsap.set(el, { clearProps: "all" })
        window.scrollTo(0, 0)
        navDone.value = true
        done()
      } })
      .fromTo(el, { clipPath: "inset(0 0 0 100%)" },
                  { clipPath: "inset(0 0 0 0%)", duration: 1, ease: "soft" })
      .from(el.firstElementChild, { xPercent: 50, duration: 1, ease: "soft" }, "<")
  },
}
</script>
```

`inset()` cho kết quả tương đương `polygon(...)` của site gốc nhưng viết ngắn hơn.

## 5. Chỉ chạy animation trang khi đã sẵn sàng

```ts
export function useOnPageReady(cb: () => void) {
  const navDone = useState("navigation-state", () => true)
  onMounted(() => navDone.value ? cb() : watch(navDone, v => v && cb(), { once: true }))
}
```

## 6. Parallax theo chuột, chỉ bật khi section đang hiện

```ts
export function useMouseParallax(section: string, target: string, strength = 25) {
  const { $gsap, $ScrollTrigger } = useNuxtApp()
  onMounted(() => {
    const xTo = $gsap.quickTo(target, "x", { duration: 0.6 })
    const yTo = $gsap.quickTo(target, "y", { duration: 0.6 })
    const move = (e: MouseEvent) => {
      xTo((e.clientX - innerWidth / 2) / strength)
      yTo((e.clientY - innerHeight / 2) / strength)
    }
    $ScrollTrigger.create({
      trigger: section, start: "top bottom", end: "bottom top",
      onToggle: ({ isActive }) =>
        isActive ? addEventListener("mousemove", move) : removeEventListener("mousemove", move),
    })
    onUnmounted(() => removeEventListener("mousemove", move))
  })
}
```

Ở đây dùng `quickTo` thay cho `gsap.to` trong mỗi lần `mousemove`, nên đỡ tốn tài nguyên hơn cách site gốc làm.

## 7. Marquee ảnh dự án (không cần Swiper)

```css
.marquee { overflow: hidden; }
.marquee__track { display: flex; width: max-content; animation: marquee 30s linear infinite; }
@keyframes marquee { to { transform: translateX(-50%); } }   /* nhân đôi danh sách item */
@media (prefers-reduced-motion: reduce) { .marquee__track { animation: none; } }
```

## 8. Checklist cải tiến so với site gốc

- [ ] Tôn trọng `prefers-reduced-motion`: tắt Lenis, SplitText và chuyển trang khi người dùng bật tuỳ chọn này
- [ ] Không đặt `maximum-scale=1` trong viewport
- [ ] Dùng `useSeoMeta` (thuộc tính `property="og:*"` đúng chuẩn), thêm sitemap và canonical
- [ ] Chỉ import plugin GSAP thực sự dùng, và lazy-load Lottie
- [ ] Gọi `gsap.context().revert()` khi unmount để tránh rò rỉ ScrollTrigger giữa các trang
- [ ] Dùng `@media (hover: hover)` cho hover effect, và một breakpoint chính khoảng 1024px
- [ ] Tạo design tokens bằng CSS variables theo pattern `--self-*` / `--x, fallback` cho component
