# Guideline: Quản lý Task & Bug trên Linear (Docs-First Approach)

Đây là hướng dẫn hoàn chỉnh để team quản lý sprint hiệu quả trên Linear, đảm bảo tính kỷ luật và chi tiết như quy trình Jira nhưng vẫn tuân thủ triết lý "Docs First" của Wealify.

## 0. Nguyên tắc cốt lõi (Docs-First)
Trong hệ thống Wealify:
- **Spec là Chân Lý (Single Source of Truth)**: Mọi requirements, logic nghiệp vụ, edge cases phải nằm trong file Markdown (tại `vault-2` hoặc thư mục Docs).
- **Linear là Công Cụ Thực Thi**: Linear chỉ dùng để track trạng thái "Ai làm gì, bao giờ xong".
- **Rule Bất Di Bất Dịch**: Không viết requirements chi tiết trực tiếp vào mô tả Linear. Phải luôn có link trỏ về Spec.

---

## 1. Cấu trúc Hierarchy (Tổ chức)
Ánh xạ từ mô hình Agile/Jira sang Linear:

| Khái niệm Jira/Agile | Đối tượng Linear | Giải thích |
| :--- | :--- | :--- |
| **Epic / Feature Request** | **Project** | Một tính năng lớn (Vd: Wealify Snapshot 2025). Project chứa nhiều tasks. |
| **Sprint** | **Cycle** | Chu kỳ làm việc (thường 1-2 tuần). |
| **User Story / Task** | **Issue** | Một đầu việc cụ thể để hoàn thành tính năng. |
| **Sub-task** | **Sub-issue / Checklist** | Các bước nhỏ để hoàn thành task lớn (Vd: Cắt html, css, ghép API). |
| **Bug** | **Issue (Label: Bug)** | Lỗi phần mềm cần sửa. |

---

## 2. Quy chuẩn Log Task (Feature Development)
Sử dụng khi tạo task phát triển tính năng mới.

### 2.1 Tiêu đề (Title)
Cấu trúc: `[Team] <Tên ngắn gọn của task>`
*Ví dụ:*
- `[BE] API Get User Snapshot Data`
- `[FE] Implement Wrap-up UI Slide 1`
- `[Mobile] Integrate Push Notification Service`

### 2.2 Nội dung (Description) - BẮT BUỘC
Sử dụng template sau:

```markdown
**Context (Nguồn sự thật)**:
- Spec: [Paste Link tới file Markdown Spec tại đây]
- Design: [Paste Link tới màn hình Figma cụ thể]

**Acceptance Criteria (Tiêu chí nghiệm thu)**:
> Copy từ Spec hoặc Action Plan sang.
- [ ] AC1: Button hiển thị đúng state disabled khi chưa nhập tiền.
- [ ] AC2: API trả về đúng format JSON như document.
- [ ] AC3: Xử lý đúng case mất mạng (retry mechanism).

**Technical Notes (Ghi chú kỹ thuật)**:
- Sử dụng endpoint `/v1/snapshot`.
- Lưu ý cache lại response này 24h.
```

### 2.3 Estimation (Ước lượng)
Sử dụng T-shirt sizing hoặc Point để đánh giá độ phức tạp:
- **1 Point (Small)**: < 2 giờ.
- **2 Points (Medium)**: 2 - 4 giờ (nửa ngày).
- **3 Points (Large)**: 4 - 8 giờ (1 ngày).
- **5+ Points (X-Large)**: > 1 ngày -> **CẦN BREAK NHỎ HƠN**.

---

## 3. Quy chuẩn Log Bug
Quy trình báo lỗi chuẩn để Dev không phải hỏi lại thông tin (Zero Back-and-forth).

### 3.1 Tiêu đề (Title)
Cấu trúc: `[Bug] <Mô tả lỗi ngắn gọn và rõ ràng>`
*Ví dụ:*
- `[Bug] App crash khi bấm nút Back ở màn Payment`
- `[Bug] Sai số dư hiển thị sau khi topup thành công`

