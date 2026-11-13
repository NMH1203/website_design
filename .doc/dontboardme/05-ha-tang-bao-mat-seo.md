# 05. Hạ tầng, bảo mật và SEO

## 1. Hạ tầng

| Thành phần | Quan sát |
|---|---|
| Reverse proxy | `Server: nginx/1.24.0 (Ubuntu)` cho cả `dontboardme.com` và `api.dontboardme.com` |
| Frontend | Nuxt (Nitro, Node server), header `x-powered-by: Nuxt` |
| Backend | Strapi tại subdomain `api.` |
| Nén | `Vary: Accept-Encoding` (gzip hoặc brotli) |
| CDN | `cdnURL: ""`, tức là **không dùng CDN**; asset `/_nuxt/` được Nginx/Nitro phục vụ trực tiếp |

**(Suy luận)** Cả hai dịch vụ chạy trên cùng một VPS Ubuntu (có thể dùng PM2 hoặc Docker), Nginx proxy tới hai cổng khác nhau.

`robots.txt`:
```
User-agent: *
Allow: /404
```
Không có `sitemap.xml`. Đây là một thiếu sót SEO.

## 2. Header bảo mật (trên `api.dontboardme.com`, do Strapi sinh)

Strapi dùng middleware `koa-helmet`, nên có sẵn:

```
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none';
  img-src 'self' data: blob: https://market-assets.strapi.io; frame-ancestors 'self'; ...
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
Referrer-Policy: no-referrer
X-Permitted-Cross-Domain-Policies: none
```

Trái lại, **site Nuxt chính không có các header này** (chỉ có `x-powered-by`). Nếu tự làm, bạn có thể dùng module `nuxt-security`, và nên tắt `x-powered-by` để không lộ framework.

## 3. SEO

- SSR nên crawler đọc được nội dung đầy đủ ngay trong HTML.
- Composable `usePageMeta(seo)` lấy dữ liệu SEO từ CMS rồi sinh các thẻ sau:

```js
useHead({
  title: seo.meta.title,
  meta: [
    { name: "description",       content: seo.meta.description },
    { name: "og:title",          content: seo.og.title },
    { name: "og:description",    content: seo.og.description },
    { name: "og:image",          content: seo.og.image.url },
    { name: "twitter:title",     content: seo.og.title },
    { name: "twitter:image",     content: seo.og.image.url },
  ]
})
```

**Lỗi nhỏ đáng chú ý** (để không lặp lại khi tự làm):
- Thẻ OpenGraph đúng chuẩn phải dùng `property="og:title"`, nhưng site lại dùng `name="og:title"`.
- `<title>Main</title>`: tiêu đề trang chủ chỉ là "Main", không có thương hiệu hay từ khoá.
- Viewport đặt `maximum-scale=1`, chặn người dùng phóng to, gây hại cho accessibility.
- Không có sitemap và thẻ canonical.

Gợi ý khi làm với Nuxt: dùng `useSeoMeta({ ogTitle, ogImage, twitterCard })` để Nuxt tự sinh đúng thuộc tính, kèm module `@nuxtjs/sitemap`.

## 4. Hiệu năng (nhận xét)

| Ưu điểm | Nhược điểm |
|---|---|
| SSR và CSS critical được inline | GSAP cùng 7 plugin nằm ngay trong entry (~270 KB chưa nén) |
| `modulepreload` cho các chunk của trang | lottie-web 307 KB bị preload từ trang chủ |
| Prefetch link bằng IntersectionObserver (mặc định của NuxtLink) | Không dùng CDN, ảnh không qua `@nuxt/image` |
| Không có script tracking bên thứ ba | |
