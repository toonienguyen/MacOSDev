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

### Music app — mỗi track có field `title` và `artist` TÁCH RIÊNG (dễ Find & Replace)

| ID | File | Title | Artist | Album | Genre | Cover |
|---|---|---|---|---|---|---|
| t1 | `track-1.mp3` | Neon Skyline | Vector Fields | Neon Skyline — Single | Synthwave | `cover-1.jpg` |
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

---

## 8. Ý tưởng: biến 4 track thành 4 liên khúc/album, host nhạc trên CDN (R2)

Vì code chỉ quan tâm đúng 4 file `.mp3` tồn tại (`track-1.mp3` → `track-4.mp3`), không quan tâm bên trong có bao nhiêu bài — mỗi file có thể là **1 liên khúc ghép nhiều bài nối liền** thay vì 1 bài đơn. Cách này mở rộng được nội dung mà **không cần sửa code**, an toàn tuyệt đối, không rủi ro gãy app như việc chèn logic fetch động (đã thử và gãy — xem bài học ở cuối mục này).

### Bước 1 — Ghép nhạc thành liên khúc

Dùng Audacity hoặc ffmpeg để nối nhiều file mp3 thành 1:
```bash
ffmpeg -i "concat:bai1.mp3|bai2.mp3|bai3.mp3" -acodec copy track-1.mp3
```
Các file mp3 đầu vào nên cùng bitrate/sample rate để tránh lỗi khi nối kiểu `concat:` protocol; nếu khác nhau, cân nhắc dùng filter `concat` của ffmpeg thay vì cách trên.

Lặp lại tạo `track-2.mp3`, `track-3.mp3`, `track-4.mp3` — mỗi file là 1 liên khúc/album riêng.

Progress bar trong app Music/Podcasts vẫn chạy đúng theo tổng thời lượng file mới (không hardcode số giây, code tự đọc `duration` thật từ file audio).

### Bước 2 — Vì liên khúc nặng hơn nhiều (có thể 30–50MB/file thay vì 3–5MB), host trên CDN thay vì để chung với code

Upload 4 file `track-*.mp3` lên **Cloudflare R2 bucket** (bật public access), lấy 4 URL dạng:
```
https://pub-xxxxx.r2.dev/track-1.mp3
https://pub-xxxxx.r2.dev/track-2.mp3
https://pub-xxxxx.r2.dev/track-3.mp3
https://pub-xxxxx.r2.dev/track-4.mp3
```

### Bước 3 — Trong file JS, Find & Replace (Notepad, giống hệt cách đổi tên "kimi" ở mục 7)

```
Find:    "/track-1.mp3"
Replace: "https://pub-xxxxx.r2.dev/track-1.mp3"
```
Lặp lại cho `track-2`, `track-3`, `track-4`. Mỗi track chỉ có đúng 1 chỗ dùng `src:"/track-X.mp3"` trong code, nên chỉ cần 1 lần Replace All mỗi file.

### Cover ảnh — KHÔNG cần đổi gì

`cover-1.jpg` → `cover-4.jpg` và `podcast-cover.jpg` vẫn giữ nguyên tên file, đặt cùng cấp `index.html` như bình thường (xem mục 2) — không cần đưa lên CDN, không cần sửa code.

### Muốn đổi tên hiển thị (title) VÀ tên ca sĩ (artist) — hướng dẫn cụ thể

Đính chính: `artist` là **field riêng biệt**, tách bạch hoàn toàn khỏi `title`, không nằm trong mô tả — Find & Replace rất an toàn và đơn giản, y hệt cách đổi tên "kimi" ở mục 7. Copy sẵn bảng dưới, đổi cả 8 dòng nếu muốn thay hết 4 track:

| Track | Find | Replace ví dụ |
|---|---|---|
| t1 title | `title:"Neon Skyline"` | `title:"Liên Khúc 1"` |
| t1 artist | `artist:"Vector Fields"` | `artist:"Tên bạn"` |
| t2 title | `title:"Golden Hour"` | `title:"Liên Khúc 2"` |
| t2 artist | `artist:"Café Mono"` | `artist:"Tên bạn"` |
| t3 title | `title:"Blue Note Sessions"` | `title:"Liên Khúc 3"` |
| t3 artist | `artist:"The Meridian Trio"` | `artist:"Tên bạn"` |
| t4 title | `title:"Drift"` | `title:"Liên Khúc 4"` |
| t4 artist | `artist:"Isla Wave"` | `artist:"Tên bạn"` |

