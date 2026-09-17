---
name: ldg-ket-noi-data-crm
description: Connect HTML forms to CRM external tracking.
version: 0.1.0
author: Minh Duc (minhduc6560-gif), Hermes Agent
license: MIT
platforms:
- linux
- macos
- windows
metadata:
  hermes:
    tags:
    - agency
    - crm
    - gohighlevel
    - external-tracking
    - forms
    related_skills:
    - ghl-workflow-automation
    - durable-crm-lead-pipelines
---

# Kết nối data CRM bằng HighLevel External Tracking

Dùng skill này khi cần giữ nguyên giao diện custom HTML form nhưng gửi submission, attribution và Contact vào GoHighLevel/HighLevel bằng External Tracking Script. Quy trình ưu tiên không tạo External Form rác, không đổi identifier sau khi production và không đưa Private Integration Token vào frontend.

## When to Use

- Landing page HTML/CSS/JS cần gửi lead hoặc đơn hàng vào HighLevel.
- Cần dùng `External Form Submission` và workflow trigger của External Tracking.
- Muốn thử mapping Name, Phone, Email trước khi tạo custom fields.
- Cần chẩn đoán submission có nhưng Contact upsert thất bại.

Không dùng khi cần retry chắc chắn, chống mất lead, multi-step phức tạp, thanh toán hoặc nhiều đích đến. Các trường hợp đó nên dùng server-side bridge và Contact Upsert API.

## Quy tắc chọn trình duyệt - bắt buộc

- Mọi thao tác cần đăng nhập hoặc điều khiển giao diện tại `{{CRM_APP_BASE_URL}}` phải mở bằng **an authenticated browser session**, dùng đúng Chrome profile đã đăng nhập CRM application.
- **Không mở CRM application trong tab Preview bên phải của Hermes.** Preview là WebView/Electron có cookie và phiên đăng nhập riêng, dễ bị chặn đăng nhập và không kế thừa phiên Chrome.
- Tab Preview bên phải chỉ dùng để xem landing page local/public, HTML artifact hoặc QA giao diện không yêu cầu phiên đăng nhập CRM application.
- Nếu tác vụ có thể hoàn thành bằng API/MCP, ưu tiên API/MCP. Chỉ chuyển sang giao diện CRM application khi API/MCP không hỗ trợ hoặc người dùng yêu cầu.
- Trước khi thao tác UI, xác nhận cửa sổ đích là ứng dụng **Google Chrome**, không phải pane Preview của Hermes.

## Điều kiện đầu vào

Cần có:

- HighLevel Sub-account/Location đích.
- Script tại `Settings → External Tracking → Copy Script`.
- URL landing page hoặc local preview.
- HTML form nằm trực tiếp trong DOM, không nằm trong iframe.
- Tất cả field cần gửi có thuộc tính `name`.

Script thường có dạng:

```html
<script
  src="https://link.msgsndr.com/js/external-tracking.js"
  data-tracking-id="tk_xxxxxxxxxxxxxxxxx">
</script>
```

Đặt trước `</body>`. Không đưa annotation như `@url:` vào `src`.

## Quy tắc bất biến cho External Form identifier

External Form của HighLevel không có luồng xóa đáng tin cậy trong UI. Mỗi `formId/source-id` khác nhau có thể tạo thêm một External Form vĩnh viễn. Vì vậy coi identifier như tài nguyên production bất biến.

Trước lần POST thật đầu tiên:

1. Chọn một identifier ASCII cố định, ví dụ `lp-example-order-v1`.
2. Chỉ dùng `a-z`, `0-9`, dấu `-`, `_` hoặc `.`.
3. Không dùng dấu tiếng Việt, emoji, xuống dòng hoặc khoảng trắng.
4. Không đổi identifier khi sửa CSS, label, validation hoặc field mapping.
5. Chỉ tạo `v2` khi business yêu cầu một form/analytics flow mới và đã được duyệt.

HighLevel tracking script hiện nhận identifier theo thứ tự ưu tiên:

```text
id → name → data-formid → data-name → heading/title fallback
```

