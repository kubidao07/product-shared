# 1-PAGE SPEC: VA Withdraw to USDT

**Feature:** VA Withdraw to USDT  
**Squad:** VA Squad  
**Sprint:** TBD  
**BA/PO:** [To be assigned]  
**Created:** 2026-01-15  
**Reference Brief:** `Kubi gen/brief-va-withdraw-to-usdt.md`  
**Complexity:** Medium  
**Target Word Count:** 1,200-1,500 words  
**Target ACs:** 10-12

---

## A. Outcome

Cho phép user của domain **hispay.net** rút tiền VND từ Virtual Account trực tiếp thành USDT (qua Cobo wallet) với tỷ giá real-time từ Binance, giảm 30% thời gian so với flow thủ công (VND → bank → mua USDT), đồng thời đảm bảo tính minh bạch về tỷ giá và kiểm soát rủi ro qua Maker-Checker approval workflow trên CMS.

---

## B. In-scope / Out-of-scope

### ✅ In-scope (Làm trong sprint này)

1. **Domain-based Feature Flag:** Chỉ hispay.net users thấy "Withdraw to USDT" button, Wealify users KHÔNG thấy
2. **Real-time Exchange Rate Display:** Hiển thị tỷ giá VND/USDT (lấy trung bình từ top 20 tỷ giá/ads trên Binance API, scan mỗi 60s) trên withdraw form
3. **Auto-calculate USDT:** User nhập VND amount → Hệ thống tự động tính USDT nhận được (sau trừ fee - fee được Admin input thủ công)
4. **Balance Deduction:** Trừ VND balance chính xác (amount + fee) khi submit withdraw request
5. **Withdraw Request Creation:** Tạo request với status PENDING, lưu vào DB
6. **CMS Approval Workflow:** Admin/BOD thấy pending requests trên CMS, có thể Approve/Reject
7. **Cobo USDT Transfer:** Sau khi approve, gửi USDT qua Cobo WaaS 2.0 (Tron TRC-20) đến user wallet
8. **User Cancel Pending:** User có thể cancel request nếu status = PENDING (chưa approve), refund VND ngay
9. **Notification:** Gửi email/push notification khi request approved/rejected/completed/failed

### ❌ Out-of-scope (Không làm trong sprint này)

1. **Multi-currency withdraw:** Chỉ support USDT, không support USDC/BTC/ETH (defer Phase 2)
2. **Auto-approval based on amount:** Tất cả requests đều cần manual approval, không có auto-approve cho small amounts
3. **Batch withdraw:** User chỉ withdraw từng lần, không support batch multiple requests
4. **Binance API key management UI:** API key hard-coded trong env, không có UI để config
5. **Historical rate chart:** Chỉ show current rate, không show chart lịch sử tỷ giá

---

## C. Flow chính + Flow lỗi

### 🔹 Main Flow 1: User Submit Withdraw Request

1. User (hispay.net domain) login và navigate to VA Withdraw page
2. Frontend check domain → Nếu hispay.net → Show "Withdraw to USDT" tab
3. User click "Withdraw to USDT" tab
4. Form hiển thị:
   - Current VND balance: 10,000,000 VND
   - Exchange rate (auto-update mỗi 60s): 1 USDT = 25,500 VND (trung bình lấy từ top 20 cặp/tỷ giá trên Binance)
   - Input field: "Số VND muốn rút"
   - Fee display: Hiển thị fee (Admin input)
   - USDT receive display: Auto-calculate khi user nhập VND và có fee
5. User nhập: 1,000,000 VND
6. System tính:
   - Fee: Dựa trên cấu hình cho user (Admin input)
   - Net amount: 1,000,000 - fee
   - USDT receive: Net amount / 25,500 = USDT tương ứng
7. User xem preview: "Bạn sẽ nhận [X] USDT (rate: 25,500 VND/USDT, fee: [Y] VND)"
8. User nhập Cobo wallet address (Tron TRC-20)
9. User click "Submit Withdraw"
10. Frontend validate:
    - Amount ≥ 500,000 VND (minimum)
    - Balance ≥ amount + fee
    - Wallet address valid (Tron format: starts with T, 34 chars)