⚠️ Lưu ý khi Find — **đây không phải rủi ro hiếm, mà xảy ra ở 3/4 track**: `title` và `album` trùng chuỗi y hệt nhau ở t2, t3, t4 (`"Golden Hour"`, `"Blue Note Sessions"`, `"Drift"` đều là cả `title` lẫn `album`); riêng t1 thì `title:"Neon Skyline"` còn là **một phần chuỗi con** nằm bên trong `album:"Neon Skyline — Single"`. Gõ Find thiếu tiền tố field (chỉ gõ `"Drift"` thay vì `title:"Drift"`) sẽ khiến Replace All đổi nhầm cả `album`, hoặc dính vào giữa chuỗi khác như t1.

**Quy tắc an toàn bắt buộc**: luôn gõ Find kèm **tên field đứng ngay trước** (`title:"..."` hoặc `artist:"..."`), không bao giờ gõ mỗi phần chuỗi tên không kèm ngoặc kép. Bảng trên đã viết đúng chuẩn này — copy y nguyên, đừng rút gọn bớt phần `title:`/`artist:` khi gõ vào Notepad.

⚠️ **Thêm 1 lớp trùng lặp khác, đã xác minh cụ thể**: tên ca sĩ không chỉ trùng nội bộ Music app, mà 3/4 tên còn **lặp lại y hệt bên trong `desc` của Podcast episodes** (dùng chung file audio nhưng object hoàn toàn khác — xem phần dưới):

| Artist | Xuất hiện lần 2 ở đâu |
|---|---|
| `Vector Fields` | `desc:"...a synthwave set featuring Vector Fields."` (Night Circuit 042) |
| `Café Mono` | `desc:"...courtesy of Café Mono."` (Night Circuit 040) |
| `Isla Wave` | `desc:"...with Isla Wave in the mix."` (Night Circuit 041) |

Vì cách Find `artist:"Café Mono"` (có tiền tố `artist:`) đã đủ đặc hiệu, **không trùng** với các câu `desc` ở trên (những câu đó không có chữ `artist:` đứng trước) — nên làm đúng theo bảng Find/Replace đã cho là an toàn, sẽ không vô tình đổi luôn phần mô tả podcast. Nhưng nếu bạn tính rút gọn Find chỉ còn `"Café Mono"` (bỏ tiền tố `artist:`) để "cho nhanh", **Replace All sẽ đổi luôn cả câu mô tả podcast** — đây chính là lý do quy tắc "luôn giữ tiền tố field" ở trên là bắt buộc, không phải khuyến nghị tùy chọn.

Việc đổi title/artist ở đây **độc lập hoàn toàn** với việc đổi file audio (CDN) ở các bước trên — làm cả hai nếu muốn tên hiển thị khớp đúng với nhạc mới, hoặc chỉ làm 1 trong 2 nếu chỉ cần đổi 1 phần.

### Về field `album` — chỉ là nhãn hiển thị, tùy chọn, không ảnh hưởng số lượng file thật

App tự động **gom nhóm track theo chuỗi `album:"..."` trùng nhau** để tạo view Albums — không có danh sách Album riêng biệt nào phải khai báo trước. Cơ chế: track nào có cùng chuỗi `album` y hệt thì tự xếp chung 1 album.

- Muốn 1 album có nhiều bài: đặt cùng 1 chuỗi `album:"..."` cho nhiều track (VD cả 4 track đều `album:"Ten Album"` → ra 1 album 4 bài)
- Muốn mỗi track 1 album riêng: giữ mặc định, mỗi track 1 tên `album` khác nhau (như hiện tại)

Dù chọn cách nào, **vẫn chỉ có đúng 4 file `.mp3` vật lý** — field `album` chỉ đổi cách hiển thị nhóm trên UI, không tạo thêm hay giảm bớt file thật, không ảnh hưởng việc phát nhạc. Tùy gu cá nhân, không có cách nào "đúng/sai".