Nếu có nhiều thuộc tính, chúng phải cho cùng một identifier. Tốt nhất chỉ có một nguồn identifier kỹ thuật. Tên hiển thị tiếng Việt phải tách riêng bằng heading, `aria-label` hoặc nội dung trang.

### Lỗi `source-id`

Không dùng tên có dấu làm identifier. Backend HighLevel có thể đưa `formId` vào HTTP header `source-id` và trả lỗi:

```text
Error in upserting contact - Invalid character in header content ["source-id"]
```

Khi gặp lỗi này, submission có thể xuất hiện nhưng Contact không được upsert. Đổi identifier thành ASCII an toàn rồi tạo submission mới; submission cũ không tự sửa.

## Mapping field mặc định

Bắt đầu với field chuẩn trước khi tạo custom fields:

| Field khách nhập | `name` đề xuất | Label máy đọc |
|---|---|---|
| Họ và tên | `first_name` | `First Name` |
| Số điện thoại | `phone` | `Phone` |
| Email | `email` | `Email` |

- Email nên bắt buộc trong lần test đầu để Contact matching ổn định.
- Số điện thoại nên test ở dạng quốc tế, ví dụ `+849****4567`.
- Nếu giao diện cần tiếng Việt nhưng mapper cần label tiếng Anh, tách visual label khỏi machine-readable label và xác minh payload thực tế. Không giả định chỉ đổi CSS là đủ.
- Province, address, quantity và note có thể xuất hiện dưới dạng unmapped fields trước khi tạo custom fields.

## Quy trình chuẩn

### 1. Đọc form và khóa identifier

- Dùng `read_file` hoặc `search_files` để xác định thẻ `<form>`, field names, submit handler và vị trí `</body>`.
- Ghi lại tracking ID, Location, technical form ID, display name, domain và page path trong registry của project.
- Xác nhận technical form ID là ASCII/header-safe trước khi tiếp tục.

Hoàn tất khi identifier đã được duyệt và không còn thuộc tính nhận diện cạnh tranh.

### 2. Chuẩn hóa cấu trúc form

Form phải có:

- Một thẻ `<form>` thật.
- Input/select/textarea hiển thị trong DOM.
- `name` cho mọi field.
- `type="email" required` nếu Email bắt buộc.
- Nút `type="submit"`.

External Tracking hỗ trợ AJAX/custom submit, nhưng handler không được gọi `stopPropagation()` hoặc chặn listener capture của tracking script. Nếu dùng `preventDefault()`, phải xác minh request tracking vẫn phát sinh trong browser thật.

Hoàn tất khi native validation và submit event đều hoạt động.

### 3. Gắn script idempotent

- Chèn script đúng một lần trước `</body>`.
- Nếu có generator, generator phải xóa phiên bản cũ trước khi chèn lại.
- Build hai lần liên tiếp và đếm tracking ID; kết quả phải bằng `1`.

Hoàn tất khi không có duplicate script.

### 4. Static QA

Kiểm tra trước khi cho request ra internet:

- Tracking ID đúng Location.
- Identifier khớp registry.
- Identifier ASCII/header-safe.
- Email required nếu được yêu cầu.
- Field names và labels đúng.
- Không có duplicate IDs hoặc missing assets.
- Giao diện desktop/mobile không regression.

Hoàn tất khi tất cả check đều PASS.

### 5. Intercepted Network QA

Mở trang trong Chrome/CDP hoặc browser devtools, cho tracking script tải thật nhưng chặn POST tới backend trước khi gửi. Xác minh request dự kiến:

```text
POST https://backend.leadconnectorhq.com/external-tracking/events
```

Payload phải có:

```json
{
  "type": "external_form_submission",
  "formId": "lp-project-form-v1",
  "formData": {
    "first_name": "...",
    "phone": "+84...",
    "email": "..."
  },
  "formLabels": {
    "first_name": "First Name",
    "phone": "Phone",
    "email": "Email"
  }
}
```

