# Failure Analysis — Lab 18: Production RAG

**Họ và tên học viên:** [Họ và tên]  
**Khóa:** K4 - Track 3A  

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.8167 | 0.7143 | -0.1024 |
| Answer Relevancy | 0.7120 | 0.6914 | -0.0206 |
| Context Precision | 0.9250 | 0.9417 | +0.0167 |
| Context Recall | 0.9000 | 0.8889 | -0.0111 |

*(Lưu ý: Sự sụt giảm của Faithfulness và Answer Relevancy ở bản Production chủ yếu do tài khoản OpenRouter bị giới hạn rate limit - Lỗi 402 In-flight budget exhausted - khiến RAGAS không thể chấm điểm chính xác và trả về 0 cho nhiều câu).*

## Bottom-5 Failures

### #1
- **Question:** Nhân viên được nghỉ bao nhiêu ngày khi kết hôn?
- **Expected:** Nhân viên được nghỉ 3 ngày làm việc có lương khi kết hôn, không trừ vào phép năm.
- **Got:** (Không thể sinh câu trả lời đúng / Hoặc lỗi Rate Limit).
- **Worst metric:** faithfulness (0.0)
- **Error Tree:** Output sai → Context đúng? (Có) → Query OK? (Có) → Lỗi sinh văn bản hoặc chấm điểm.
- **Root cause:** Khả năng cao do API gọi dồn dập khiến RAGAS chấm lỗi (0 điểm), hoặc do LLM sinh câu trả lời bị lẫn lộn thông tin.
- **Suggested fix:** Cải thiện cấu hình API OpenRouter, giảm temperature.

### #2
- **Question:** Bao lâu phải đổi mật khẩu một lần?
- **Expected:** Theo chính sách hiện hành (v2.0), mật khẩu phải được thay đổi mỗi 120 ngày.
- **Got:** (Không thể sinh câu trả lời đúng / Trả lời sai 90 ngày).
- **Worst metric:** faithfulness (0.0)
- **Error Tree:** Output sai → Context đúng? (Có chứa cả 2) → Xung đột phiên bản.
- **Root cause:** Trong dữ liệu có cả chính sách cũ (90 ngày) và mới (120 ngày). Mô hình bị nhiễu không biết cái nào có hiệu lực.
- **Suggested fix:** Bổ sung bước lọc Metadata (Qdrant filter) theo version mới nhất ở bước Retrieval.

### #3
- **Question:** Có cần kích hoạt xác thực đa yếu tố (MFA) không?
- **Expected:** Có, bắt buộc kích hoạt MFA theo v2.0.
- **Got:** (Không thể sinh câu trả lời đúng / Trả lời không cần theo v1.0).
- **Worst metric:** faithfulness (0.0)
- **Error Tree:** Output sai → Context đúng? (Có chứa cả 2) → Xung đột phiên bản.
- **Root cause:** Sự tồn tại của văn bản chính sách cũ (v1.0 không yêu cầu MFA) gây nhiễu cho mô hình sinh văn bản.
- **Suggested fix:** Lọc cứng Metadata `deprecated=True` khỏi quá trình tìm kiếm.

### #4
- **Question:** Khi phát hiện malware trên máy, nhân viên có nên tự xử lý không?
- **Expected:** KHÔNG. Tuyệt đối không tự xử lý.
- **Got:** (Không thể sinh câu trả lời đúng / Lỗi Rate Limit).
- **Worst metric:** faithfulness (0.0)
- **Error Tree:** Lỗi ở khâu Generation.
- **Root cause:** RAGAS không đánh giá được do lỗi 402 từ OpenRouter, hoặc LLM sinh văn bản thiếu chữ "KHÔNG" dứt khoát.
- **Suggested fix:** Thêm System Prompt nhắc nhở LLM trả lời "CÓ/KHÔNG" trước khi giải thích.

### #5
- **Question:** Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không?
- **Expected:** KHÔNG. Chỉ nhân viên chính thức mới được hưởng.
- **Got:** (Không thể sinh câu trả lời đúng).
- **Worst metric:** faithfulness (0.0)
- **Error Tree:** Lỗi ở khâu Generation / Context extraction.
- **Root cause:** Document chunking lấy được chunk nói về bảo hiểm nhưng không giữ được điều kiện "áp dụng cho nhân viên chính thức".
- **Suggested fix:** Đảm bảo Structure-aware chunking cắt đủ câu điều kiện.

## Case Study (cho presentation)

**Question chọn phân tích:** Bao lâu phải đổi mật khẩu một lần?

**Error Tree walkthrough:**
1. Output đúng? → Không, mô hình trả lời theo chính sách cũ (90 ngày).
2. Context đúng? → Có, Context Precision rất cao (0.94), tức là tài liệu chứa đáp án 120 ngày (v2.0) ĐÃ CÓ trong top K, nhưng tài liệu cũ cũng bị lấy lên.
3. Query rewrite OK? → Truy vấn khớp với nội dung của cả hai văn bản cũ và mới.
4. Fix ở bước: Bổ sung logic lọc Metadata theo `version` ở bước Retrieval (Module 2).

**Nếu có thêm 1 giờ, sẽ optimize:**
- Thêm trường `effective_date` hoặc `version` vào metadata tự động (Module 5).
- Cấu hình lại Qdrant trong `m2_search.py` để sử dụng `Filter` từ chối các chunk thuộc văn bản cũ.
