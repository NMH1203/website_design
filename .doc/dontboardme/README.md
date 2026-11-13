# Nghiên cứu kỹ thuật: dontboardme.com

> Ngày phân tích: 2026-11-13
> Phương pháp: tải HTML đã SSR, header HTTP, 43 chunk JS/CSS trong `/_nuxt/` và một phản hồi API công khai, rồi đọc tĩnh mã đã build (minified). **Không** chạy trình duyệt tự động, không đăng nhập, không gửi form.

Đây là website của một dịch vụ trông thú cưng tại nhà (pet sitting / dog walking). Phần kỹ thuật đáng học nằm ở **motion design**: chuyển trang bằng clip-path, chữ hiện theo từng dòng, cuộn mượt và parallax theo chuột, tất cả xây trên **Nuxt 3 + GSAP**.

## Mục lục

| File | Nội dung |
|---|---|
| [01-tong-quan-stack.md](01-tong-quan-stack.md) | Bảng tổng hợp công nghệ, phiên bản và bằng chứng |
| [02-kien-truc-nuxt-strapi.md](02-kien-truc-nuxt-strapi.md) | SSR Nuxt 3, Nitro làm BFF, Strapi CMS, routes, luồng dữ liệu, form đặt lịch |
| [03-animation-gsap-lenis.md](03-animation-gsap-lenis.md) | GSAP plugins, SplitText, ScrollTrigger, chuyển trang, preloader, Lenis |
| [04-ui-lottie-swiper-css.md](04-ui-lottie-swiper-css.md) | Lottie, Swiper (marquee Instagram), hệ CSS, font, breakpoint |
| [05-ha-tang-bao-mat-seo.md](05-ha-tang-bao-mat-seo.md) | Nginx, header bảo mật, CSP, SEO meta |
| [06-ap-dung-cho-portfolio.md](06-ap-dung-cho-portfolio.md) | Cách dựng lại các kỹ thuật này cho portfolio (kèm code mẫu) |

## Tóm tắt một dòng

**Nuxt 3.6.2 (Vue 3.3.4, SSR) → server route `/api/*` của Nitro → Strapi headless CMS (`api.dontboardme.com`)**, chạy sau **Nginx trên Ubuntu**. Animation dùng **GSAP 3.12.2** (ScrollTrigger, ScrollToPlugin, SplitText, DrawSVG, Flip, CustomEase), **Lenis 1.0.25** cho cuộn mượt, **lottie-web** và **Swiper**.

## Mức độ tin cậy

- **Quan sát trực tiếp**: thấy rõ trong header, HTML hoặc chuỗi trong bundle (phiên bản, tên plugin, endpoint, cấu hình).
- **Suy luận**: kết luận rút ra từ mẫu (pattern) chứ chưa xác nhận. Chỗ nào là suy luận sẽ được ghi rõ "(suy luận)".
- **Chưa biết**: mã server Nitro và cấu hình Strapi không công khai nên không xem được.