Không chạy POST thật chỉ để xem payload. Nếu request bị cố ý chặn, tracking script có thể retry; số request retry không đồng nghĩa có nhiều submission production.

Hoàn tất khi payload đúng và chưa tạo External Form/Contact rác.

### 6. Một submission production

Chỉ chạy sau khi static QA và intercepted QA đều PASS:

1. Đóng toàn bộ tab/cache chứa phiên bản form cũ.
2. Deploy hoặc hard refresh bản đã khóa identifier.
3. Dùng Email test mới và Phone chuẩn `+84`.
4. Submit đúng một lần.
5. Chờ 10-60 giây.
6. Kiểm tra `Sites → Forms → Submissions → External Forms`.
7. Kiểm tra Contact bằng Email/Phone.
8. Kiểm tra Contact activity và field mapping.

Hoàn tất khi đúng một submission và một Contact được tạo/cập nhật.

### 7. Workflow production

Chỉ bật workflow sau khi Contact upsert PASS. Filter tối thiểu theo:

- Canonical external form ID.
- Domain production.
- Page path.
- UTM nếu cần.

Không dùng trigger chung `All External Forms` nếu Location đã có form test/rác.

## Registry đề xuất

Mỗi form cần một bản ghi:

```text
Project
GHL Location
Tracking ID
Technical form ID
Display name
Domain
Page path
Version
Status: sandbox | active | obsolete
Owner
Date locked
```

Mọi thay đổi identifier phải có phê duyệt. Field mapping và UI có thể đổi mà không đổi form ID.

## Sandbox và production

- Dùng một HighLevel Location sandbox cho các thử nghiệm end-to-end lặp lại.
- Production chỉ nhận một submission QA sau khi identifier đã khóa.
- Không test đồng thời từ localhost, staging và production nếu các bản đang dùng identifier khác nhau.
- Không submit từ tab cũ sau khi đổi bản deploy.

## Pitfalls

1. **Đổi tên form để sửa typo:** tạo External Form mới. Sửa display name, không đổi technical ID.
2. **Identifier có dấu:** có thể gây lỗi header `source-id` và Contact upsert fail.
3. **Form có `id` và `name` khác nhau:** script ưu tiên `id`, khiến dashboard không dùng tên mong đợi.
4. **Label tiếng Việt không map field mặc định:** dùng key/label chuẩn và xác minh payload.
5. **Submission xuất hiện nhưng Contact không có:** đọc cột lỗi và Contact activity; kiểm tra `source-id`, field key, Phone format và Location.
6. **Thông báo thành công trên UI:** không phải bằng chứng GHL đã nhận. Bằng chứng là External Form Submission và Contact record.
7. **Ad blocker:** có thể chặn `link.msgsndr.com` hoặc `backend.leadconnectorhq.com`.
8. **Generator chạy nhiều lần:** có thể nhân tracking script nếu không idempotent.
9. **Email/Phone cũ:** HighLevel có thể update Contact cũ thay vì tạo mới.
10. **Form cũ không xóa được:** đánh dấu obsolete và filter workflow/report theo canonical ID.

## Verification checklist

- [ ] Tracking script xuất hiện đúng một lần.
- [ ] Tracking ID thuộc đúng Location.
- [ ] Canonical form ID khớp registry.
- [ ] Form ID chỉ chứa ASCII/header-safe characters.
- [ ] Không thay form ID sau submission production đầu tiên.
- [ ] `first_name`, `phone`, `email` có key/label chuẩn.
- [ ] Email validation hoạt động.
- [ ] Intercepted payload có `external_form_submission`.
- [ ] Intercepted POST chưa tạo dữ liệu rác.
- [ ] Submission production xuất hiện trong External Forms.
- [ ] Contact được upsert đúng một lần.
- [ ] Workflow chỉ lọc canonical form/domain/path.
- [ ] Các form cũ được đánh dấu obsolete, không dùng trong automation.

## Nguồn chính thức

- HighLevel Support - Tracking External Forms with GoHighLevel: https://help.gohighlevel.com/support/solutions/articles/155000006092-tracking-external-forms-with-gohighlevel