### 3.2 Priority (Độ ưu tiên) - Quan trọng
Sử dụng field Priority của Linear:
- **Urgent (Blocker)**: Không thể test tiếp luồng chính, crash app, mất tiền, lộ data. -> **Fix Ngay Lập Tức**.
- **High**: Tính năng chính bị sai, ảnh hưởng lớn đến user. -> **Fix trong Sprint này**.
- **Medium**: Lỗi chức năng phụ, UI/UX chưa mượt, edge case hiếm gặp. -> **Fix nếu còn thời gian**.
- **Low**: Lỗi typo, màu sắc lệch nhẹ, cosmetic. -> **Backlog**.

### 3.3 Nội dung (Description) - BẮT BUỘC
Template chuẩn cho Bug:

```markdown
**Environment (Môi trường test)**:
- OS: iOS 17.0 / Android 14 / Web Chrome v120
- App Version: v2.5.0 (Staging build 102)

**Steps to Reproduce (Các bước tái hiện)**:
1. Đăng nhập user B (sđt: 0909xxx).
2. Vào màn hình Wallet -> Chọn Withdraw.
3. Nhập số tiền 50k -> Bấm Confirm.

**Expected Result (Kết quả mong muốn)**:
- Hiển thị popup xác nhận OTP.

**Actual Result (Kết quả thực tế)**:
- Màn hình load mãi mãi (loading spinner) không phản hồi.

**Evidence (Bằng chứng)**:
- [Đính kèm Ảnh chụp màn hình]
- [Đính kèm Link Video quay lỗi]
- [Paste Log lỗi/Crash log nếu có]
```

### 3.4 Liên kết & Truy vết (Traceability)
Để biết Bug thuộc tính năng nào, sử dụng tính năng **Relations** của Linear:

- **Related to**: Dùng để link Bug với Task gốc (Feature Task).
  - *Vị trí UI*: Nhìn cột bên phải (Sidebar), tìm mục **Relations** -> Bấm dấu `+`.
  - *Shortcut*: Bấm tổ hợp `Ctrl + Shift + R` (hoặc `Cmd + Shift + R`) hoặc `Ctrl + K` gõ "Relate".
  - *Mục đích*: Khi mở Task Feature, sẽ biết ngay có bao nhiêu bug liên quan.
- **Parent Issue**: Ít dùng cho Bug, trừ khi Bug đó quá lớn cần chia nhỏ thành sub-tasks.
- **Project**: Luôn add Bug vào **Project** tương ứng (Vd: `Wealify 2025`). Điều này giúp xem được "Health" của Project (tỷ lệ bug/feature).

> **Quy tắc**: Nếu Bug phát sinh **ngay trong quá trình dev** tính năng -> Comment hoặc tạo Sub-task trong Task chính. Nếu Bug phát sinh **sau khi merge** -> Tạo Issue Bug riêng và Link `Related to` về Task chính.

---

## 4. Sprint (Cycle) Workflow
Vòng đời của vấn đề trong 1 Sprint:

1.  **Triage / Backlog**:
    - Task/Bug mới tạo, chưa được gán vào Cycle nào.
    - PM/Lead review, điền đủ thông tin, Spec, assign người làm rồi mới đẩy vào Cycle.

2.  **Todo (Trong Cycle)**:
    - Task đã chốt sẽ làm trong tuần này.
    - Dev **TỰ GIÁC** kéo task từ trên xuống dưới theo độ ưu tiên.

3.  **In Progress**:
    - Dev bắt đầu code.
    - **Nguyên tắc**: Tại 1 thời điểm chỉ để 1 task ở trạng thái In Progress.
    *Tạo nhánh git từ nút "Copy branch name" trong Linear.*

4.  **In Review**:
    - Dev đã code xong, tạo Pull Request (PR).
    - Link PR được gắn vào Issue Linear.
    - Tag người review.

5.  **Done / Canceled**:
    - **Done**: Code đã được merge và deploy lên môi trường Staging/Prod. Đã nghiệm thu (QA Verified).
    - **Canceled**: Task bị hủy bỏ, không làm nữa (cần ghi lý do vào comment).

## 5. Definition of Done (DoD)
Dev chỉ được phép kéo task sang cột **Done** khi:
- [ ] Code đã merge vào branch chính (`develop`/`main`).
- [ ] Unit Test đã pass (nếu có).
- [ ] Đã tự verify trên môi trường Staging.
- [ ] Các checklist AC trong ticket đã được tích chọn hết.
- [ ] Đã cập nhật lại tài liệu Spec/Docs nếu trong quá trình làm có thay đổi logic khác với ban đầu.
