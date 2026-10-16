# Bot Telegram bán tài khoản

Bot hiển thị các sản phẩm đang mở bán trên website, cho khách nạp tiền bằng QR và tự động nhận tài khoản sau khi thanh toán thành công.

Dữ liệu Telegram được lưu riêng trên Firebase dưới nhánh `telegramBot/`:

- `telegramBot/users` — ví và thông tin hoạt động của khách Telegram.
- `telegramBot/deposits` — yêu cầu nạp tiền của bot.
- `telegramBot/orders` — đơn hàng và tài khoản đã cấp trong bot.

Các nhánh `users`, `orders` và `deposit_requests` của website không được dùng cho dữ liệu mới của Telegram. Giao diện website cũng lọc các bản ghi Telegram cũ còn sót lại khỏi danh sách người dùng, đơn hàng và duyệt nạp tiền.

## Cấu hình website

Trong biến môi trường Production của Netlify, giữ nguyên các biến đang có và thêm:

```text
TELEGRAM_BOT_SHARED_SECRET=chuoi-bi-mat-it-nhat-32-ky-tu
TELEGRAM_BOT_ADMIN_IDS=123456789,987654321
```

Sau khi đổi biến môi trường, cần deploy lại website để Netlify Function nhận giá trị mới.

`TELEGRAM_BOT_ADMIN_IDS` là Telegram ID của các admin được phép cộng/trừ số dư. Giá trị này phải giống danh sách `ADMIN_TELEGRAM_IDS` (hoặc `ADMIN_TELEGRAM_ID`) của bot. Biến này được kiểm tra trực tiếp tại Netlify, nên người không phải admin không thể tự gọi API tài chính dù biết đường dẫn API.

Mỗi yêu cầu nạp tiền tạo một nội dung chuyển khoản riêng theo dạng `Chuyentien_12345` (5 chữ số). Bot sẽ tự nhận diện nội dung này từ webhook thanh toán và cộng đúng vào ví Telegram tương ứng.

## Cấu hình bot

Tạo file `.env` từ `.env.example` trong cùng thư mục với `bot.py`:

```env
BOT_TOKEN=token-lay-tu-BotFather
ADMIN_TELEGRAM_ID=123456789
WEB_API_BASE_URL=https://ten-mien-website-cua-ban/api
TELEGRAM_BOT_SHARED_SECRET=chuoi-giong-tren-Netlify-it-nhat-32-ky-tu
PORT=8000
```

`BOT_TOKEN` và secret không được đưa lên GitHub hoặc gửi công khai. Nếu token đã xuất hiện trong ảnh/chat, hãy thu hồi và tạo token mới trong BotFather.

## Chạy local

Mở CMD tại thư mục bot và chạy:

```cmd
py -m venv .venv
.venv\Scripts\python.exe -m pip install -r requirements.txt
.venv\Scripts\python.exe bot.py
```

Sau đó gửi `/start` cho bot. Khi chạy local, bot nhận tin nhắn Telegram nhưng webhook SePay không thể gọi vào `localhost` nếu chưa có địa chỉ public.

## Quản trị

Admin dùng đúng số Telegram ID (không phải username) trong `ADMIN_TELEGRAM_ID` rồi gửi `/admin` để xem. Bot cũng hỗ trợ thêm nhiều ID bằng `ADMIN_TELEGRAM_IDS`, ngăn cách bằng dấu phẩy:

- số khách Telegram;
- tổng đơn, đơn hoàn tất và doanh số;
- lượt nạp, tổng tiền đã nạp và số dư khách;
- danh sách khách hoạt động gần đây.

### Lưu nhãn emoji sản phẩm qua các lần deploy

Nhãn emoji, emoji động và sticker gán trong `/admin` được lưu bền vững tại Firebase dưới `telegramBot/productBadges`, ngoài bản cache SQLite. Khi bot khởi động, bot tải nhãn từ Firebase; nếu Firebase chưa có nhãn nhưng file SQLite cũ vẫn còn dữ liệu, bot tự đồng bộ nhãn lên Firebase. Vì vậy tính năng vẫn dùng được khi máy chủ xóa ổ đĩa cục bộ lúc deploy lại.

Khi cập nhật tính năng này, hãy deploy Netlify trước rồi deploy/restart bot. Nếu bản deploy trước đã làm mất cả file SQLite chứa nhãn và Firebase chưa từng lưu các nhãn đó, cần gán lại một lần; các lần deploy sau nhãn sẽ được giữ. Việc lưu bền vững cần Netlify cấu hình Firebase như hiện tại và biến `TELEGRAM_BOT_ADMIN_IDS` chứa UID admin bot.

### Sao lưu dữ liệu SQLite trước khi deploy lại