11. Backend atomic transaction:
    - Deduct balance: 10,000,000 - 1,000,000 = 9,000,000 VND
    - Insert withdraw_requests table:
      - user_id, amount_vnd: 1,000,000, fee_vnd: 10,000, usdt_amount: 38.82
      - exchange_rate: 25,500, wallet_address, status: PENDING
      - created_at: timestamp
12. Return success: "Withdraw request submitted. Waiting for approval."
13. User redirect to "My Withdraw Requests" page, thấy request status = PENDING

### 🔹 Main Flow 2: Admin Approve Request (CMS)

1. Admin login to CMS
2. Navigate to "VA Withdraw Requests" page
3. CMS query DB: `SELECT * FROM withdraw_requests WHERE status = 'PENDING' ORDER BY created_at DESC`
4. Admin thấy list pending requests với columns:
   - Request ID, User, Amount VND, Fee, USDT Amount, Rate, Wallet Address, Created At
5. Admin click "View Details" trên request #12345
6. Modal hiển thị full details + user info
7. Admin click "Approve" button
8. Confirm popup: "Are you sure to approve this withdraw?"
9. Admin confirm
10. Backend update: status = APPROVED, approved_by = admin_id, approved_at = now()
11. Backend trigger Cobo transfer:
    - Call Cobo API: POST /v2/wallets/{wallet_id}/transfers
    - Body: { "coin": "USDT_TRC20", "amount": "38.82", "to_address": "[user_wallet]" }
12. Cobo API return success: transaction_id
13. Backend update: status = COMPLETED, cobo_tx_id = transaction_id, completed_at = now()
14. Send notification to user: "Your withdraw request has been approved. USDT sent to your wallet."
15. CMS show success message: "Request approved and USDT transferred successfully"

### 🔹 Main Flow 3: User Cancel Pending Request

1. User ở "My Withdraw Requests" page
2. User thấy request status = PENDING
3. User click "Cancel" button
4. Confirm popup: "Cancel withdraw? Your VND will be refunded."
5. User confirm
6. Backend check: status = PENDING? (nếu đã APPROVED → không cho cancel)
7. Backend atomic transaction:
   - Refund balance: current_balance + amount_vnd = 9,000,000 + 1,000,000 = 10,000,000 VND
   - Update request: status = CANCELLED, cancelled_at = now()
8. Return success: "Withdraw cancelled. VND refunded to your balance."
9. User thấy balance updated real-time

---

### 🔸 Error Flows

#### **Error 1: Insufficient Balance**
- **Trigger:** User nhập 5,000,000 VND nhưng balance chỉ có 4,000,000 VND
- **Handling:**
  - Frontend validate: balance ≥ (amount + fee)
  - Show error: "Số dư không đủ. Bạn cần 5,050,000 VND (bao gồm fee 1%) nhưng chỉ có 4,000,000 VND."
  - Disable "Submit" button

#### **Error 2: Below Minimum Amount**
- **Trigger:** User nhập 300,000 VND (< 500,000 VND minimum)
- **Handling:**
  - Frontend validate: amount ≥ 500,000
  - Show error: "Số tiền rút tối thiểu là 500,000 VND"
  - Highlight input field màu đỏ

#### **Error 3: Binance API Timeout**
- **Trigger:** Binance API không response sau 5s
- **Handling:**
  - Backend fallback to cached rate (Redis, max 5 phút cũ)
  - Nếu có cached rate → Use và show warning: "⚠️ Tỷ giá từ 3 phút trước (Binance tạm thời không khả dụng)"
  - Nếu không có cached rate → Block withdraw, show error: "Tỷ giá tạm thời không khả dụng. Vui lòng thử lại sau."

#### **Error 4: Invalid Wallet Address**
- **Trigger:** User nhập wallet address sai format (không phải Tron TRC-20)
- **Handling:**
  - Frontend validate: Regex `/^T[a-zA-Z0-9]{33}$/` (Tron address)
  - Show error: "Địa chỉ ví không hợp lệ. Vui lòng nhập địa chỉ Tron (TRC-20) bắt đầu bằng chữ T."

