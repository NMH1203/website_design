# 04. UI: Lottie, Swiper và hệ CSS

## 1. Lottie (lottie-web 5.12.2)

```js
// Trang chủ: chú chó chạy lặp vô hạn
lottie.loadAnimation({ container: document.getElementById("white-dog__anim"),
  renderer: "svg", loop: true, autoplay: true, path: "/Dog.json" })

// About-us: quả bóng, chỉ chạy một lần khi trang đã sẵn sàng
const anim = lottie.loadAnimation({ container: el, renderer: "svg",
  path: "/Balls.json", loop: false, autoplay: false })
watch(navigationDone, () => anim.play())   // hoặc chờ preloader
```

- File JSON được đặt trong `public/` và tải bằng `path`, không bundle vào JS.
- Dùng renderer `svg` để hình sắc nét ở mọi kích thước.
- Animation one-shot dùng `autoplay: false` và được **đồng bộ với `navigation-state`** (xem file 03).
- Điểm trừ: phải tải trọn bộ lottie-web (307 KB). Có thể thay bằng `lottie_light` hoặc `@lottiefiles/dotlottie-web` để nhẹ hơn.

## 2. Swiper

### Marquee Instagram chạy liên tục

```vue
<Swiper :modules="[Autoplay]" :autoplay="{ delay: 0 }" :speed="2500"
        slides-per-view="auto" space-between="0" loop :allow-touch-move="false">
```

Đây là mẹo quen thuộc: `delay: 0` kết hợp `speed` lớn và một CSS bổ sung (suy luận):
```css
#instagram-slider .swiper-wrapper { transition-timing-function: linear !important; }
```
thì slider trở thành **dải ảnh chạy đều không dừng**, thay cho marquee CSS. Khối Instagram còn có hiệu ứng nghiêng theo chuột (xem file 03, mục 6d).

### Trang About: `effect: "creative"`

Dùng `EffectCreative` để tạo chuyển cảnh tuỳ biến giữa các slide (thẻ chồng lên nhau, có tịnh tiến và xoay).

### Trang chủ: slider testimonials

Bật `loop: true` và `autoplay: true`, kèm modal "đọc tiếp" hiện số thứ tự testimonial đang xem trên tổng số.

## 3. Hệ CSS

- **Scoped CSS của Vue SFC** (`[data-v-xxxx]`). Không dùng Tailwind hay thư viện UI nào.
- **Tiền xử lý**: **(suy luận)** SCSS, vì tên class theo kiểu BEM lồng nhau như `.care-pet__semicircle` và `.cost-of-walking__card`.
- **Design tokens bằng CSS custom properties**:

```css
--c-red, --c-red2, --c-blue, --c-dark-white, --c-ultra-dark...
/* Biến cục bộ của button, cho phép override theo ngữ cảnh: */
--self-bg-color:     var(--bg-btn-color,   var(--c-blue));
--self-text-color:   var(--text-btn-color, var(--c-dark-white));
--self-active-color: var(--active-btn-color, var(--c-ultra-dark...));
```

  Đây là pattern **"biến có giá trị mặc định"**: component đọc `--self-*`, còn component cha chỉ cần đặt `--bg-btn-color` là đổi được màu. Không phải viết thêm class modifier.

- **Font**: `Neue Montreal` (grotesk hiện đại, dùng cho chữ thân, xuất hiện 15 lần) và `Bayon` (display hẹp, dùng cho tiêu đề lớn).
- **Breakpoint**: gần như chỉ có `@media (max-width: 1024px)` (255 lần). Một breakpoint duy nhất chia desktop và tablet/mobile. Ngoài ra có `@media (hover: hover)` (22 lần) để **hover effect chỉ áp dụng trên thiết bị có chuột**.
- Kích thước tính bằng `rem` (ví dụ `0.0625rem` = 1px, `8.5rem`).
- Màu nền nhạt `#ede2e2` và đỏ nhấn `#E33529`.

## 4. Các component tiêu biểu

| Component | Vai trò |
|---|---|
| `CommonPreloader` | bộ đếm 0→100 chạy theo con trỏ |
| `CarePetRightBall`, `ArrowDown` | SVG trang trí có animation |
| `CostWalking` | thẻ bảng giá; trên mobile mỗi thẻ hiện lần lượt |
| `InstagramBlock` | marquee ảnh và nút follow nghiêng theo chuột |
| `Footer` | lấy dữ liệu từ `/api/footer` một lần rồi chia sẻ |
| `progress-line` | thanh tiến trình mảnh (1px, opacity .4) có lớp fill màu `--c-red`, dùng cho slider hoặc form nhiều bước |
| Input có validate | placeholder nổi, kèm icon lỗi SVG |