Không đưa file backup `.db` hoặc file dữ liệu `.sqlite3` lên GitHub. File backup có thể chứa UID, tên và username Telegram, nhãn sản phẩm cùng trạng thái đồng bộ tồn kho. Bot tự tạo bản sao định kỳ; admin cũng có thể gửi `/backup` hoặc bấm **💾 Gửi backup ngay** trong `/admin` để bot gửi file `.db` vào chat Telegram admin. Hãy tải file đó về và cất ở nơi riêng tư, an toàn.

Nếu nền tảng chạy bot có ổ đĩa tạm thời, dữ liệu SQLite có thể mất khi thay máy hoặc deploy lại. Hãy kiểm tra nền tảng có ổ đĩa lưu bền vững được gắn tại `BOT_DATA_DIR` hay không. Khi cần khôi phục, dừng bot, đặt bản sao vào thư mục dữ liệu với tên `telegram_shop.sqlite3`, rồi mới chạy lại bot. Bản `.db` nhận qua Telegram là bản sao SQLite nên có thể dùng để khôi phục; không đổi nội dung hay mở bằng phần mềm chỉnh sửa cơ sở dữ liệu trước khi phục hồi.

### Cộng hoặc trừ số dư khách

Từ `/admin`, bấm **💸 Cộng / trừ số dư** (hoặc gửi lệnh `/money`), rồi lần lượt:

1. gửi Telegram ID của khách;
2. chọn **Cộng tiền** hoặc **Trừ tiền**;
3. nhập số tiền và ghi chú tùy chọn;
4. kiểm tra màn hình xác nhận trước khi bấm cập nhật.

Khách cần từng gửi `/start` để bot có ví Telegram riêng. Mỗi điều chỉnh được ghi lịch sử vào Firebase kèm Telegram ID admin, số dư trước/sau và mã tham chiếu; bấm lại cùng nút xác nhận không cộng/trừ lần hai. Khách cũng nhận thông báo số dư mới sau khi thao tác thành công.

Mỗi đơn hoàn tất được gửi cho khách và bot đồng thời gửi đầy đủ thông tin đơn cùng tài khoản đã cấp cho admin.

Bot hiển thị thông tin hỗ trợ tại `@tai_khoan_xin` (có thể đổi bằng biến `SUPPORT_TELEGRAM`).

## Giá riêng cho Telegram

Trong trang quản trị website, mỗi sản phẩm có thêm trường **Giá riêng trên Telegram (VNĐ)**.
Bỏ trống trường này để bot dùng giá web. Khi có giá riêng, bot Telegram dùng giá đó cả lúc hiển thị và lúc trừ số dư; giá web và tồn kho vẫn giữ độc lập.

Sản phẩm nguồn **Thủ công** chỉ xuất hiện trong danh sách bot khi đã nhập giá riêng Telegram lớn hơn 0. Khi khách đặt mua, bot trừ số dư và số lượng đang bán, lưu đơn riêng trong `telegramBot/orders`, rồi hiển thị mã đơn, tên sản phẩm, giá, ghi chú và nút mở chat admin để khách gửi mã đơn nhận hàng. Bot cũng gửi thông tin đơn cho admin. Các đơn này chờ admin giao hàng thủ công; sản phẩm web nguồn kho và nhà cung cấp tiếp tục theo luồng giao hàng hiện có.

## Cộng tác viên (CTV)

CTV là người được admin duyệt để mua hàng trong một danh mục riêng với giá do admin đặt cho từng sản phẩm, sau đó tự bán lại và tự chịu trách nhiệm với khách của họ. Bot chỉ ghi nhận đơn mua của CTV; không tính hoa hồng giới thiệu và không tự quản lý việc CTV bán lại.

### Duyệt một CTV

1. Khách cần mở bot và gửi `/start` ít nhất một lần để bot ghi nhận tài khoản Telegram.
2. Lấy **Telegram UID** của khách (dãy số, không phải `@username`). Có thể xem UID trong danh sách khách trên trang quản trị hoặc yêu cầu khách gửi UID.
3. Trên website, vào trang quản trị → **Người dùng** → khu vực **Cộng tác viên Telegram**. Nhập UID rồi bấm **Thêm CTV**.
4. Nhờ khách mở lại bot hoặc gửi `/start`. Người đã duyệt sẽ thấy nút **🏪 Kênh CTV**; tài khoản khác không thấy mục này.

Muốn ngừng quyền, bấm **Tạm khóa** ở dòng CTV. Có thể bấm **Mở lại** để cấp quyền lại sau. Khóa CTV không xóa đơn cũ hay thay đổi số dư.

### Đặt giá CTV và mua hàng

