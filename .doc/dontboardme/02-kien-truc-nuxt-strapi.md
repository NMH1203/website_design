# 02. Kiến trúc: Nuxt 3 SSR + Nitro BFF + Strapi

## Sơ đồ

```
Trình duyệt
   │  GET /about-us              (HTML đã render sẵn)
   ▼
Nginx (Ubuntu) ──► Nuxt 3 / Nitro server (Node)
                       │  server/api/*.ts      (suy luận)
                       │  gọi runtimeConfig.public.apiUrl
                       ▼
               Strapi CMS  https://api.dontboardme.com
                       │
                       └─ /uploads/*  (ảnh, ảnh OpenGraph)
```

## 1. SSR và hydrate

- HTML trả về đã có nội dung đầy đủ (377 KB), kèm `<script id="__NUXT_DATA__">` chứa payload, và client hydrate từ payload này nên không phải gọi API lại.
- `window.__NUXT__.config`:

```js
{ public: { apiUrl: "https://api.dontboardme.com" },
  app: { baseURL: "/", buildAssetsDir: "/_nuxt/", cdnURL: "" } }
```

→ URL của Strapi được để ở `runtimeConfig.public`, ai cũng xem được. Điều này bình thường vì đó chỉ là URL công khai.

- CSS critical của từng component được **inline** vào `<style>` trong HTML (ví dụ `.toys-wrap[data-v-010466b5]{...}`), giúp tránh FOUC.

## 2. Nitro làm BFF (Backend-for-Frontend)

Client không gọi Strapi trực tiếp mà gọi **route nội bộ** `/api/*` cùng origin:

| Endpoint | Dùng ở |
|---|---|
| `GET /api/home` | trang chủ |
| `GET /api/about-us` | about |
| `GET /api/service`, `/api/service-list` | services, book-now |
| `GET /api/pricing` | pricing |
| `GET /api/faq` | faq |
| `GET /api/contact` | contacts |
| `GET /api/footer` | layout chung |
| `GET /api/blog`, `/api/blog/:slug` | blog |
| `GET /api/privacy-terms` | privacy, terms |
| `POST /api/form/book` | form đặt lịch |

Thử gọi `GET https://dontboardme.com/api/home` thì nhận về JSON **đã được làm phẳng**:

```json
{ "main": { "titleLeft": "...", "titleRight": "...", "smallDescription": "...",
            "middleBlock": {...}, "testimonials": [...], "seo": {...} },
  "priceList": [...] }
```

JSON gốc của Strapi v4 có dạng `{ data: { id, attributes: {...} } }`. Việc dữ liệu ở đây không còn lớp `attributes` cho thấy **(suy luận)** file `server/api/home.get.ts` đã gọi Strapi kèm `populate`, rồi **chuẩn hoá** dữ liệu trước khi trả về. Cách làm này có ba lợi ích:

1. Không lộ cấu trúc và query `populate` của Strapi ra client.
2. Gộp nhiều collection vào một response, như `main` và `priceList` ở trên.
3. Có thể thêm cache hoặc token phía server.

## 3. Mẫu tải dữ liệu trong page

Mã sau khi build (đã đặt lại tên biến cho dễ đọc):

```js
const { data } = await useAsyncData("home-page", () => $fetch("/api/home"))
if (!data.value) throw createError({ statusCode: 500, statusMessage: "Error Getting Data", fatal: true })
usePageMeta(data.value.main.seo)          // composable tự viết → useHead
const footer = useState("footer-data-common")
```

- `useAsyncData` với key cố định nên dữ liệu được SSR, lưu vào payload rồi tái sử dụng khi hydrate.
- Lỗi được đẩy sang trang error của Nuxt bằng `createError({ fatal: true })`.
- Dữ liệu footer chỉ fetch **một lần** ở layout, sau đó chia sẻ qua `useState`.

## 4. State toàn cục bằng `useState` (không dùng Pinia)

| Key | Ý nghĩa |
|---|---|
| `preloader-state` | preloader đã chạy xong hay chưa |
| `animate-page-while-preloader` | cho phép animation của trang bắt đầu khi preloader sắp kết thúc |
| `navigation-state` | `false` trong lúc chuyển trang; `true` khi đã xong, dùng để kích hoạt animation của trang mới |
| `footer-data-common` | dữ liệu footer |

## 5. Form đặt lịch `/book-now` (nhiều bước)

- Có **6 bước** (hằng `b=6`). Bấm Enter thì sang bước tiếp, nút Back gọi `router.back()`.
- Ở bước cuối, form gửi request như sau:

```js
await $fetch("/api/form/book", {
  method: "POST",
  body: { data: {
    one_service: serviceId,
    Pet_Name, How_long, When_to_meet,   // ngày được đổi sang định dạng Mỹ MM/DD/YYYY
    Message, Client_Name, Phone_number, Email
  }},
  query: { service: <tên dịch vụ> }
})
```

- Body có dạng `{ data: {...} }`, khớp với API tạo entry của Strapi v4. Tên field viết hoa kiểu `Pet_Name` cho thấy đó là tên field trong Content-Type Builder của Strapi.
- Helper `formattingDateToUS`: `"2026-11-13"` thành `"11/13/2026"`.
- Validate email bằng hàm tự viết `validateEmail`. Ô input lỗi hiện icon SVG màu đỏ `#E33529`.
- Không thấy captcha. **(Suy luận)** Việc chống spam, nếu có, được làm phía server.

## 6. Bài học kiến trúc

- **Headless CMS và Nuxt SSR** là combo phổ biến cho site marketing: khách hàng tự sửa nội dung và SEO trên Strapi, còn dev lo phần giao diện và motion.
- **Đặt BFF giữa client và CMS** giúp giữ token, ẩn cấu trúc CMS và trả dữ liệu đúng dạng UI cần.
- SEO của từng trang (`seo.meta`, `seo.og`) cũng do CMS quản lý.