### ⚠️ Lưu ý: đây là Music app, khác với Podcast episodes

Podcast "Night Circuit"/"The Gradient Hour" dùng chung file audio nhưng có **object hoàn toàn riêng**, với field `title` (tên tập) và `desc` (mô tả tập) — không có field `artist` tách riêng, tên nghệ sĩ nằm lồng trong câu `desc`. Ví dụ: `desc:"Lo-fi warmth to close out the season, courtesy of Café Mono."`. Nếu muốn đổi tên xuất hiện trong mô tả podcast, phải sửa nguyên câu `desc` đó, không có field riêng để Find & Replace gọn như bên Music.

### ⚠️ Bài học từ lần thử "fetch động" — vì sao chọn cách Find & Replace thay vì thêm logic mới

Đã thử hướng nâng cao hơn: viết thêm hàm `fetch('/photos.txt')` để tự động đọc số lượng ảnh không giới hạn từ 1 file txt, chèn trực tiếp vào file JS đã minify. Cách này **bị lỗi cú pháp 2 lần liên tiếp** — nguyên nhân cuối cùng xác định là do quá trình copy code qua nhiều bước trung gian đã vô tình cắt đứt một chuỗi template literal (dấu backtick) trong code gốc, không phải do logic sai.

**Kết luận rút ra**: với file JS 2.4MB đã minify, không có source gốc, mọi thao tác **thêm logic mới** (function, fetch, biến mới) đều rủi ro cao vì khó trích xuất/dán lại chính xác 100% ký tự đặc biệt. Thao tác **chỉ đổi giá trị chuỗi có sẵn** (Find & Replace URL, tên) an toàn hơn rất nhiều và nên là lựa chọn mặc định, trừ khi thực sự cần thay đổi cấu trúc/số lượng (trường hợp đó cần source code gốc chưa build — xem mục 3).

---

## 9. Đổi chữ cái đầu (avatar tròn) ở màn hình khóa

Sau khi đổi tên user (mục 7), tên hiển thị ("toonie" chẳng hạn) đổi đúng, nhưng **chữ cái trong avatar tròn xanh không tự đổi theo** — vì đây là 1 chuỗi hardcode **độc lập, tách rời hoàn toàn** khỏi tên user, không tự tính từ ký tự đầu như nhiều app thật thường làm.

### Vị trí chính xác trong code

```js
style:{background:"linear-gradient(160deg,#5EB3F8,#1463E8)", ...},
children:e.jsx("span",{..., className:"text-white text-[40px] font-semibold", children:"K"})
```

### Cách đổi — ĐÃ XÁC MINH số lần trùng lặp trước khi khuyến nghị

| Cách Find | Số lần khớp trong file | Có an toàn dùng không |
|---|---|---|
| `"K"` (chỉ 1 ký tự, có ngoặc kép) | **9 lần** | ❌ Tuyệt đối không dùng — trùng lung tung khắp file |
| `children:"K"` | **3 lần** | ❌ Không dùng — trùng thêm avatar Mail/Contacts của người liên hệ khác tên K, không liên quan user hệ thống |
| `text-[40px] font-semibold",children:"K"` | **1 lần, đúng chỗ LockScreen** | ✅ Dùng cách này |

**Find & Replace đúng (Notepad, `Ctrl+H`):**
```
Find:    text-[40px] font-semibold",children:"K"
Replace: text-[40px] font-semibold",children:"T"
```
(Đổi `"T"` thành chữ cái đầu bạn muốn, viết hoa, chỉ 1 ký tự — vì UI thiết kế cho avatar 1 chữ cái, viết nhiều ký tự vào đây có thể vỡ layout hình tròn.)

### Bài học nhỏ rút ra thêm

Không phải mọi phần hiển thị "liên quan tới tên user" đều tự động đồng bộ với nhau trong code đã minify — mỗi chỗ hiển thị có thể là biến động (tự cập nhật theo tên) hoặc chuỗi tĩnh hardcode riêng (như trường hợp avatar này), tùy cách tác giả gốc viết. Luôn xác minh số lần khớp thật (`grep -oa "..." file.js | wc -l`) trước khi Replace All, đừng giả định 1 chuỗi ngắn chỉ xuất hiện đúng 1 chỗ.
