# 01. Tổng quan tech stack

## Bảng công nghệ

| Lớp | Công nghệ | Phiên bản | Bằng chứng |
|---|---|---|---|
| Framework | **Nuxt 3** (SSR) | **3.6.2** | `versions:{nuxt:"3.6.2"}` trong entry; header `x-powered-by: Nuxt`; `__NUXT_DATA__` |
| UI | **Vue 3** | **3.3.4** | chuỗi `"3.3.4"` trong bundle, thuộc tính scoped `data-v-xxxx` |
| Bundler | **Vite** (mặc định của Nuxt 3) | — | tên chunk dạng `Name.<8-hex>.js`, `modulepreload` |
| Router | vue-router (do Nuxt sinh tự động) | — | `path:"/blog/:slug()"` là cú pháp file-based routing của Nuxt |
| Head/SEO | unhead | — | chuỗi `unhead` trong entry |
| HTTP client | `$fetch` (ofetch) | — | `$fetch("/api/home")` |
| Animation | **GSAP** | **3.12.2** | `version:"3.12.2"` |
| Plugin GSAP | ScrollTrigger, ScrollToPlugin, SplitText, DrawSVGPlugin, Flip, CustomEase, Observer | 3.12.2 | `provide:{gsap, ScrollTrigger, ScrollToPlugin, DrawSVGPlugin, SplitText, Flip}` |
| Cuộn mượt | **Lenis** | **1.0.25** | `lenisVersion="1.0.25"` |
| Animation vector | **lottie-web** (renderer SVG) | 5.12.2 | `bodymovin`, `loadAnimation({path:"/Dog.json"})` |
| Slider | **Swiper** (bản Vue) | — | chunk `swiper-vue.*.js`, `effect:"creative"` |
| Validate | yup (chỉ thấy ở trang services) cùng hàm tự viết `validateEmail` | — | chuỗi `yup` trong `services.*.js` |
| CMS / API | **Strapi** (headless) | v4 (suy luận) | CSP chứa `market-assets.strapi.io`, ảnh dạng `/uploads/Name_<10hex>.png` |
| Web server | **Nginx 1.24.0 / Ubuntu** | 1.24.0 | header `Server` |
| Font | Neue Montreal (chữ thân), Bayon (tiêu đề) | — | `font-family` trong CSS |

## Không tìm thấy

Không có dấu vết của Pinia, Axios, Tailwind, Google Analytics/GTM, Facebook Pixel, reCAPTCHA, Stripe hay Three.js. Chuỗi `three` chỉ khớp trong lottie, không phải thư viện Three.js.

→ Site **không dùng tracking bên thứ ba**, và state toàn cục dùng `useState` có sẵn của Nuxt thay cho Pinia.

## Các trang (routes)

```
/                 Trang chủ
/about-us         Giới thiệu
/services         Dịch vụ
/pricing          Bảng giá
/book-now         Form đặt lịch nhiều bước
/contacts         Liên hệ
/faq              Câu hỏi thường gặp
/blog             Danh sách bài viết
/blog/:slug       Chi tiết bài viết
/privacy-policy   Chính sách bảo mật
/terms-of-use     Điều khoản
```

## Kích thước bundle (chưa nén)

| Chunk | Kích thước | Ghi chú |
|---|---|---|
| `lottie.*.js` | 307 KB | lottie-web bản đầy đủ, nặng nhất |
| `entry.*.js` | 270 KB | Vue, Nuxt runtime, GSAP và toàn bộ plugin, Lenis |
| `services.*.js` | 246 KB | trang dịch vụ (có yup) |
| `swiper-vue.*.js` | 160 KB | Swiper; Vite cũng gom vào đây một số helper Vue dùng chung |
| `about-us.*.js` | 68 KB | |
| `index.*.js` (home) | 57 KB | |

Nhận xét: GSAP cùng mọi plugin được đăng ký **toàn cục ngay trong entry**, nên trang nào cũng phải tải. Lottie chỉ được gọi ở trang chủ và about-us nhưng lại bị `modulepreload` ngay từ trang chủ.