1. Mở sửa sản phẩm trên website và nhập **Giá CTV (VNĐ)**. Bỏ trống hoặc đặt giá không hợp lệ sẽ ẩn sản phẩm khỏi danh mục CTV.
2. Giá CTV là số tiền CTV thực trả cho mỗi sản phẩm. Khuyến mãi chung không áp dụng cho giá này. Giá Telegram bán lẻ và giá web vẫn hoạt động theo cấu hình riêng hiện có.
3. Kênh CTV có thể hiển thị sản phẩm giao tự động từ nhà cung cấp API và sản phẩm giao thủ công, nếu sản phẩm đã có giá CTV. Đơn API tiếp tục được cấp tự động; đơn thủ công chờ admin giao như hiện tại. Bot đánh dấu đơn là **CTV** trong lịch sử để admin phân biệt.
4. CTV cần có đủ số dư trong ví Telegram. Họ nạp tiền và mua trong bot như khách thường, nhưng bắt đầu từ **🏪 Kênh CTV** để dùng đúng giá CTV.

Trước khi duyệt CTV, admin nên thống nhất riêng với người đó về giá họ bán lại, cách hỗ trợ khách, hoàn tiền/bảo hành và xử lý đơn giao thủ công. Bot không đặt giá bán lại, không thu tiền từ khách cuối của CTV và không tự quyết định mức lời của CTV.

### Sau khi cập nhật

Phần quản trị và API cần được deploy lại trên Netlify; worker Telegram cần được cập nhật/restart để có nút CTV. Sau đó nhập giá CTV cho sản phẩm, duyệt UID và nhờ CTV gửi `/start` lại. Tính năng dùng nhánh dữ liệu Firebase `telegramBot/ctv`; không cần chuyển dữ liệu đơn hay ví đang có.

## Mã đơn hàng

Đơn mua từ bot dùng mã ngắn dạng `DH-XXXXXXXX`, không chứa Telegram ID của khách.

## Deploy worker

Bot chạy polling nên cần deploy như một worker/service bằng `Procfile`. Vì bot có thêm endpoint `/sepay/webhook`, nền tảng deploy cần cấp HTTPS public URL. Cấu hình webhook SePay tới:

```text
https://dia-chi-public-cua-bot/sepay/webhook
```

Có thể mở đường dẫn trên bằng trình duyệt để kiểm tra bot đã chạy bản hỗ trợ mã `Chuyentien_12345` hay chưa. Kết quả cần có `memoFormat` và `acceptsLegacyMemo`.

Nếu dùng chung một tài khoản ngân hàng cho bot bán tài khoản và bot OTP, SePay chỉ cần trỏ tới một URL. Trên bot nhận webhook chính, đặt `SEPAY_FORWARD_URL` bằng URL public của bot còn lại. Hai bot phải dùng cùng `SEPAY_WEBHOOK_TOKEN` nếu biến này được bật. Mỗi bot chỉ xử lý đúng loại đơn của mình, giao dịch chưa khớp sẽ được chuyển tiếp một lần.

Chỉ dùng bot để phân phối tài khoản mà bạn có quyền bán hoặc phân phối.
# Nguồn DuckyStore Locket Gold

Website hỗ trợ thêm API CTV DuckyStore như một nguồn giao hàng tự động. Trong trang quản trị, vào **Nguồn API**, chọn **DuckyStore — Locket Gold CTV**, đặt tên nguồn và nhập API key lấy từ bot DuckyStore. Key được thử kết nối qua `/api/v1/me`, mã hóa ở backend Netlify và không gửi lại cho trình duyệt. Cấu hình `PROVIDER_MASTER_KEY` trên Netlify vẫn cần có trước khi lưu key.

Khi tạo hoặc sửa sản phẩm, chọn **Nguồn API**, chọn nguồn DuckyStore và gói Gold cần bán. Số lượng khả dụng được tính theo số lượt còn trong ví CTV và số lượt gói cần. Nhập giá bán web, giá Telegram hoặc giá CTV theo kênh muốn mở. Có thể chuyển sản phẩm sang **Thủ công** bất kỳ lúc nào; đơn thủ công tiếp tục chờ admin giao như luồng hiện có.

Trước khi thanh toán, khách nhập username Locket hoặc link profile. Website và bot Telegram chuyển username tới backend; backend mới gọi API cấp Gold và chỉ đánh dấu đơn hoàn tất khi nhận phản hồi thành công. Với đơn Telegram, bot hỏi username trước khi hiển thị phương thức thanh toán và lưu username trong đơn QR đang chờ.

Tài liệu DuckyStore hiện hiển thị hai đường dẫn cấp Gold: `/api/v1/grant` trong code mẫu và `/api/v1/ctv/gold` trong bảng endpoint. Backend thử `/grant` trước và chỉ chuyển sang `/ctv/gold` nếu endpoint đầu trả HTTP 404. Request gửi `testflight: false` để dùng đúng số lượt được ghi ở gói; khách vẫn cần kiểm tra username trước khi xác nhận. API key không cần đặt trong mã nguồn bot hoặc frontend.
