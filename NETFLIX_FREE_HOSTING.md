# Hướng dẫn đấu API Netflix Free

## Base URL

~~~text
https://store.kmsdev.id.vn/api/netflix-free
~~~

## 1. Lấy danh sách link

~~~http
GET /links
Accept: application/json
~~~

JavaScript:

~~~js
const res = await fetch(
  'https://store.kmsdev.id.vn/api/netflix-free/links'
)

const json = await res.json()
const links = json.data?.links || []
~~~

Chỉ cho phép chọn những link có:

~~~js
link.available === true
~~~

Các trường thường dùng:

~~~text
id
label
plan
country
profile_name
video_quality
available_slots
available
~~~

## 2. Tạo link đăng nhập

~~~http
POST /claim
Content-Type: application/json
~~~

Body:

~~~json
{
  "cookie_id": 760,
  "type": "pc"
}
~~~

JavaScript:

~~~js
const res = await fetch(
  'https://store.kmsdev.id.vn/api/netflix-free/claim',
  {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Accept': 'application/json'
    },
    body: JSON.stringify({
      cookie_id: 760,
      type: 'pc'
    })
  }
)

const json = await res.json()
const assignment = json.data
~~~

Giá trị type hỗ trợ:

~~~text
pc
mobile
~~~

## 3. Kiểm tra trạng thái

Chỉ cho phép mở/copy khi:

~~~js
assignment?.token &&
(
  assignment.status === 'active' ||
  assignment.is_ready === true
)
~~~

Nếu is_pending là true, chờ khoảng 15 giây rồi gọi lại GET /links.

## 4. Mở link Netflix

~~~js
const url =
  'https://store.kmsdev.id.vn/api/netflix-free/open?token=' +
  encodeURIComponent(assignment.token)

window.open(url, '_blank', 'noopener,noreferrer')
~~~

Endpoint open tự xử lý chuyển người dùng tới phiên Netflix. Không tự nối token tới domain khác.

## 5. Copy link

~~~js
const url =
  'https://store.kmsdev.id.vn/api/netflix-free/open?token=' +
  encodeURIComponent(assignment.token)

await navigator.clipboard.writeText(url)
~~~

Nếu Clipboard API không hỗ trợ, dùng fallback textarea.

## 6. Trả link

~~~http
POST /release
Content-Type: application/json
~~~

Body:

~~~json
{
  "token": "TOKEN_NHAN_DUOC_TU_CLAIM"
}
~~~

JavaScript:

~~~js
await fetch(
  'https://store.kmsdev.id.vn/api/netflix-free/release',
  {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Accept': 'application/json'
    },
    body: JSON.stringify({
      token: assignment.token
    })
  }
)
~~~

Sau khi trả link, gọi lại GET /links.

## Luồng tích hợp

~~~text
GET /links
  ↓
Người dùng chọn link
  ↓
POST /claim
  ↓
Nếu pending: chờ và gọi lại /links
  ↓
Nếu active: cho Mở link hoặc Copy link
  ↓
POST /release khi người dùng trả link
~~~

## Lưu ý

- API hiện không yêu cầu token đăng nhập.
- Không lưu hoặc ghi log assignment.token.
- Không gọi POST /claim liên tục; hệ thống có giới hạn request.
- Nếu website khác domain bị CORS, hãy gọi API qua backend proxy.
- Không dùng mode: no-cors vì sẽ không đọc được JSON.
- Response có dạng:

~~~json
{
  "ok": true,
  "message": "...",
  "data": {}
}
~~~

Mã lỗi thường gặp: 422 dữ liệu sai, 429 vượt giới hạn, 502/503 dịch vụ tạm thời không khả dụng.

<!-- Dev by KMSDEV -->