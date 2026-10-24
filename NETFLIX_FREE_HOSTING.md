# Tài liệu API đối tác — Netflix Free

Tài liệu này dành cho đối tác muốn đưa chức năng Netflix Free của website bạn vào website hoặc ứng dụng riêng. Chỉ sử dụng các endpoint public bên dưới; không cần biết hệ thống xử lý nội bộ.

## 1. Thông tin production

API production hiện tại:

~~~text
https://store.kmsdev.id.vn/api/netflix-free/links
~~~

Endpoint đã kiểm tra thực tế và đang trả HTTP 200 với JSON thành công.

Hệ thống phía sau được quản lý nội bộ. Đối tác không cần truy cập, lưu hoặc công khai bất kỳ URL nguồn, token nguồn hay thông tin nhà cung cấp nào.

## 2. Các endpoint công khai

Base URL:

~~~text
https://store.kmsdev.id.vn/api/netflix-free
~~~

| Method | Endpoint | Mục đích |
| --- | --- | --- |
| GET | /links | Lấy danh sách link Netflix khả dụng |
| POST | /claim | Yêu cầu tạo login token |
| GET | /open?token=... | Mở phiên Netflix qua hệ thống public |
| POST | /release | Trả assignment về hệ thống |

Không cần đăng nhập tài khoản KMS Shop để gọi các endpoint này. Request vẫn chịu giới hạn tốc độ ở backend.

## 3. Lấy danh sách link

Request:

~~~http
GET https://store.kmsdev.id.vn/api/netflix-free/links
Accept: application/json
~~~

Ví dụ JavaScript:

~~~js
const response = await fetch(
  'https://store.kmsdev.id.vn/api/netflix-free/links',
  { headers: { Accept: 'application/json' } }
)

const payload = await response.json()
const links = payload.data?.links || []
~~~

Response có dạng:

~~~json
{
  "ok": true,
  "message": "Đã cập nhật danh sách Netflix Free.",
  "data": {
    "links": [
      {
        "id": 760,
        "label": "Link #160476",
        "plan": "Premium",
        "country": "VN",
        "profile_name": "netflix fajar",
        "profile_count": 5,
        "locked_profile_count": 0,
        "video_quality": "UHD",
        "max_streams": 4,
        "active_count": 1,
        "opens_last_60m": 5,
        "available_slots": 3,
        "load_level": "low",
        "has_pc": true,
        "has_mobile": true,
        "expired": false,
        "on_payment_hold": false,
        "extra_member": false,
        "can_regen": true,
        "quality_checked_at": "2026-10-23 23:24:15",
        "quality_method": "membership",
        "member_since": "2025-01-12",
        "next_billing": "N/A",
        "quality_passed": true,
        "available": true
      }
    ],
    "my_assignments": [],
    "rate_used": 0,
    "rate_max": 4
  }
}
~~~

Chỉ cho người dùng claim khi:

~~~js
link.available === true
~~~

Các trường chính:

| Trường | Ý nghĩa |
| --- | --- |
| id | ID dùng để claim link |
| label | Tên hiển thị |
| plan | Gói Netflix |
| country | Quốc gia tài khoản |
| profile_name | Profile chính |
| video_quality | Chất lượng video |
| available_slots | Slot còn trống |
| has_pc / has_mobile | Có hỗ trợ PC/mobile |
| expired | Đã hết hạn |
| on_payment_hold | Đang bị giữ thanh toán |
| quality_passed | Kết quả kiểm tra chất lượng |
| available | Có thể claim hay không |

## 4. Tạo link

Request:

~~~http
POST https://store.kmsdev.id.vn/api/netflix-free/claim
Content-Type: application/json
Accept: application/json
~~~

Body:

~~~json
{
  "cookie_id": 760,
  "type": "pc"
}
~~~

type được hỗ trợ:

~~~text
pc
mobile
~~~

Ví dụ:

~~~js
const response = await fetch(
  'https://store.kmsdev.id.vn/api/netflix-free/claim',
  {
    method: 'POST',
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      cookie_id: 760,
      type: 'pc'
    })
  }
)

const payload = await response.json()
const assignment = payload.data
~~~

Assignment sẵn sàng:

~~~json
{
  "ok": true,
  "message": "Link đã sẵn sàng.",
  "data": {
    "success": true,
    "token": "TOKEN_DO_BACKEND_TRA_VE",
    "cookie_id": 760,
    "type": "pc",
    "expires_at": "2026-10-24T00:30:00+07:00",
    "status": "active",
    "is_pending": false,
    "is_ready": true
  }
}
~~~

Assignment đang chờ:

~~~json
{
  "data": {
    "token": "TOKEN_DO_BACKEND_TRA_VE",
    "cookie_id": 760,
    "type": "pc",
    "expires_at": "2026-10-24T00:30:00+07:00",
    "status": "pending",
    "is_pending": true,
    "is_ready": false
  }
}
~~~

Khi is_pending=true hoặc is_ready=false, không cho mở/copy ngay. Nên gọi lại /links sau khoảng 15 giây để cập nhật my_assignments.

