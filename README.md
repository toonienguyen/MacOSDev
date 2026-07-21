# macOS 27 — Ghi chú deploy & tùy biến

## 1. Cấu trúc thư mục bắt buộc khi deploy

App là **static site** (build từ Vite/React), không có backend. Chỉ cần host bằng bất kỳ static server nào (Cloudflare Pages, `python3 -m http.server`, v.v.) — không chạy được bằng cách mở trực tiếp `file://` vì `<script type="module">` bị chặn CORS.

```
your-site/
├── index.html
├── robots.txt
├── wallpaper-tahoe-day.jpg
├── wallpaper-glass-light.jpg
├── wallpaper-glass-dark.jpg
├── wallpaper-aurora.jpg
├── wallpaper-bigsur.jpg
├── photo-1.jpg  ... photo-8.jpg
├── track-1.mp3  ... track-4.mp3
├── cover-1.jpg  ... cover-4.jpg
├── podcast-cover.jpg
└── assets/
    ├── index-Bfk0NWYJ.js
    └── index-BqrKPhU7.css
```

⚠️ Tất cả file media nằm **cùng cấp** với `index.html` (KHÔNG để trong `assets/`). Tên file phải khớp **chính xác** (chữ thường, dấu gạch ngang, đúng đuôi file) vì code gọi thẳng theo path cố định.

---

## 2. Bảng tên file bắt buộc

| Loại | Tên file | Số lượng |
|---|---|---|
| Wallpaper | `wallpaper-tahoe-day.jpg`, `wallpaper-glass-light.jpg`, `wallpaper-glass-dark.jpg`, `wallpaper-aurora.jpg`, `wallpaper-bigsur.jpg` | 5 |
| Photos | `photo-1.jpg` → `photo-8.jpg` | 8 |
| Music (audio) | `track-1.mp3` → `track-4.mp3` | 4 |
| Music (cover) | `cover-1.jpg` → `cover-4.jpg` | 4 |
| Podcast | `podcast-cover.jpg` | 1 |

Nếu thiếu file → Wallpaper tự fallback sang **gradient CSS** (không trắng màn hình); Photos/Music sẽ vỡ ảnh / không phát được ở đúng chỗ thiếu.

---

## 3. Giới hạn quan trọng: dữ liệu bị HARDCODE

Danh sách Photos, Music, Podcast là **seed data cứng** trong file JS đã build (không đọc động theo thư mục).

- ✅ **Thay được nội dung**: đè ảnh/nhạc mới lên đúng tên file cũ (`photo-3.jpg`, `track-2.mp3`...) → app tự hiển thị/phát nội dung mới.
- ❌ **Không thêm được số lượng**: không thể thêm `photo-9.jpg`, `track-5.mp3`... vì code không có object tương ứng gọi tới các file đó.
- ❌ **Không đổi được tên bài hát / ca sĩ / metadata** chỉ bằng cách đổi file — các thông tin này là **text hardcode riêng** trong file JS, tách biệt với file audio/ảnh thật.

Muốn mở rộng số lượng hoặc sửa cấu trúc dữ liệu → bắt buộc phải có **source code gốc** (chưa build/minify), sửa mảng dữ liệu, build lại bằng Vite.

Riêng **Wallpaper** có ngoại lệ: có sẵn nút **"Add from Photos..."** trong Settings, cho phép user tự upload ảnh riêng (giới hạn 700KB, lưu local trong trình duyệt) — đây là cách hợp lệ duy nhất để thêm wallpaper tùy chỉnh mà không cần sửa code.

---

## 4. Bảng metadata Music / Podcast hiện tại (để tra khi cần sửa code)

| ID | File | Title | Artist | Album | Genre | Cover |
|---|---|---|---|---|---|---|
| t1 | `track-1.mp3` | Neon Skyline — Single | *(chưa xác định)* | — | Synthwave | `cover-1.jpg` |
| t2 | `track-2.mp3` | Golden Hour | Café Mono | Golden Hour | Lo-Fi | `cover-2.jpg` |
| t3 | `track-3.mp3` | Blue Note Sessions | The Meridian Trio | Blue Note Sessions | Jazz | `cover-3.jpg` |
| t4 | `track-4.mp3` | Drift | Isla Wave | Drift | Ambient | `cover-4.jpg` |

### Podcast "Night Circuit" — dùng CHUNG file audio với Music

Podcast không có file `.mp3` riêng — mỗi "tập" chỉ là metadata khác (tên tập, mô tả) bọc quanh **cùng 4 file `track-*.mp3`** phía trên.

| Tập podcast | File dùng chung | Ghi chú |
|---|---|---|
| Night Circuit 041: Midnight Drift | `track-4.mp3` | Ambient, nhắc tới Isla Wave |
| Night Circuit 040: Golden Hour Mix | `track-2.mp3` | Lo-fi, nhắc tới Café Mono |
| *(tập khác)* | `track-1.mp3` | Synthwave, nhắc tới "Vector Fields" |