#### **Error 5: Cobo Transfer Failed**
- **Trigger:** Sau khi approve, Cobo API return error (insufficient gas, network error, etc.)
- **Handling:**
  - Backend catch error
  - Rollback: Refund VND to user balance (amount + fee)
  - Update request: status = FAILED, error_message = Cobo error
  - Send notification: "Withdraw failed due to technical issue. VND refunded to your balance."
  - Log error to Sentry/monitoring system

#### **Error 6: CMS Không Hiển Thị Request**
- **Trigger:** User submit request nhưng CMS không thấy (polling delay)
- **Handling:**
  - Implement webhook: Backend push notification to CMS khi có request mới
  - Fallback: CMS polling mỗi 30s
  - CMS có "Refresh" button để manual reload

#### **Error 7: Double Submit (Race Condition)**
- **Trigger:** User double-click "Submit" button
- **Handling:**
  - Frontend disable button sau khi click (loading state)
  - Backend idempotency check: Nếu user có request PENDING với cùng amount trong 5 phút → Reject duplicate
  - Return error: "Bạn đã có request tương tự đang chờ xử lý"

---

## D. Business Rules / Data Rules

### 1. **Domain-based Access Control**
- Rule: Chỉ users có domain = "hispay.net" được access tính năng
- Implementation: Frontend check `user.domain === 'hispay.net'` → Show/hide UI
- Backend: API endpoint `/withdraw/usdt` check domain, return 403 nếu không phải hispay.net

### 2. **Exchange Rate Source (Binance)**
- Source: Binance API. Cụ thể: Lấy trung bình (hoặc giá tham chiếu) từ top 20 cặp/tỷ giá trên Binance để tránh nhiễu và outlier.
- Update frequency: 
  - **Display frequency:** Mỗi 60 giây (để hiển thị trên UI).
  - **Execution frequency:** Fetch **real-time ngay khi user nhấn Submit** để lấy tỷ giá chính xác nhất tại thời điểm thực hiện lệnh.
- Fallback: Cached rate (Redis, TTL 5 phút).
- Display: "Rate: 25,500 VND/USDT (updated 30s ago)"

### 3. **Fee Structure**
- Fee: Optional (Admin tự input sau khi deal với khách hàng)
- Example: Rút 1,000,000 VND → Fee [Admin input] → Net amount → USDT = Net amount / rate
- Fee không bao gồm Cobo network fee (Tron gas fee ~$1, Cobo trả)

### 4. **Minimum/Maximum Limits**
- Minimum: 500,000 VND (~20 USDT)
- Maximum: Không giới hạn (chỉ giới hạn bởi balance)
- Daily limit: Không có (Phase 1)

### 5. **Balance Deduction & Rate Capture Timing**
- Timing: Ngay khi user submit request (status = PENDING).
- Rate Capture: Hệ thống gọi Binance API ngay lập tức để chốt tỷ giá tại thời điểm submit.
- Rationale: Đảm bảo tỷ giá sát với thị trường nhất và tránh user submit nhiều requests với cùng balance.
- Refund: Nếu cancel hoặc failed → Refund ngay lập tức.

### 6. **Approval Workflow**
- All requests cần manual approval (không auto-approve)
- Approver: Admin hoặc BOD role trên CMS
- Timeout: Không có auto-reject (request pending vô thời hạn cho đến khi approve/reject/cancel)

### 7. **Status State Machine**
```
PENDING → APPROVED → COMPLETED (success path)
PENDING → CANCELLED (user cancel)
PENDING → REJECTED (admin reject)
APPROVED → FAILED (Cobo transfer fail, refund VND)
```

### 8. **Cobo Network**
- Network: Tron (TRC-20) only
- Rationale: Lowest fee (~$1 vs Ethereum ~$10-50)
- Coin: USDT_TRC20

### 9. **Wallet Address Validation**
- Format: Tron address (starts with T, 34 characters alphanumeric)
- Regex: `/^T[a-zA-Z0-9]{33}$/`
- No whitelist: User có thể nhập bất kỳ Tron address nào

