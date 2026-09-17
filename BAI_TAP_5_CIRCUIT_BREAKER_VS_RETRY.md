# BÀI TẬP 5: TRADE-OFF: CIRCUIT BREAKER HAY RETRY PATTERN?

---

## 1. Bối cảnh bài toán

Order‑Service cần gọi sang hệ thống của đơn vị vận chuyển **Giao Hàng Nhanh (GHTK)** để tạo vận đơn. Hệ thống của GHTK có đặc thù:
- Thường xuyên bị **"chớp nháy" mạng (Network Glitch)**
- Lâu lâu có 1 request gọi sang bị **Timeout**
- Nếu gọi lại ngay lập tức thì lại **thành công (Transient Failure)**

---

## PHẦN 1 – ĐỀ XUẤT ĐA GIẢI PHÁP

### Giải pháp 1: Dùng Circuit Breaker thuần túy

**Ý tưởng:** Đặt một "cầu dao" giữa Order-Service và GHTK. Khi số lần gọi thất bại liên tiếp vượt ngưỡng, Circuit Breaker sẽ **"mở"** – tự động ngắt mọi request tiếp theo mà không gửi đi, trả về lỗi ngay lập tức (fail-fast). Sau một khoảng thời gian "nghỉ" (half-open state), nó cho phép 1 request test xem đã phục hồi chưa.

**Ưu điểm:**
- Ngăn hệ thống bị quá tải bởi các request liên tục gửi đến GHTK đang lỗi
- Phát hiện nhanh khi GHTK sập hẳn, tránh lãng phí tài nguyên (thread, connection)
- Fail-fast → phản hồi nhanh cho người dùng

**Nhược điểm:**
- **Không phù hợp với Transient Failure (lỗi chớp nháy mạng):** Vì Circuit Breaker sau khi mở sẽ ngắt toàn bộ, nếu lỗi chỉ kéo dài vài giây thì hệ thống vẫn bị từ chối dịch vụ không đáng có
- Không tự thử lại request → người dùng nhận lỗi mà không có cơ hội để request thành công

---

### Giải pháp 2: Dùng Retry Pattern kết hợp Exponential Backoff

**Ý tưởng:** Khi gặp lỗi, hệ thống tự động thử lại request với khoảng cách thời gian ngày càng tăng giữa các lần retry (ví dụ: 2s → 4s → 8s).

**Ưu điểm:**
- **Rất phù hợp với Transient Failure:** Lỗi chớp nháy mạng thường tự hết → Retry giúp request thành công ở lần thử tiếp theo mà không cần can thiệp thủ công
- Tăng khả năng chịu lỗi (resilience) mà không cần thay đổi logic nghiệp vụ
- Exponential Backoff tránh "thundering herd" (đồng loạt gửi lại cùng lúc)

**Nhược điểm:**
- Tăng độ trễ (latency) cho mỗi request do phải chờ retry
- Nếu GHTK thực sự sập hẳn (System Crash), Retry sẽ lãng phí tài nguyên khi liên tục gửi request đến hệ thống không phản hồi
- Cần quản lý state để tránh retry vô hạn

---

## PHẦN 2 – SO SÁNH

| Tiêu chí | Transient Failure (Chớp nháy mạng) | System Crash (GHTK sập hẳn) |
|---|---|---|
| **Circuit Breaker** | ❌ **Không phù hợp** – Mở cầu dao quá sớm → từ chối request dù GHTK đang phục hồi | ✅ **Rất phù hợp** – Phát hiện lỗi liên tục → ngắt hoàn toàn, bảo vệ hệ thống |
| **Retry + Backoff** | ✅ **Rất phù hợp** – Tự thử lại, mạng đã ổn → request thành công | ❌ **Không phù hợp** – Retry vô nghĩa, lãng phí tài nguyên, có thể làm hệ thống tệi hơn |

**Kết luận:** Bối cảnh đề bài là **Transient Failure (chớp nháy mạng)** → **Retry Pattern với Exponential Backoff là giải pháp tối ưu.**

---

## PHẦN 3 – TRIỂN KHAI (YAML)

```yaml
retry-config:
  service: ghtk-create-shipment
  max-attempts: 3
  backoff:
    base-delay: 2000ms
    multiplier: 1
    type: fixed
  retry-on:
    - exception: TimeoutException
  fail-fast: false
  idempotency-key-header: X-Idempotency-Key
```

**Giải thích cấu hình:**
- `max-attempts: 3` → Tối đa 3 lần thử lại (lần đầu + 2 lần retry)
- `base-delay: 2000ms` → Chờ 2 giây giữa mỗi lần retry
- `retry-on: TimeoutException` → Chỉ retry khi gặp lỗi Timeout
- `idempotency-key-header` → Header để đảm bảo tính idempotency

---

## BẪY DỮ LIỆU – IDEMPOTENCY

### Vấn đề:
Nếu GHTK **trừ tiền tài khoản trên mỗi lần gọi API**, việc Retry sẽ gây ra rủi ro:

Khi request tạo vận đơn **thành công** nhưng **response bị mất do timeout** → Client (Order-Service) cho rằng lỗi → Retry → GHTK nhận request thứ 2 → **Trừ tiền lần 2** → Người dùng bị tính phí **2 lần** cho 1 đơn hàng.

### Khái niệm Idempotency:
> **Idempotency** (Tính idempotent) là tính chất mà việc thực hiện một thao tác **nhiều lần** cũng cho **cùng một kết quả** như thực hiện **một lần**.

### Giải pháp:
- Sử dụng **Idempotency Key** (khóa duy nhất cho mỗi request, ví dụ: `orderId_xyz`)
- GHTK lưu trữ Idempotency Key → Nếu nhận request trùng key, trả về kết quả lần trước mà **không thực hiện lại thanh toán**
- Yêu cầu GHTK hỗ trợ header `X-Idempotency-Key` trên API

**Tóm lại:** Khi triển khai Retry Pattern cho hệ thống có thanh toán, **bắt buộc** phải có cơ chế Idempotency để tránh duplicate transaction.
