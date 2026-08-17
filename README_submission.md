# Báo Cáo Nộp Bài Lab 17 - Multi-Memory Agent với Zep

**Họ và tên**: Nhu Van Hung  
**Mã SV**: 2A202601372  

---

## 1. Phân Tích Kỹ Thuật (Core Questions)

### Câu 1: Tầng bộ nhớ quan trọng nhất trong bộ test
Tầng **Long-term / Declarative Memory** (Context Block) và **Episodic Memory** đóng vai trò quyết định nhất. Cụ thể, các test case như `E02`, `E03`, `E08`, `E09` đòi hỏi truy xuất chính xác các fact/preference xuyên suốt nhiều session của user (ví dụ: `lan-lab17` đổi coding style sang Python hay cấu hình database URI mới). Nếu không có tầng này, agent sẽ không nhớ được context quá quá trình hội thoại.

### Câu 2: Trade-off giữa Zep Context Block vs Tự Build (Redis + Qdrant)
- **Zep Context Block (Managed)**: Giúp tự động trích xuất graph, phân loại facts/episodes, xử lý recency/validity ranges và tự động lắp ráp context theo đợt relevance. Ưu điểm là giảm thời gian phát triển, tự động hóa compaction; nhược điểm là phụ thuộc API bên ngoài, chi phí Cloud API và độ trễ mạng.
- **Tự build (Redis + Qdrant)**: Giúp toàn quyền kiểm soát dữ liệu, latency cực thấp, chi phí vận hành cố định; tuy nhiên phải tự triển khai pipeline trích xuất tri thức, xử lý xung đột fact (recency wins) và tự viết logic compaction phức tạp.

### Câu 3: Guardrail chống Memory Poisoning
Để ngăn chặn người dùng vô tình hoặc cố ý nạp thông tin sai lệch/độc hại vào bộ nhớ long-term:
1. **Validation & Filtering Guardrail**: Kiểm tra và loại bỏ các chỉ thị độc hại (prompt injection/poisoning) trước khi lưu vào memory graph.
2. **Consent & User Scope Isolation**: Xác thực quyền `memory_opt_in` (như trong `data/consent.json`) và cách ly tuyệt đối dữ liệu theo `user_id` để tránh cross-user poisoning.
3. **Provenance & Fact Expiration**: Gắn mốc thời gian (validity range/timestamp) cho facts để thông tin mới đúng đắn có thể ghi đè (overrule) fact cũ.

---

## 2. Phân Tích Thực Hành & Benchmark Case

- **E07 (Mixed Context)**: Truy vấn yêu cầu kết hợp cả thông tin cá nhân của user trong **Long-term Memory** (Context Block) và tri thức dùng chung từ **Semantic Memory** (Payment rules/Idempotency-Key), được lắp ráp chuẩn xác nhờ `ContextBudgetManager` (10/4/3/3).
- **E08 (Conflict & Recency)**: Khi user cập nhật thông tin mới (ví dụ đổi preference/backend), Zep tự động cập nhật validity range cho fact cũ và ưu tiên thông tin mới nhất (`Recency wins`), giúp agent đưa ra câu trả lời chính xác mà vẫn bảo toàn lịch sử.
- **E10 (Compaction)**: Dù hội thoại kéo dài và số turn cũ bị nén (compacted), các durable notes và deadline/task mở quan trọng vẫn được duy trì trong context mà không bị mất mát khi cắt giảm window size.
- **Privacy Drill (Forget)**: Xóa toàn bộ memory cá nhân bằng `python -m src.forget --user-id minh-lab17`. Xác minh dữ liệu bị xóa hoàn toàn khỏi Zep và Redis (`Redis user keys remaining: 0`).