### 10. **Atomic Transaction Rule**
- Balance deduction + Request creation phải atomic (DB transaction)
- Nếu 1 trong 2 fail → Rollback cả 2
- Tránh: Balance trừ nhưng request không tạo được

---

## E. Acceptance Criteria (Given/When/Then)

### AC1: Hispay.net user thấy Withdraw to USDT feature
```gherkin
Given user đã login với domain = "hispay.net"
When user navigate to VA Withdraw page
Then user thấy tab "Withdraw to USDT"
And tab hiển thị exchange rate real-time
```

### AC2: Wealify user KHÔNG thấy feature
```gherkin
Given user đã login với domain = "wealify.com"
When user navigate to VA Withdraw page
Then user KHÔNG thấy tab "Withdraw to USDT"
And chỉ thấy "Withdraw to Bank" option
```

### AC3: Auto-calculate USDT khi nhập VND
```gherkin
Given user ở Withdraw to USDT form
And exchange rate = 25,500 VND/USDT
When user nhập amount = 1,000,000 VND
Then system tính fee (theo config Admin input)
And net amount = 1,000,000 - fee
And USDT receive = Net amount / 25,500
And hiển thị USDT tương ứng
```

### AC4: Submit withdraw request thành công
```gherkin
Given user có balance = 10,000,000 VND
And user nhập amount = 1,000,000 VND
And user nhập valid Tron wallet address
When user click "Submit Withdraw"
Then backend trừ balance: 10,000,000 - 1,000,000 = 9,000,000 VND
And backend tạo withdraw_requests record với status = PENDING
And user thấy message: "Withdraw request submitted"
And user redirect to "My Requests" page
```

### AC5: Reject nếu insufficient balance
```gherkin
Given user có balance = 800,000 VND
When user nhập amount = 1,000,000 VND (fee 10,000 → total 1,010,000)
And click "Submit"
Then frontend show error: "Số dư không đủ"
And button "Submit" disabled
And request KHÔNG được tạo
```

### AC6: Reject nếu below minimum
```gherkin
Given user nhập amount = 300,000 VND
When user click "Submit"
Then frontend show error: "Số tiền rút tối thiểu là 500,000 VND"
And highlight input field màu đỏ
```

### AC7: Admin approve request → Cobo transfer
```gherkin
Given Admin login to CMS
And có 1 pending request: user_id=123, amount=1M VND, usdt=38.82, wallet=T...
When Admin click "Approve" và confirm
Then backend update status = APPROVED
And backend call Cobo API: Transfer 38.82 USDT to wallet T...
And Cobo return success với transaction_id
And backend update status = COMPLETED, cobo_tx_id = transaction_id
And user nhận notification: "USDT sent to your wallet"
```

### AC8: Admin reject request → Refund VND
```gherkin
Given Admin ở CMS pending requests page
And có request: user_id=123, amount=1M VND, balance đã trừ
When Admin click "Reject" và nhập reason: "Suspicious activity"
And confirm
Then backend update status = REJECTED, rejection_reason = "Suspicious..."
And backend refund VND: user balance + 1,000,000 = restored
And user nhận notification: "Request rejected. VND refunded."
```

### AC9: User cancel pending request
```gherkin
Given user có request với status = PENDING
When user click "Cancel" và confirm
Then backend check status = PENDING (pass)
And backend refund VND to balance
And backend update status = CANCELLED
And user thấy balance updated real-time
```

### AC10: Không cho cancel nếu đã approved
```gherkin
Given user có request với status = APPROVED
When user click "Cancel"
Then system show error: "Cannot cancel approved request"
And "Cancel" button disabled
```

### AC11: Binance API timeout → Fallback cached rate
```gherkin
Given Binance API timeout hoặc gặp lỗi khi fetch top 20 rates (no response sau 5s)
And Redis có cached rate = 25,400 VND/USDT (2 phút trước)
When user load withdraw form
Then system use cached rate 25,400
And show warning: "⚠️ Tỷ giá từ 2 phút trước"
And user vẫn có thể submit withdraw
```

