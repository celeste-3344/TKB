# Thời khóa biểu — build & bảo mật

## 1. Cấu trúc repo
- `src/index.html` — mã nguồn dễ đọc, đây là file bạn **sửa** khi cần thay đổi TKB.
- `.github/workflows/deploy.yml` — tự động build + nén (minify) + deploy lên GitHub Pages mỗi khi push vào nhánh `main`.
- `database.rules.json` — Security Rules mẫu cho Firebase Realtime Database (chỉ tham khảo, phải copy tay vào Firebase Console → Realtime Database → Rules).

Bạn **không** commit trực tiếp bản đã "nén" — nó được sinh ra tự động trong Actions và chỉ nằm ở nhánh deploy/artifact, không có trong lịch sử `main`.

## 2. Vì sao trang được nén trông giống ảnh bạn gửi
`deploy.yml` dùng gói `html-minifier-terser` — công cụ dựng bởi các trình biên dịch thật (Terser cho JS, clean-css cho CSS) chạy trên máy chủ GitHub lúc build. Nó gộp toàn bộ HTML/CSS/JS thành vài dòng dài, bỏ hết comment/khoảng trắng — đúng kiểu bạn thấy trong ảnh chụp "Xem nguồn trang".

Mình **không** tự viết một bộ nén tay cho toàn bộ 1700+ dòng JavaScript, vì nén sai một dấu chấm phẩy có thể làm hỏng cả app (đặc biệt phần đồng bộ Firebase). Để build của bạn đáng tin cậy, hãy để một trình phân tích cú pháp JS thật (Terser) làm việc đó — đó chính là việc GitHub Actions đang làm.

**Lưu ý quan trọng:** nén mã nguồn chỉ làm khó người xem lướt qua "View Source", **không phải** là một lớp bảo mật thật sự. Bất kỳ ai rành kỹ thuật vẫn có thể format lại (`prettier`, DevTools "pretty print") để đọc được toàn bộ logic trong vài giây. Coi đây là "làm gọn/khó đọc hơn", không phải "mã hoá" hay "giấu".

## 3. Về API key Firebase — điều cần hiểu trước tiên
`apiKey` trong `firebaseConfig` **không phải** là một secret theo kiểu API key thông thường (như key của OpenAI, Stripe...). Theo đúng thiết kế của Google/Firebase, key này định danh *dự án* Firebase của bạn để trình duyệt biết gửi request tới đâu — nó **luôn luôn** phải nằm trong mã JavaScript chạy trên trình duyệt người dùng, dù bạn có nén, mã hoá base64, hay giấu ở đâu đi nữa, vì trình duyệt bắt buộc phải đọc được nó để gọi Firebase. Không có cách nào deploy một trang tĩnh (GitHub Pages) mà "giấu" được key này khỏi người dùng cuối — kể cả website lớn của Google cũng để lộ y hệt vậy.

**Điều thực sự bảo vệ dữ liệu của bạn là:**

1. **Firebase Security Rules** (file `database.rules.json` ở trên) — quy định ai được đọc/ghi vào đâu. App của bạn đã đăng nhập ẩn danh (`signInAnonymously`) trước khi đọc/ghi, nên rule `auth != null` là hợp lý — chỉ chặn được truy cập "vô danh hoàn toàn", không chặn được người vào thẳng trang của bạn (vì họ cũng tự động được đăng nhập ẩn danh). Nếu muốn chặt hơn, cân nhắc thêm giới hạn theo cấu trúc dữ liệu, giới hạn kích thước ghi, v.v.
2. **Giới hạn key theo tên miền (HTTP referrer)** — vào Google Cloud Console → APIs & Services → Credentials → chọn key `AIzaSy...` → "Set an application restriction" → Websites → thêm đúng domain GitHub Pages của bạn (vd: `tenban.github.io/*`). Việc này ngăn người khác copy key của bạn dùng cho web/app khác.
3. **Authorized domains trong Firebase Auth** — Firebase Console → Authentication → Settings → Authorized domains → thêm domain GitHub Pages, nếu chưa có.
4. *(Tuỳ chọn, chặt hơn)* **Firebase App Check** — thêm một lớp xác minh "request này đến từ đúng app của bạn, không phải script tự động" nếu bạn lo bị spam ghi dữ liệu.

Việc build tự động ở trên (`__FIREBASE_API_KEY__` + GitHub Secret) chỉ giúp: (a) key không nằm trong lịch sử Git từ đầu, (b) tránh GitHub tự động gửi cảnh báo "phát hiện secret bị lộ" khi bạn push — **không** làm key trở nên bí mật hơn với người dùng cuối cùng, vì nó vẫn xuất hiện trong HTML đã build khi bạn "View Source".

## 4. Thiết lập lần đầu
1. Vào repo trên GitHub → Settings → Secrets and variables → Actions → New repository secret:
   - Tên: `FIREBASE_API_KEY`
   - Giá trị: `AIzaSyDrKlBvXTppGDOcYp9KVdmKoAGKXcXbMFQ` (key hiện tại của bạn)
2. Settings → Pages → Build and deployment → Source: chọn **GitHub Actions**.
3. Copy nội dung `database.rules.json` vào Firebase Console → Realtime Database → Rules → Publish.
4. Google Cloud Console → giới hạn key theo domain như mục 3 ở trên.
5. Push code lên nhánh `main` — Action sẽ tự build + deploy.

## 5. Nếu muốn giấu một API key *thật sự* nhạy cảm sau này
Nếu sau này bạn thêm một dịch vụ có key thật sự nhạy cảm (ví dụ key có tính phí theo lượt gọi, hoặc key admin), **không** để nó chạy trực tiếp từ trình duyệt / GitHub Pages — GitHub Pages chỉ phục vụ file tĩnh, không có backend để giữ bí mật. Khi đó bạn cần một lớp trung gian (proxy) như Cloudflare Worker, Vercel/Netlify Function, hay một server nhỏ — nơi key được giữ ở biến môi trường phía server, trình duyệt chỉ gọi vào proxy đó chứ không thấy key gốc.
