---
Feature: Verify Phone Number via OTP (Telegram)
Squad: Wallet Squad
Complexity: Medium
AC Count: 8
---

# 1-PAGE SPEC: Verify Phone Number via OTP (Telegram)

**Feature:** Verify Phone Number via OTP  
**Tình trạng:** In Review  
**Người tạo:** AI Assistant  
**Ngày tạo:** 2026-02-04  
**Brief Reference:** [brief-verify-otp-telegram.md](file:///d:/Ng%E1%BB%8Dc/ai-thought-partner/vaults/vault-2-current-state/Wallet-Squad/brief-verify-otp-telegram.md)

---

### 1. Tổng quan (Outcome)

Tính năng yêu cầu người dùng mới khi đăng ký tài khoản phải xác thực **cả hai thông tin**:
1.  **Số điện thoại:** Thông qua Telegram (OTP/Deep Link).
2.  **Email:** Thông qua mã OTP gửi về email.

**Mục tiêu chính:** 
-   **Clean Data:** Đảm bảo SĐT và Email đều hoạt động và chính chủ.
-   **Security:** Tăng cường bảo mật 2 lớp thông tin ngay từ đầu.
-   **Marketing:** Phục vụ Telesale (SĐT) và Email Marketing (Email) chính xác.

## 2. Business Rules

### 2.1. Phương thức gửi OTP
-   **Kênh gửi:** Telegram.
-   **Cơ chế:** Hệ thống sử dụng **Telegram Gateway API** để gửi mã OTP đến tài khoản Telegram được liên kết với số điện thoại đăng ký.
-   **Ràng buộc:** OTP phải được gửi đến đúng số điện thoại mà người dùng đã nhập tại bước Đăng ký.

### 2.2. OTP Validation
-   **Thời gian hết hạn (Expiry):** 10 phút kể từ lúc phát sinh.
-   **Định dạng:** 6 chữ số (ví dụ: 123456).
-   **Giới hạn:** 
    -   Cho phép "Resend OTP" sau 60 giây.
    -   Tối đa 5 lần gửi/ngày/số điện thoại (để tránh spam).

### 2.3. Flow Đăng ký (Validate)
-   Người dùng **BẮT BUỘC** phải verify OTP thành công mới hoàn tất quá trình đăng ký (Account Created).
-   Nếu chưa verify: Tài khoản ở trạng thái `PENDING_VERIFICATION` hoặc không được tạo (tùy kiến trúc).

---

## 3. User Flow (Luồng người dùng)

### 3.1. Form Đăng ký
Người dùng truy cập màn hình đăng ký và nhập các thông tin sau:
1.  **Full Name:** Tên hiển thị người dùng.
2.  **Số điện thoại:** SĐT chính chủ (Dùng để nhận OTP Telegram).
3.  **OTP số điện thoại:** Input nhập mã 6 số (Trigger nút "Gửi mã" bên cạnh/bên dưới sau khi nhập SĐT).
4.  **Email:** Địa chỉ email.
5.  **How do you know about Wealify:** Dropdown/Selection (Theo tracking ref spec).
6.  **Password:** Mật khẩu đăng nhập.
7.  **Confirm Password:** Xác nhận mật khẩu.
8.  **Referral Code:** Mã giới thiệu (Optional).

### 3.2. Luồng chi tiết
1.  **Input Data:** User điền thông tin (Full name, SĐT...).
2.  **Verify Phone (Chọn 1 trong 2 flow tùy giải pháp):**
    *   **Flow A: Manual OTP (3rd Party)**
        -   User bấm nút **"Gửi mã"**.
        -   Hệ thống gửi Message OTP qua Telegram.
        -   User mở Telegram/Phone để lấy mã -> Nhập vào ô "OTP Phone".
        -   Hệ thống auto-validate khi nhập đủ 6 số (hoặc bấm verify).
    *   **Flow B: Deep Link (In-house)**
        -   User bấm nút **"Verify via Telegram"**.
        -   Hệ thống mở tab mới/redirect sang App Telegram (Deep Link).
        -   User bấm Start/Share Contact trên Telegram.
        -   Màn hình đăng ký Wealify tự động polling -> Hiển thị "Đã xác thực" (Green Check) ngay khi thành công.
3.  **Verify Email:**
    -   Sau khi Verify Phone thành công, hệ thống hiển thị trạng thái **"Đã xác thực"** (Green Check) bên cạnh số điện thoại và disable input SĐT.
    -   Màn hình đăng ký tự động focus vào ô nhập Email và gửi OTP (6 số) qua Email.
    -   User check mail, nhập mã vào ô **"OTP Email"**.
    -   Hệ thống validate mã.
4.  **Complete:** User bấm nút **"Đăng ký"**. 
    -   Hệ thống check valid các field còn lại (Pass khớp Confirm Pass...).
    -   Tạo tài khoản thành công (PhoneVerified = True, EmailVerified = True).

---

### 3.3. Handling Edge Cases & UX Solutions

| Scenario | Behavior with Option A (Manual OTP) | Behavior with Option B (Deep Link) |
| :--- | :--- | :--- |
| **User on Desktop** | **Action:** User phải tự mở điện thoại hoặc app Telegram Desktop để lấy mã. <br> **Friction:** Trung bình. User có thể lười check. | **Action:** User click link/nút "Verify". Hệ thống tự trigger mở app Telegram Desktop (hoặc Web) để start Bot. <br> **Friction:** Thấp. Seamless flow. |
| **No Telegram Account** | **Action:** UI báo lỗi khi gửi tin thất bại. User không biết phải làm gì nếu không đọc kỹ. | **Action:** Link chuyển hướng đến trang download/login của Telegram. <br> **Benefit:** Hướng dẫn user cài đặt ngay lập tức. |

*Note: Option B (Deep Link) giải quyết tốt hơn vấn đề trải nghiệm trên Desktop và hướng dẫn cài đặt cho user chưa có tài khoản.*

---

### 3.4. Luồng Chỉnh sửa hồ sơ (Edit Profile)

Khi người dùng đã có tài khoản và muốn thay đổi thông tin SĐT hoặc Email, hệ thống yêu cầu xác thực lại để đảm bảo bảo mật.

#### 3.4.1. Thay đổi Số điện thoại (SĐT)
1.  **Input:** User nhập SĐT mới tại màn hình Chỉnh sửa hồ sơ.
2.  **Verify:** User bấm nút **"Xác thực SĐT mới"**.
3.  **Process:** Hệ thống áp dụng quy trình xác thực qua Telegram (chọn **Option A** hoặc **Option B** tương tự lúc Đăng ký).
4.  **Constraint:** SĐT cũ vẫn được giữ nguyên cho đến khi SĐT mới được xác thực thành công.
5.  **Completion:** Sau khi xác thực thành công, hệ thống cập nhật SĐT mới vào DB và gửi thông báo (Email/Push) báo cáo việc thay đổi SĐT.

#### 3.4.2. Thay đổi Email
1.  **Input:** User nhập Email mới.
2.  **Verify:** User bấm nút **"Xác thực Email mới"**.
3.  **Process:** Hệ thống gửi mã OTP 6 số về **Email mới**.
4.  **Completion:** User nhập đúng mã OTP -> Hệ thống cập nhật Email mới và gửi thông báo xác nhận về **Email cũ** (để cảnh báo nếu không phải chính chủ thực hiện).

---

## 4. Technical Requirements (Telegram Integration)

### 4.1. Telegram Integration Strategy

Hệ thống hỗ trợ 2 phương án tích hợp để linh hoạt triển khai hoặc A/B Testing. Tùy thuộc vào khả năng của provider, team sẽ chọn một trong hai.

#### 4.1.1. Option A: Manual OTP (via Telegram Gateway API)
Sử dụng dịch vụ **Telegram Gateway** chính chủ để gửi mã OTP. Phí $0.01/1 tin nhắn thành công (link reference: https://core.telegram.org/gateway)
*   **Reference:** [Verification Tutorial](https://core.telegram.org/gateway/verification-tutorial)

*   **Workflow:**
    1.  **User Input:** Nhập SĐT -> Bấm "Gửi mã".
    2.  **Backend:**
        *   **Check Ability (Optional):** Gọi `checkSendAbility` để kiểm tra SĐT có hợp lệ và có thể nhận tin nhắn không.
            *   *Nếu OK:* Tiếp tục.
            *   *Nếu Fail:* Báo lỗi ngay cho User (SĐT chưa đăng ký Telegram).
        *   **Generate & Send:** Backend tạo OTP (e.g., `123456`) -> Gọi API `sendVerificationMessage`.
    3.  **Telegram Gateway:**
        *   Hệ thống Telegram gửi tin nhắn Service Notification tới User: *"Your verification code is 123456"*.
    4.  **User Action:** Nhập mã vào Form.
    5.  **Verify:** Backend verify code (so khớp với Redis).

*   **Technical Requirements (Telegram Gateway API):**
    *   **Endpoint:** `https://gateway.telegram.org/application/json`
    *   **Auth:** `Authorization: Bearer <API_TOKEN>`
    *   **Methods:**
        1.  `checkSendAbility`:
            *   Input: `{"phone_number": "..."}`
            *   Output: `request_id`, `status` (possible values: `possible`, `inaccurate_number`, etc.)
        2.  `sendVerificationMessage`:
            *   Input: 
                ```json
                {
                  "phone_number": "...",
                  "code": "123456",
                  "code_length": 6,
                  "ttl": 600,
                  "callback_url": "https://api.wealify.com/webhook/telegram-delivery"
                }
                ```
            *   Output: `request_id`, `delivery_status`.
    *   **Cost Management:** Chỉ bị tính phí khi message `delivery_status` là `sent`. Sử dụng `checkSendAbility` để tránh spam vào số rác.

#### 4.1.2. Option B: Deep Link (Recommended - In-House)
Sử dụng tính năng **Deep Linking** của Telegram Bot tự xây dựng (In-House Bot) để xác thực người dùng mà không cần nhập code thủ công.

*   **Workflow:**
    1.  **User Input:** User nhập SĐT `P` tại màn đăng ký -> Bấm "Verify".
    2.  **Backend:** 
        *   **Generate Token:** Tạo `verification_token` (UUID, 32 chars) unique gắn với `session_id` và SĐT `P`.
        *   **Store:** Lưu vào Cache (Redis) với TTL = 10 phút.
        *   **Construct Link:** Backend tự ghép link dựa trên biến môi trường `TELEGRAM_BOT_NAME`.
            *   Format: `https://t.me/{TELEGRAM_BOT_NAME}?start={verification_token}`
        *   **Response:** Trả về *Full Deep Link* cho Client.
    3.  **Frontend:**
        *   Mở Deep Link trong tab mới (hoặc redirect).
        *   Đồng thời hiển thị trạng thái "Waiting for verification..." (Polling status hoặc lắng nghe WebSocket).
    4.  **Telegram App:**
        *   User click "Start" (hoặc tự động nhận lệnh `/start {verification_token}`).
        *   **Bot:** Kiểm tra token. Nếu hợp lệ -> Gửi tin nhắn xin quyền chia sẻ SĐT:
            *   *Message:* "Chào bạn, vui lòng bấm nút dưới đây để xác thực số điện thoại."
            *   *Button:* `[Share My Contact]` (`request_contact=True`).
    5.  **User Verification:**
        *   User bấm `Share My Contact`.
        *   **Bot:** Nhận được object `contact`.
        *   **Logic:** So sánh `contact.phone_number` (Telegram trả về) với `P` (SĐT lưu trong Redis theo token).
            *   *Match:* Gọi API Backend `POST /internal/verify-phone` -> Đánh dấu session đã verify.
            *   *Mismatch:* 
                *   **Telegram Bot phản hồi ngay lập tức:** "❌ Số điện thoại Telegram của bạn ([Phone_Tele]) không trùng khớp với số đã đăng ký ([Phone_Register]). Vui lòng dùng đúng tài khoản Telegram."
                *   **Backend:** Đánh dấu verify thất bại (reason: mismatch).
    6.  **Completion:**
        *   Backend cập nhật trạng thái verify (Success/Fail).
        *   **Frontend (Wealify):** 
            *   *Success:* 
                *   Hiển thị trạng thái **"Đã xác thực"** (Green Check/Badge) bên cạnh số điện thoại.
                *   Disable input SĐT (không cho sửa).
                *   Tự động chuyển focus/flow sang bước Verify Email.
            *   *Fail (Mismatch):* Hiển thị thông báo lỗi ngay dưới input SĐT: "Xác thực thất bại. Số điện thoại Telegram không trùng khớp. Vui lòng thử lại."

*   **Technical Constraints:**
    *   **Token Security:** Token chỉ dùng 1 lần (One-time use) và hết hạn sau 10 phút.
    *   **Privacy:** Chỉ request lấy Contact khi token hợp lệ.
    *   **Bot Privacy Mode:** Cần tắt Group Privacy mode (nếu cần) hoặc chỉ hoạt động 1-1 interaction.

### 4.2. API Schema (Draft)
-   `POST /api/auth/register-init`: Gửi thông tin basic + SĐT -> Trả về `temp_token`.
-   `POST /api/auth/otp/send`: Input `phone`, `channel: TELEGRAM`.
-   `POST /api/auth/otp/verify`: Input `phone`, `otp_code` -> Trả về `access_token` (Login thành công).

---

## 5. Acceptance Criteria (Kiểm thử)

### AC1: Happy Path (Dual Verify)
-   **Given:** User nhập SĐT `0912345678` và Email `test@email.com`.
-   **When:** User verify Telegram thành công.
-   **Then:** Hệ thống cho phép tiếp tục/gửi OTP về Email `test@email.com`.
-   **When:** User nhập đúng OTP Email.
-   **Then:** Nút "Đăng ký" sáng (enabled). Bấm Đăng ký -> Thành công. DB ghi nhận `phone_verified=true`, `email_verified=true`.

### AC2: Sai OTP
-   **When:** User nhập `111111` (sai mã).
-   **Then:** Hệ thống báo lỗi "Mã xác thực không đúng". User được nhập lại.

### AC3: Hết hạn OTP
-   **Given:** Đã quá 10 phút kể từ khi mã được gửi.
-   **When:** User nhập đúng mã cũ.
-   **Then:** Hệ thống báo lỗi "Mã OTP đã hết hạn". Nút "Gửi lại" được active.

### AC4: Resend OTP Cooldown
-   **Given:** User vừa bấm "Gửi mã".
-   **When:** User bấm lại "Gửi mã" trong vòng 60 giây.
-   **Then:** Nút "Gửi mã" bị disable và hiển thị countdown (e.g., "Gửi lại sau 45s").

### AC5: Daily Limit
-   **Given:** User đã gửi OTP 5 lần trong ngày.
-   **When:** User bấm "Gửi mã" lần thứ 6.
-   **Then:** Hệ thống báo lỗi "Bạn đã vượt quá giới hạn gửi mã trong ngày. Vui lòng thử lại sau 24h."

### AC6: Deep Link Expiry
-   **Given:** User mở Deep Link Telegram sau 10 phút kể từ khi trigger trên Web.
-   **When:** User bấm "Start" trong Bot.
-   **Then:** Bot phản hồi: "Phiên làm việc đã hết hạn. Vui lòng quay lại ứng dụng để lấy link mới."

### AC7: Phone Normalization (Deep Link)
-   **Given:** User nhập SĐT `0912345678` trên Web.
-   **When:** User share contact từ Telegram (trả về `+84912345678`).
-   **Then:** Hệ thống normalize và so khớp thành công -> Đánh dấu `Verified`.

### AC8: Email OTP Conflict
-   **Given:** User chưa verify Phone thành công.
-   **When:** User cố tình trigger API gửi Email OTP.
-   **Then:** Hệ thống chặn và yêu cầu hoàn thành xác thực SĐT trước.

### AC9: Edit Profile - Change Phone Success
-   **Given:** User đang ở màn hình Edit Profile.
-   **When:** User nhập SĐT mới và verify Telegram thành công.
-   **Then:** SĐT được cập nhật. Hệ thống gửi thông báo bảo mật về Email.

### AC10: Edit Profile - Change Email Security
-   **Given:** User thay đổi Email thành công.
-   **Then:** Hệ thống gửi 1 thông báo "Bạn đã đổi email thành công" đến Email cũ để đảm bảo an toàn tài khoản.

---

## 6. Telemetry & Monitoring (Success Metrics)

### 6.1. Analytics Events
- `auth_phone_verify_start`: Khi user bấm "Gửi mã/Verify".
- `auth_phone_verify_success`: Khi verify thành công (Option A/B).
- `auth_phone_verify_fail`: Khi sai OTP hoặc mismatch SĐT (kèm `reason`).
- `auth_email_verify_success`: Khi hoàn tất cả 2 bước.

### 6.2. Key Metrics
- **Success Rate (Phone):** Mục tiêu > 90%.
- **Deep Link Conversion:** Tỷ lệ user click link -> Share contact thành công.
- **Drop-off Rate:** User dừng lại ở bước verify Phone.

---

## 7. OKR Alignment
- **Objective:** Tăng trưởng người dùng thực.
- **Key Result:** Giảm 50% tài khoản ảo/spam thông qua hệ thống xác thực 2 lớp (Phone + Email).
