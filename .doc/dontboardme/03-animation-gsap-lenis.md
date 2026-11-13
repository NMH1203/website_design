# 03. Animation: GSAP + Lenis

Đây là phần làm nên "chất" của site. Toàn bộ motion chạy bằng **GSAP 3.12.2**, cuộn mượt bằng **Lenis 1.0.25**.

## 1. Plugin Nuxt cung cấp GSAP toàn cục

Trong `entry.js` (đã đặt lại tên):

```js
export default defineNuxtPlugin(() => {
  gsap.registerPlugin(ScrollTrigger, ScrollToPlugin, DrawSVGPlugin, SplitText, CustomEase, Flip)
  CustomEase.create("my-ease", "0.24, 1, 0.36, 1")
  return { provide: { gsap, ScrollTrigger, ScrollToPlugin, DrawSVGPlugin, SplitText, Flip } }
})
```

Trong component, plugin được lấy ra bằng `const { $gsap } = useNuxtApp()`.

- **`my-ease` = cubic-bezier(0.24, 1, 0.36, 1)**: một đường ease-out rất "mềm", gần với expo-out. Cả site dùng chung đường cong này nên chuyển động trông đồng bộ.
- Ở bản 3.12, SplitText và DrawSVG là plugin trả phí (Club GreenSock). Từ GSAP 3.13 (năm 2025), **toàn bộ plugin đã miễn phí**, nên ngày nay có thể dùng thoải mái.

## 2. Chữ hiện theo từng dòng (SplitText line reveal)

Đây là composable `useShowPageAnimFirst` (đã đặt lại tên):

```js
function revealLines(selector, vars = {}, immediate = false) {
  const els = document.querySelectorAll(selector)
  const inView = els[0].getBoundingClientRect().top <= innerHeight
  if (inView) gsap.set(els, { opacity: 1 })

  // tách 2 lần: lớp ngoài làm "mặt nạ" (overflow hidden), lớp trong là phần chuyển động
  const { lines: masks } = new SplitText(els, { type: "lines", linesClass: "text-line-wrap" })
  const { lines }        = new SplitText(masks, { type: "lines" })

  return gsap.from(lines, {
    yPercent: 100, rotate: 30, transformOrigin: "left",
    stagger: 0.15, duration: 0.8, delay: 0.1, ease: "power4.out",
    ...(!inView && { scrollTrigger: { trigger: els[0], start: "top bottom" } }),
    onComplete: () => gsap.set(masks, { overflow: "unset" }),
    clearProps: "all", ...vars
  })
}
```

Mẹo đáng học:

- **Tách hai lớp**: lớp ngoài `.text-line-wrap { overflow: hidden }` làm mặt nạ, lớp trong trượt lên từ `yPercent: 100` và **xoay 30°** quanh mép trái, tạo cảm giác chữ "lật" lên.
- Nếu phần tử đã nằm trong viewport thì chạy ngay; nếu chưa thì gắn ScrollTrigger.
- `clearProps: "all"` và trả `overflow` về như cũ khi xong, để layout không bị kẹt style inline.

Composable thứ hai làm hiệu ứng **fade-up** cho đoạn văn: `yPercent: 28, opacity: 0, duration: 1.2, ease: "power2.out"`, đặt `willChange` trước khi chạy.

## 3. Đồng bộ animation với preloader và chuyển trang

```js
function useShowPageAnim(cb) {
  const preloaderDone = useState("preloader-state")
  const animWhilePreloader = useState("animate-page-while-preloader")
  const navigationDone = useState("navigation-state")
  if (preloaderDone.value)
    onMounted(() => watch(navigationDone, ok => ok && cb()))   // lần sau: đợi chuyển trang xong
  else
    watch(animWhilePreloader, ok => ok && cb())                // lần đầu: đợi preloader
}
```

→ **Animation vào trang không bao giờ chạy khi người dùng chưa nhìn thấy trang.** Đây là pattern then chốt khi kết hợp preloader, page transition và animation.

## 4. Preloader

- Component `CommonPreloader`: một bộ đếm `value: 0 → 100` chạy bằng GSAP và **chạy theo con trỏ chuột** (lưu toạ độ `{x, y}` từ `mousemove`).
- Trong lúc preloader chạy thì cuộn trang bị khoá (`lockScroll`).
- Khi xong, một timeline thu nhỏ hoặc biến mất về phía vị trí chuột, rồi set `preloader-state = true`.

## 5. Chuyển trang bằng clip-path (route middleware)