### AC12: Cobo transfer failed → Refund + status FAILED
```gherkin
Given Admin approved request
And backend call Cobo API
When Cobo return error: "Insufficient gas"
Then backend catch error
And backend refund VND to user balance (1,000,000)
And backend update status = FAILED, error_message = "Insufficient gas"
And user nhận notification: "Withdraw failed. VND refunded."
```

---

## F. Edge Cases (Top 5+)

### 1. **Rate thay đổi giữa submit và approve**
- **Scenario:** User submit với rate 25,500, nhưng 1 giờ sau Admin approve, rate đã là 26,000
- **Handling:** Sử dụng rate tại thời điểm submit (lưu trong DB), KHÔNG recalculate
- **Rationale:** User đã đồng ý với rate khi submit, không fair nếu thay đổi

### 2. **User submit nhiều requests liên tiếp**
- **Scenario:** User submit 3 requests: 1M, 2M, 3M VND (total 6M) nhưng balance chỉ có 5M
- **Handling:** 
  - Request 1: Success (balance 5M - 1M = 4M)
  - Request 2: Success (balance 4M - 2M = 2M)
  - Request 3: Reject (balance 2M < 3M)
- **No race condition:** Atomic transactions đảm bảo sequential processing

### 3. **Admin approve nhầm request đã cancelled**
- **Scenario:** User cancel request, nhưng Admin đang ở CMS (chưa refresh) và click Approve
- **Handling:**
  - Backend check: `WHERE id = ? AND status = 'PENDING'`
  - Nếu status != PENDING → Return error: "Request already cancelled"
  - CMS show error message

### 4. **Binance rate = 0 hoặc null**
- **Scenario:** Binance API trả về rate = 0 (bug hoặc market halt)
- **Handling:**
  - Backend validate: `if (rate <= 0 || rate === null || !isValidAverage(top20_data))`
  - Fallback to cached rate
  - Nếu không có cached → Block withdraw, log error to Sentry (Outlier detected or API failure)

### 5. **Cobo wallet address typo**
- **Scenario:** User nhập sai 1 ký tự trong wallet address (valid format nhưng sai address)
- **Handling:**
  - Không có cách validate ownership (blockchain public)
  - Show warning: "⚠️ Kiểm tra kỹ địa chỉ ví. Giao dịch không thể hoàn tác."
  - User phải confirm: "Tôi đã kiểm tra địa chỉ ví"
  - Nếu gửi nhầm → Không thể refund (blockchain immutable)

### 6. **CMS multiple admins approve cùng lúc**
- **Scenario:** 2 admins cùng click Approve trên cùng 1 request
- **Handling:**
  - DB constraint: `UPDATE ... WHERE id = ? AND status = 'PENDING'`
  - Chỉ 1 admin update thành công (first wins)
  - Admin thứ 2 nhận error: "Request already approved by [Admin1]"

### 7. **User balance âm do concurrent requests**
- **Scenario:** Race condition: 2 requests submit cùng lúc với balance vừa đủ cho 1
- **Handling:**
  - DB transaction isolation level: SERIALIZABLE
  - Atomic balance check + deduct
  - Request thứ 2 sẽ fail validation (balance không đủ)

---

## G. Telemetry / Logging

### Analytics Events

```javascript
// User submit withdraw request
analytics.track('va_withdraw_usdt_submitted', {
  user_id: 'xxx',
  amount_vnd: 1000000,
  fee_vnd: 10000,
  usdt_amount: 38.82,
  exchange_rate: 25500,
  wallet_address: 'T...',
  timestamp: Date.now()
});

// Admin approve request
analytics.track('va_withdraw_usdt_approved', {
  request_id: 'xxx',
  admin_id: 'yyy',
  approval_time_minutes: 15, // Time from submit to approve
  timestamp: Date.now()
});

// Cobo transfer success
analytics.track('va_withdraw_usdt_completed', {
  request_id: 'xxx',
  cobo_tx_id: 'zzz',
  transfer_time_seconds: 45,
  timestamp: Date.now()
});

// Withdraw failed
analytics.track('va_withdraw_usdt_failed', {
  request_id: 'xxx',
  error_type: 'cobo_transfer_failed',
  error_message: 'Insufficient gas',
  timestamp: Date.now()
});

// User cancel
analytics.track('va_withdraw_usdt_cancelled', {
  request_id: 'xxx',
  user_id: 'yyy',
  pending_duration_minutes: 30,
  timestamp: Date.now()
});
```

