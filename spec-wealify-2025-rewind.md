# 1-Page Spec: Wealify Snapshot 2025 (Rewind)

## A. Outcome
Tính năng "Wealify Snapshot 2025" giúp người dùng nhìn lại hành trình tài chính trong năm qua, tạo cảm xúc tích cực và gắn kết với thương hiệu. Tạo hiệu ứng lan tỏa (viral) thông qua việc khuyến khích user chia sẻ "Result Card" cá nhân hóa lên mạng xã hội (Marketing Branding).

## B. In-scope / Out-of-scope

### ✅ In-scope
- **Back-end Job**: Pre-calculate dữ liệu và phân loại user (Batch processing) theo 5 tiêu chí ưu tiên.
- **UI/UX**: Flow hiển thị 3 Slides (Intro, Activity, Result Card) với motion/animation.
- **Entry points**: Home Banner, Pop-up (lần đầu mở), Push Notification deep link.
- **Sharing**: Chức năng chụp/xuất ảnh Result Card (tỷ lệ 9:16) để share Social.

### ❌ Out-of-scope
- Tính toán dữ liệu Real-time (vi phạm BR performance).
- Người dùng tự tùy chỉnh background/theme của slide.
- Xem lại lịch sử Rewind các năm trước (chỉ support 2025).

## C. Flow

### Main Flow
1. **System (Batch Job)**: Chạy job offline, tính toán số liệu (Volume, Tx Count, Days...) và gán Danh hiệu cho từng User -> Lưu DB.
2. **User**: Mở app Wealify.
3. **App**: Check config event & trạng thái `has_seen_rewind` của user.
4. **App**: Hiển thị **Pop-up** "2025 REWIND" (với user chưa xem).
5. **User**: User tap vào Pop-up (hoặc Banner ở Home Screen).
6. **App**: Trình chiếu Slide:
   - **Slide 1 (Intro)**: Chào mừng, thời gian đồng hành.
   - **Slide 2 (Activity)**: Tổng quan dòng tiền, tháng đỉnh cao.
   - **Slide 3 (Result)**: Huy hiệu, Danh hiệu, Quote tương ứng.
7. **User**: Tại Slide 3, bấm button **[Chia sẻ ngay]**.
8. **App**: Generate ảnh 9:16 -> Gọi Native Share Sheet (để post Story FB/Insta).

### Error Flows
1. **Data not ready**: Job lỗi hoặc chưa chạy xong cho user này -> Không hiện Pop-up/Banner.
2. **Share fail**: User từ chối quyền truy cập thư viện ảnh -> Hiển thị dialog hướng dẫn cấp quyền / thử lại.
3. **Assets load fail**: Ảnh/Icon không tải được -> Hiển thị placeholder/loading, hạn chế crash flow.

## D. Business Rules / Logic
1. **Data Fields (BR01)**:
   Với mỗi User ID, hệ thống cần trả về:
   - `user_name`: Tên hiển thị của khách hàng.
   - `join_date`: Ngày tạo tài khoản.
   - `days_with_wealify`: Tính từ join_date đến hiện tại.
   - `total_tx_count`: Tổng số lượng giao dịch trong năm 2025.
   - `peak_month`: Tháng có số lượng giao dịch cao nhất (Ví dụ: "Tháng 11").
   - `total_volume`: Tổng giá trị giao dịch (trước phí) trong năm 2025.
   - `avg_monthly_volume`: total_volume / 12 (hoặc chia cho số tháng hoạt động nếu user mới).
2.  **Pre-calculation Compliance (BR02)**: Tuyệt đối không tính toán khi user bấm xem. Mọi data `{total_volume}`, `{peak_month}`, `{rank_title}`... phải được chuẩn bị sẵn trong bảng `Rewind_2025_Report`.