```js
// global middleware (đã đặt lại tên)
export default defineNuxtRouteMiddleware((to, from) => {
  const navDone = useState("navigation-state")
  from.meta.pageTransition = { mode: "in-out", onLeave }
  to.meta.pageTransition   = { mode: "in-out", onEnter }

  function onLeave(el, done) {
    navDone.value = false
    gsap.timeline({ onComplete() { gsap.set([el, el.querySelector(".page")], { clearProps: "all" }); done() } })
        .set(el, { position: "relative", zIndex: 2 })
        .to(el.querySelector(".page"), { duration: 1 })
  }

  function onEnter(el, done) {
    gsap.set(el, { position: "fixed", top: 0, left: 0, width: "100%", zIndex: 3 })
    gsap.timeline({ onComplete() {
        gsap.set(el, { clearProps: "all" })
        gsap.set(window, { scrollTo: 0 })          // ScrollToPlugin
        setTimeout(() => navDone.value = true, 500)
        done()
      } })
      .fromTo(el, { clipPath: "polygon(100% 0%,100% 0%,100% 100%,100% 100%)" },
                  { clipPath: "polygon(0% 0%,100% 0%,100% 100%,0% 100%)", duration: 1 })
      .from(el.querySelector(".page"), { xPercent: 50, duration: 1 }, "<")
  }
})
```

Cách hoạt động:

- Mode `in-out`: trang mới được đặt `position: fixed` **đè lên** trang cũ.
- Trang mới hiện dần từ mép phải nhờ `clip-path` mở từ một đường thẳng đứng thành hình chữ nhật đầy đủ.
- Cùng lúc đó, nội dung bên trong trượt từ `xPercent: 50` về 0, tạo hiệu ứng **parallax "trang trượt"**.
- Khi xong thì xoá style, cuộn về đầu trang, rồi bật `navigation-state` để animation của trang mới bắt đầu.

## 6. ScrollTrigger: các pattern dùng trên trang chủ

**a) Scrub theo cuộn**: hai quả bóng trôi ngược chiều nhau:

```js
const tl = gsap.timeline({ defaults: { scrollTrigger: {
  trigger: ".care-pet_content", start: "top bottom", end: "center center", scrub: true } } })
tl.to(".care-pet_left-ball",  { yPercent: 50 })
  .to(".care-pet_right-ball", { yPercent: -50 })   // chiều ngược lại (suy luận)
```

**b) Thẻ "How it works" xếp chồng khi cuộn**: mỗi `.wrap-process-card` di chuyển tới vị trí `top` đã tính sẵn, với `scrub: true`. Các callback `onEnter` và `onLeaveBack` cập nhật chỉ số bước đang active (một `ref`) để highlight trên UI.

**c) Responsive bằng `gsap.matchMedia()`**:

```js
mm.add("(max-width: 1024px)", () => cards.forEach(c =>
  gsap.from(c, { scrollTrigger: { trigger: c, start: "top bottom" }, y: 40, opacity: 0, duration: .6 })))
mm.add("(min-width: 1024px)", () => /* animation khác cho desktop */)
```

**d) Chỉ lắng nghe `mousemove` khi section đang hiện trên màn hình**: đây là pattern tối ưu hiệu năng:

```js
ScrollTrigger.create({
  trigger: ".our-services", start: "top bottom", end: "bottom center",
  onEnter:     () => addEventListener("mousemove", onMove),
  onEnterBack: () => addEventListener("mousemove", onMove),
  onLeave:     () => removeEventListener("mousemove", onMove),
  onLeaveBack: () => removeEventListener("mousemove", onMove),
})
onUnmounted(() => removeEventListener("mousemove", onMove))
```

Các section áp dụng: `.our-services` (ảnh chạy theo chuột), `.plash-move-cards`, `.care-pet_content` (parallax), và khối Instagram (nghiêng theo chuột: `(clientX - width/2) / 25`).

## 7. Lenis: cuộn mượt

```js
const lenis = new Lenis({ duration: 1.2, orientation: "vertical", smoothWheel: true })
```

- Bản 1.0.25 (package cũ `@studio-freight/lenis`). Ngày nay package đã đổi tên thành `lenis`.
- Hỗ trợ các thuộc tính `data-lenis-prevent`, `data-lenis-prevent-wheel` và `data-lenis-prevent-touch` để chặn cuộn mượt trong popup hoặc vùng có thanh cuộn riêng.
- **(Suy luận)** Lenis được nối với ScrollTrigger theo cách chuẩn: `lenis.on('scroll', ScrollTrigger.update)` và `gsap.ticker.add(t => lenis.raf(t*1000))`.

## 8. Tóm tắt "công thức motion"

| Thành phần | Giá trị |
|---|---|
| Ease chủ đạo | `cubic-bezier(.24,1,.36,1)`, `power4.out`, `power2.out` |
| Thời lượng | 0.6s (mobile), 0.8s (chữ), 1.0s (chuyển trang), 1.2s (fade và Lenis) |
| Stagger chữ | 0.15s mỗi dòng |
| Breakpoint animation | 1024px |
| Tương tác | parallax theo chuột chỉ bật khi section đang trong viewport |
