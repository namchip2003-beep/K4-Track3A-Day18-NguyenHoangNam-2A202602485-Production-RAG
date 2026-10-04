# Reflection & Action Plan

**Họ và tên:** Nguyễn Hoàng Nam  
**Khóa:** K4 - Track 3A  

## Phần 1: Lecture Mapping
Trong bài Lab 18 này, em đã lập trình các module khớp với lý thuyết RAG nâng cao như sau:
1. **Advanced Chunking (M1):** Em đã viết thuật toán Semantic Chunking (dựa vào độ tương đồng ngữ nghĩa) và Hierarchical Chunking (chia parent-child) để bảo toàn ngữ cảnh.
2. **Hybrid Search (M2):** Kết hợp thuật toán BM25 (tìm kiếm từ khóa) và Qdrant (tìm kiếm ngữ nghĩa vector), sau đó hợp nhất kết quả bằng RRF (Reciprocal Rank Fusion).
3. **Cross-Encoder Reranking (M3):** Dùng mô hình `BAAI/bge-reranker-v2-m3` để đối chiếu chéo cặp (query, document) tăng độ chính xác tuyệt đối.
4. **RAGAS Evaluation (M4):** Khai báo và sử dụng các metric tiên tiến như Faithfulness, Answer Relevancy, Context Precision/Recall để tự động chấm điểm bằng LLM.
5. **Enrichment (M5):** Áp dụng LLM (thông qua OpenRouter) để tự động sinh tóm tắt, HyQA và câu Contextual Prepend cho mỗi đoạn văn trước khi embedding, giúp giải quyết triệt để việc thất thoát ngữ cảnh.

## Phần 2: Challenges & Debugging
- **Khó khăn 1:** Gặp lỗi 402 Rate Limit (In-flight budget exhausted) từ OpenRouter do hàm đánh giá của RAGAS gửi cùng lúc quá nhiều request để chấm điểm song song.
- **Cách khắc phục:** Sửa lại file `m4_eval.py`, cấu hình thêm tham số `run_config=RunConfig(max_workers=1)` để RAGAS bắt buộc phải chạy tuần tự từng câu hỏi một, tránh bị giới hạn API.
- **Khó khăn 2:** Mô hình LLM khi chấm điểm đánh giá điểm Faithfulness thấp cho một số câu hỏi liên quan đến chính sách vì hệ thống lấy nhầm văn bản cũ.
- **Cách khắc phục:** Lập bảng Error Tree Analysis để nhận diện nguyên nhân gốc là "Xung đột phiên bản". Đề xuất giải pháp thêm siêu dữ liệu (Metadata) phân biệt version ở bước Chunking.

## Phần 3: Action Plan
- **Dự định áp dụng:** Em sẽ áp dụng module Hybrid Search và Cross-Encoder vào hệ thống RAG phục vụ tra cứu quy trình trong công việc cá nhân. Kỹ thuật Contextual Prepend và HyQA ở M5 cũng cực kỳ tiềm năng để tìm kiếm tài liệu dài bị lặp ý.
- **Cải thiện tương lai:** Sẽ triển khai Hard-filtering (lọc cứng) bằng Qdrant Payload để chặn triệt để các tài liệu bị gắn thẻ `deprecated=True` (như chính sách năm 2023), giúp mô hình không bao giờ sinh ra câu trả lời lỗi thời nữa.