3. **User Classification Logic (BR03)**:
   Hệ thống xác định danh hiệu theo cơ chế **Waterfall (Ưu tiên từ P1 -> P5)**. User đã trúng Priority cao sẽ bị loại khỏi danh sách xét duyệt của Priority thấp hơn.

   | Priority | Danh hiệu | Logic xác định (pseudo-code) |
   | :--- | :--- | :--- |
   | **P1** | **The Wealth Architect** | `User` $\in$ `Top_5_Percent_Volume` |
   | **P2** | **Wealify Whale** | `User` $\in$ `Top_20_Percent_Volume` AND `User` $\notin$ **P1** |
   | **P3** | **Chiến Thần Wealify** | `User` $\in$ `Top_20_Percent_TxCount` AND `User` $\notin$ {**P1**, **P2**} |
   | **P4** | **Người Đồng Hành Bền Bỉ** | (`days` > 270 AND `active_months` $\ge$ 10) AND `User` $\notin$ {**P1**, **P2**, **P3**} |
   | **P5** | **The Explorer** | Các User còn lại (`User` $\notin$ {**P1**, **P2**, **P3**, **P4**}) |

   **Định nghĩa tập hợp (Set Definitions):**
   - `Top_5_Percent_Volume`: Danh sách 5% user có `total_volume` cao nhất.
   - `Top_20_Percent_Volume`: Danh sách user có `total_volume` thuộc Top 20%**(không bao gồm Top 5%).
   - `Top_20_Percent_TxCount`: Danh sách 20% user có `total_tx_count` cao nhất
   - `Total_Users`: Tổng user active + inactive trong hệ thống.

4. **Entry Points Rule (BR05)**:
   - **Pop-up**: Chỉ trigger **1 lần duy nhất** khi mở app trong thời gian sự kiện (nếu chưa xem).
   - **Banner**: Hiển thị xuyên suốt tại Home trong thời gian sự kiện.

5. **Dynamic Content (BR04)**:
   - Text chứa placeholders (`{user_name}`, `{join_date}`, `{total_volume}`...) phải được replace bằng data thật.
   - Visual/Badge/Quote ở Slide 3 thay đổi theo Danh hiệu user nhận được.



## E. Acceptance Criteria

### AC1: Phân loại đúng Priority
```gherkin
Given User A có volume top 4% và tx_count top 30%
When Job phân loại chạy
Then User A được gán "The Wealth Architect" (Do match P1)
And Không xét tiếp các điều kiện P2, P3...
```

### AC2: The Explorer Fallback
```gherkin
Given User B là người dùng mới, volume thấp
When Job phân loại chạy
Then User B không thõa mãn P1, P2, P3, P4
And User B được gán mặc định là "The Explorer"
```

### AC3: Luồng Pop-up
```gherkin
Given User C chưa xem Rewind 2025
When User C mở app lần đầu trong ngày diễn ra
Then App hiển thị Pop-up mời xem
When User C đóng Pop-up và kill app mở lại
Then App KHÔNG hiển thị lại Pop-up (chỉ hiển thị Banner)
```

### AC4: Share Content
```gherkin
Given User đang ở Slide 3 (Result) với danh hiệu "Wealify Whale"
When User bấm Chia sẻ
Then Ảnh được tạo ra phải chứa:
  - Tên User
  - Title "WEALIFY WHALE"
  - Quote tương ứng với Whale
  - Logo Wealify & Hashtag #WealifySnapshot2025
```

## F. Edge Cases
1. **User 0 Giao dịch**: Vẫn cho trải nghiệm Rewind (The Explorer). Slide 2 (Activity) hiển thị text khéo léo thay vì số 0 trơ trọi (VD: "Bạn đang tích lũy năng lượng cho 2026...").
2. **User mới join < 1 ngày**: Có thể chưa có trong batch job -> Không hiển thị tính năng.
3. **Màn hình nhỏ/gập**: Đảm bảo text tại Slide 3 không bị cắt khi generate ảnh 9:16 trên các thiết bị có tỷ lệ màn hình dị.

## G. Telemetry
- **Events**: 
  - `rewind_popup_view`: Số lần Pop-up xuất hiện.
  - `rewind_start`: Số lần user bắt đầu xem (từ Banner + Popup).
  - `rewind_complete`: Số user xem hết Slide 3.
  - `rewind_share_click`: User bấm nút Share.
  - `rewind_share_success`: User share thành công (callback từ OS nếu có).
- **Metrics**: Completion Rate (Start/Complete), Viral Coefficient (Share/View).
