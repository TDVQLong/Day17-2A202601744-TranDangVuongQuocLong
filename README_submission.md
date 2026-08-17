# Báo Cáo Nộp Bài Lab 17 - Multi-Memory Agent với Zep

**Họ và tên học viên:** Trần Đặng Vương Quốc Long  
**Mã sinh viên:** 2A202601744  

---

## 1. Trả Lời 3 Câu Hỏi Cốt Lõi (LAB.md Section 5.2)

### Câu 1: Layer quan trọng nhất trong bộ test này và case minh họa
- **Layer quan trọng nhất**: **Long-term Memory (Zep Context Block & Facts)**.
- **Lý do & Case minh họa**: Trong bộ test này, các case thuộc Long-term Memory (`E02`, `E03`, `E08`, `E09`) chiếm số điểm cao nhất (20 điểm). Dữ liệu preference và open-loop tasks được lưu giữ xuyên suốt nhiều phiên làm việc khác nhau (cross-session). Ví dụ:
  - `E02`: Truy xuất preference ngôn ngữ Python cho project cá nhân `ORCHID-27` ở thread mới.
  - `E03`: Nhắc lại deadline open-loop `16:00` cho benchmark report.
  - `E08`: Áp dụng quy tắc recency, khi chuyển sang project công ty `BLUEBIRD-42` thì preference cập nhật thành `TypeScript` + `NestJS`.
  - `E09`: Đảm bảo User Isolation, dữ liệu của `lan-lab17` (`LOTUS-88`, `Java`, `Spring Boot`) hoàn toàn tách biệt, không bị rò rỉ sang `minh-lab17`.

### Câu 2: Trade-off giữa Managed Zep Context Block vs Tự dựng Redis + Qdrant
- **Managed Zep Cloud V3**:
  - *Ưu điểm*: Tự động trích xuất Facts, hợp nhất Knowledge Graph, tự động duy trì sliding window & compaction, và tự tạo Context Block thông minh tối ưu theo query mà không cần thiết kế pipeline phức tạp.
  - *Nhược điểm*: Phụ thuộc vào vendor bên thứ ba, chịu giới hạn rate limit và yêu cầu kết nối cloud.
- **Self-hosted Redis + Qdrant**:
  - *Ưu điểm*: Kiểm soát 100% dữ liệu tại chỗ (On-premise/GDPR compliance), latency cực thấp, không tốn chi phí SaaS.
  - *Nhược điểm*: Phải tự lập trình từ đầu toàn bộ logic chunking, vector embedding, graph extraction, conflict resolution (recency wins) và token budget trimming.

### Câu 3: Guardrail chống Memory Poisoning (Nhiễm độc bộ nhớ)
1. **Consent & Privacy Gate**: Kiểm tra registry consent (`data/consent.json`) và tự động redact thông tin cá nhân (PII như email, phone) trước khi nạp vào bộ nhớ bền vững.
2. **Instruction Isolation trong Maintenance**: Các tiến trình bảo trì (Heartbeat, Episodic maintenance) chỉ được phép clean-up, de-duplicate local notes, không được phép đưa các câu lệnh prompt injection từ user input vào system instruction/persona.
3. **Right to be Forgotten**: Cung cấp API xóa dữ liệu theo user ID (`python -m src.forget --user-id ...`), đảm bảo dữ liệu cá nhân bị xóa triệt để trên cả Zep user graph và Redis store.

---

## 2. Phân Tích Kết Quả Benchmark

### Bảng Kết Quả So Sánh (từ `reports/comparison.md`)

| Chỉ số | Memory-enabled (Student) | No-memory Baseline | Chênh lệch (Delta) |
| --- | ---: | ---: | ---: |
| **Evidence Hit Rate** | **100.0%** | **18.2%** | **+81.8%** |
| **Số Case Đạt (Pass)** | **11/11** | **2/11** | **+9 cases** |
| **Thời gian phản hồi TB (ms)** | 1015.8 ms | 0.0 ms | +1015.8 ms |
| **Mức giảm Token trung bình** | 14.2% | 81.8% | -67.6% |

### 4 Câu Phân Tích Chi Tiết:

1. **Layer có Hit Rate thấp nhất ở No-memory Baseline**: 
   - Ngoại trừ `short_term` (`E01`, `E10`) đạt pass ở baseline nhờ tin nhắn nằm ngay trong window hiện tại, tất cả các layer **Long-term**, **Episodic** và **Semantic** đều có hit rate **0%** nếu không bật bộ nhớ bền vững. Khi bật `StudentMemory`, tất cả 4 layer đều khôi phục thành công đạt Hit Rate **100% (11/11 PASS)**.
2. **Query truy xuất nhiều token nhất**:
   - `E03` (Long-term open loop): **1476 tokens** và `E02` (Long-term preference): **1472 tokens**. Lý do là Zep Context Block tổng hợp toàn bộ profile summary, relevant facts và trajectory lịch sử của user Minh.
3. **Case Mixed E07**:
   - Kết hợp giữa **Long-term Memory** (sở thích ngôn ngữ Python của user Minh) và **Semantic Memory** (`Idempotency-Key` từ guideline POST payment). Cả hai evidence bắt buộc này đều được tổng hợp chính xác vào context.
4. **Token Reduction vs Hit Rate**:
   - Baseline No-memory có Token Reduction cao (81.8%) thực chất là do bỏ qua toàn bộ dữ liệu bộ nhớ (không retrieve được thông tin), dẫn đến Hit Rate cực thấp (18.2%). Token reduction chỉ có ý nghĩa khi đi kèm với Evidence Hit Rate cao.

---

## 3. Phân Tích Kỹ Thuật: Recency & Compaction

- **E08 Recency (Xử lý xung đột thông tin)**:
  - Khi thông tin ban đầu (Python) xung đột với thông tin mới trong dự án `BLUEBIRD-42` (TypeScript + NestJS), Zep Graph áp dụng quy tắc *Recency Wins*, ưu tiên các thông tin và ràng buộc mới nhất cho dự án hiện tại mà vẫn giữ lại lịch sử provenances.
- **E10 Compaction (Nén bộ nhớ ngắn hạn)**:
  - Dù 5 lượt hội thoại rác (filler turns) làm tràn buffer raw transcript, cơ chế Sliding Window kết hợp **Durable Notes** vẫn trích xuất và bảo tồn thành công thông tin cốt lõi `REVIEW-DEADLINE-1600` (Friday at 16:00).