→ Nếu đè nội dung `track-1.mp3` bằng file nhạc khác, **cả Music app lẫn Podcast app** đều phát ra nội dung mới đó (vì trỏ chung 1 file vật lý), dù text hiển thị (tên bài/tên tập) vẫn giữ nguyên như cũ.

---

## 5. Mẹo tra cứu nhanh trong file JS đã minify (2.4MB)

Đừng mở file bằng mắt — dùng `grep` để nhảy thẳng tới đoạn cần, ví dụ:

```bash
# Tìm context quanh 1 file cụ thể
grep -oaE '.{50}track-1\.mp3.{600}' index-*.js

# Tìm tất cả domain app gọi ra ngoài (network calls)
grep -oE 'https?://[a-zA-Z0-9._-]+' index-*.js | sort -u

# Đếm số lần 1 file được tham chiếu (biết nó dùng ở mấy chỗ)
grep -oaE 'src:"/track-[0-9]\.mp3"' index-*.js | sort | uniq -c
```

---

## 6. Các API bên ngoài app có gọi (ngoài file media local)

App 100% chạy local trong trình duyệt, chỉ gọi ra ngoài cho vài tính năng cụ thể:

| API | Phục vụ app | Ghi chú độ bền |
|---|---|---|
| `api.open-meteo.com`, `geocoding-api.open-meteo.com` | Weather | Free + có gói trả phí song song → khá bền |
| `nominatim.openstreetmap.org`, `router.project-osrm.org`, `server.arcgisonline.com` | Maps | Cộng đồng/công ty lớn đứng sau, khá bền, có rate limit |
| `api.dictionaryapi.dev` | Dictionary | Dự án cá nhân + donate, không có SLA chính thức |
| `commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4` | Video demo | ⚠️ **Đã chết** (403 AccessDenied) — bucket demo cũ của Google không còn public |

Nếu các API trên có ngưng hoạt động, chỉ ảnh hưởng đúng 3 tính năng (Weather, Maps, Dictionary) — toàn bộ phần lõi OS giả lập (Finder, Notes, Calculator, TextEdit, Terminal, Settings, Mail, Calendar, Messages, Preview, Photos, Music, App Store...) vẫn chạy bình thường vì không phụ thuộc backend hay API ngoài.

---

## 7. Đổi tên user mặc định "kimi" → tên/iCloud của bạn

Tên user `"kimi"` (kèm email `kimi@icloud...`) bị **hardcode rải rác 6-8 chỗ** trong file JS, không có UI/Settings nào để tự đổi trong app. Các chỗ dùng tên này: màn hình khóa, cấu trúc thư mục ảo (`/Users/kimi/...`), Terminal (`whoami`), Activity Monitor (list process), Contacts app, tài khoản iCloud hiển thị trong Settings.

Vì cùng dùng chung 1 chuỗi gốc, chỉ cần **Find & Replace toàn bộ** là đổi đồng bộ hết, không cần sửa từng chỗ riêng lẻ.

### Cách 1 — Dòng lệnh (Git Bash / WSL / Linux / Mac)

Vào đúng thư mục chứa `index-Bfk0NWYJ.js`, chạy:

```bash
sed -i 's/"kimi"/"TenBanMuon"/g; s/kimi@icloud/TenBanMuon@icloud/g' index-Bfk0NWYJ.js
```

Thay `TenBanMuon` bằng tên bạn muốn.

### Cách 2 — Notepad có sẵn (Windows, không cần cài gì thêm)

1. Mở `index-Bfk0NWYJ.js` bằng Notepad bình thường (file lớn/1 dòng dài, cuộn hơi mỏi mắt nhưng Find & Replace vẫn hoạt động tốt)
2. `Ctrl + H` → tab Replace
3. Find: `"kimi"` (có 2 dấu ngoặc kép) → Replace: `"TenBanMuon"` (có 2 dấu ngoặc kép) → **Replace All**
4. Làm thêm lần nữa: Find: `kimi@icloud` → Replace: `TenBanMuon@icloud` → **Replace All**
5. `Ctrl + S` lưu lại

*(Notepad++ chỉ tiện hơn xíu khi mở file lớn, không bắt buộc — Notepad mặc định của Windows làm được y hệt.)*

### Sau khi đổi

- Đè file đã sửa vào `assets/index-Bfk0NWYJ.js` cũ, deploy lại (hoặc chạy local server lại)
- Nên `Ctrl+F` tìm lại `"kimi"` lần nữa để chắc chắn không sót chỗ nào

### ⚠️ Lưu ý an toàn

- Tên thay thế viết **liền, không dấu tiếng Việt, có thể có khoảng trắng** nhưng tránh ký tự đặc biệt — vì đây là chuỗi trong code JS, ký tự lạ có thể phá cú pháp gây lỗi trắng trang.
- Không tự ý `sed`/Replace All chuỗi `kimi` KHÔNG có dấu ngoặc kép bao quanh — có thể dính nhầm vào phần code không liên quan tên user, gây lỗi ngoài ý muốn.