## 5. Mở Netflix

Dùng endpoint redirect của KMS Shop:

~~~text
https://store.kmsdev.id.vn/api/netflix-free/open?token=TOKEN
~~~

Ví dụ:

~~~js
const openUrl =
  'https://store.kmsdev.id.vn/api/netflix-free/open?token=' +
  encodeURIComponent(assignment.token)

window.open(openUrl, '_blank', 'noopener,noreferrer')
~~~

Backend sẽ trả HTTP 302 tới phiên Netflix được cấp. Đối tác không được phụ thuộc vào domain đích sau redirect; chỉ sử dụng endpoint `/open` của website bạn.

Chỉ bật nút mở khi:

~~~js
assignment.token &&
(assignment.status === 'active' || assignment.is_ready === true)
~~~

## 6. Copy link

Link cần copy là URL redirect của KMS Shop:

~~~js
const copyUrl =
  'https://store.kmsdev.id.vn/api/netflix-free/open?token=' +
  encodeURIComponent(assignment.token)

await navigator.clipboard.writeText(copyUrl)
~~~

Nếu trình duyệt không hỗ trợ Clipboard API, dùng fallback textarea. Chỉ copy khi assignment đã sẵn sàng.

## 7. Trả link

Request:

~~~http
POST https://store.kmsdev.id.vn/api/netflix-free/release
Content-Type: application/json
Accept: application/json
~~~

Body:

~~~json
{
  "token": "TOKEN_DO_BACKEND_TRA_VE"
}
~~~

Ví dụ:

~~~js
await fetch(
  'https://store.kmsdev.id.vn/api/netflix-free/release',
  {
    method: 'POST',
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ token: assignment.token })
  }
)
~~~

Sau khi trả link, gọi lại /links để cập nhật slot.

## 8. Luồng tích hợp khuyến nghị

~~~text
1. GET /links
2. Hiển thị các phần tử có available = true
3. Người dùng chọn một link
4. POST /claim với cookie_id và type
5. Nếu pending:
   - khóa Mở link / Copy link
   - gọi lại /links sau 15 giây
6. Nếu active:
   - bật Mở link
   - bật Copy link
7. Khi người dùng kết thúc:
   - POST /release
   - gọi lại /links
~~~

Nên lưu assignment hiện tại trong session người dùng và xóa sau khi release hoặc hết hạn.

## 9. CORS và cách tích hợp

Nếu gọi trực tiếp từ một domain khác, trình duyệt có thể chặn do CORS.

### Khuyến nghị: backend proxy

~~~text
Browser → Backend bên tích hợp → store.kmsdev.id.vn
~~~

Ưu điểm:

- Không phụ thuộc CORS trình duyệt.
- Dễ giới hạn người dùng.
- Dễ lưu assignment theo session.
- Không expose xử lý nội bộ ra client.

### Gọi trực tiếp từ frontend

Chỉ dùng nếu CORS production đã cho phép domain đó. Không dùng mode=no-cors vì response sẽ không đọc được JSON.

## 10. Giới hạn và lỗi

| HTTP/code | Ý nghĩa | Xử lý |
| --- | --- | --- |
| 200 | Thành công | Đọc data |
| 422 | Body/token không hợp lệ | Kiểm tra cookie_id, type, token |
| 429 | Vượt giới hạn request | Chờ rồi thử lại |
| 502 | Hệ thống cấp link không xử lý được | Hiển thị lỗi thân thiện |
| 503 | Dịch vụ Netflix tạm thời không khả dụng | Thử lại sau |

Giới hạn hiện tại:

- GET danh sách: tối đa 20 lần/phút theo client.
- Claim: tối đa 8 lần/giờ theo client.

Không retry POST claim tự động nhiều lần.

## 11. Bảo mật bắt buộc

- Chỉ chạy qua HTTPS.
- Không ghi token vào log, analytics hoặc URL tracking.
- Không gửi token vào chat công khai.
- Không expose URL nguồn, token nguồn, tên nhà cung cấp hoặc thông tin xử lý nội bộ.
- Escape HTML nếu render profile_name hoặc label bằng innerHTML.
- Dùng rel=noopener noreferrer với liên kết mở tab mới.
- Khi assignment hết hạn, xóa state và yêu cầu claim lại.
- Không cho mở/copy khi assignment chưa active.

## 12. Checklist bàn giao

~~~text
[ ] GET https://store.kmsdev.id.vn/api/netflix-free/links trả HTTP 200
[ ] Frontend đọc payload từ data.links
[ ] Chỉ hiển thị link available=true
[ ] Claim gửi đúng cookie_id và type
[ ] Pending khóa Mở/Copy
[ ] Active mới cho Mở/Copy
[ ] Mở link dùng /open?token=...
[ ] Copy link dùng URL production /open?token=...
[ ] Release gửi đúng token
[ ] Có xử lý 422, 429, 502, 503
[ ] Không log hoặc public token
[ ] Nếu khác domain, đã kiểm tra CORS hoặc dùng backend proxy
~~~