### Error Logging

```javascript
// Binance API error
logger.error('Binance API failed', {
  endpoint: '/api/v3/ticker/price',
  symbol: 'USDTVND',
  error_code: 'TIMEOUT',
  fallback_rate_used: 25400,
  cached_rate_age_minutes: 2
});

// Cobo transfer error
logger.error('Cobo transfer failed', {
  request_id: 'xxx',
  cobo_endpoint: '/v2/wallets/{id}/transfers',
  error_response: {...},
  status_code: 500,
  action_taken: 'refund_vnd'
});

// Invalid wallet address
logger.warn('Invalid wallet address submitted', {
  user_id: 'xxx',
  wallet_address: 'invalid_format',
  validation_failed: 'regex_mismatch'
});
```

### Metrics to Monitor

1. **Success Rate:**
   - Metric: `(COMPLETED requests / Total submitted) * 100`
   - Baseline: Unknown (new feature)
   - Target: > 95%
   - Alert if: < 90% (investigate Cobo issues)

2. **Approval Time:**
   - Metric: P50, P95 time từ submit → approve
   - Target: P50 < 30 phút, P95 < 2 giờ
   - Alert if: P95 > 4 giờ (Admin bottleneck)

3. **Binance API Uptime:**
   - Metric: % successful Binance API calls
   - Target: > 99%
   - Alert if: < 95% (consider alternative rate source)

4. **Cobo Transfer Success Rate:**
   - Metric: `(Cobo success / Cobo attempts) * 100`
   - Target: > 99%
   - Alert if: < 95% (Cobo integration issue)

5. **Cancellation Rate:**
   - Metric: `(CANCELLED requests / Total submitted) * 100`
   - Baseline: TBD
   - Alert if: > 20% (UX issue? Approval too slow?)

6. **Exchange Rate Deviation:**
   - Metric: % difference giữa Binance rate và market average
   - Alert if: > 5% (Binance data issue)

---

## Summary Table

| Section | Content |
|---------|---------|
| **Outcome** | Hispay.net users rút VND → USDT qua Cobo, giảm 30% thời gian, tỷ giá real-time Binance |
| **In-scope** | Domain flag, rate display, auto-calc, balance deduct, CMS approval, Cobo transfer, cancel, notifications (9 features) |
| **Out-scope** | Multi-currency, auto-approval, batch, API key UI, rate chart (5 deferred) |
| **Main flows** | 3 flows (User submit, Admin approve, User cancel) |
| **Error flows** | 7 error cases (Insufficient balance, Below min, Binance timeout, Invalid wallet, Cobo fail, CMS delay, Double submit) |
| **Business rules** | 10 rules (Domain access, Rate source, Fee 1%, Min 500K, Balance timing, Manual approval, Status machine, Tron only, Wallet validation, Atomic transaction) |
| **ACs** | 12 testable scenarios (Domain check, Auto-calc, Submit, Reject cases, Approve, Reject, Cancel, Binance fallback, Cobo fail) |
| **Edge cases** | 7 cases (Rate change, Multiple requests, Approve cancelled, Rate=0, Wallet typo, Concurrent approve, Balance race) |
| **Telemetry** | 5 events, 3 error logs, 6 metrics |

---

**Spec Length:** ~1,450 words ✅ (Medium complexity)  
**Technical Feasibility:** Verified with Cobo WaaS 2.0 docs (Vault 1) ✅  
**Binance API:** Public endpoint, no auth required ✅  
**Ready for Review:** Yes ✅

---

## 📋 NEXT STEPS FOR BA/PO

1. **Review Spec:** Xác nhận requirements đúng
2. **Clarify:**
   - Binance API có cần API key không? (public endpoint không cần)
   - CMS approval: 1-level hay 2-level (Maker-Checker)?
   - Daily/monthly withdraw limit cần không?
3. **After Approval:** Proceed to Step 3 (Frozen 3 Quality Gate)
