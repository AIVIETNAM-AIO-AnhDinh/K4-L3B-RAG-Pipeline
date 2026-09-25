# Báo cáo đóng góp cá nhân

## Thông tin

- **Họ và tên:** Vũ Hải Minh
- **Mã học viên:** 2A202602452
- **Nhóm:** 5changlinhngulam
- **Repository/branch:** `AIVIETNAM-AIO-AnhDinh/K4-L3B-RAG-Pipeline` / `Minhz`

## Phần việc đã thực hiện

| Module/deliverable | Việc tôi trực tiếp làm | File/commit | Trạng thái |
|---|---|---|---|
| Task 4 — chunking, embedding, indexing | Đọc Markdown chuẩn hóa; tạo chunk có ID ổn định, `chunk_index` và metadata kế thừa; embed qua `embed_texts()`; upsert vào collection Chroma cosine và dọn ID không còn trong corpus. Cấu hình: recursive, 800 ký tự, overlap 150; mặc định `BAAI/bge-m3` 1024 chiều. | `src/task4_chunking_indexing.py`, commit `ebb3ab7` | Done — đã chạy trong `.venv`; Chroma có 243 chunk từ 12 tài liệu. |
| Task 5 — dense search | Embed query bằng `embed_texts()` của Task 4; đổi cosine distance thành similarity; trả kết quả `dense` theo schema, bỏ ID lặp, sắp điểm giảm dần và giới hạn `top_k`. | `src/task5_semantic_search.py`, commit `ebb3ab7` | Done — contract test với collection giả đã qua; chưa kiểm tra truy vấn trên Chroma đã index. |
| Task 6 — lexical search | Nạp cùng corpus chunk từ Chroma; xây BM25 với unigram âm tiết và bigram kề nhau cho tiếng Việt, Lucene-style IDF; trả kết quả `bm25` theo điểm giảm dần. | `src/task6_lexical_search.py`, commit `ebb3ab7` | Done — contract suite pass. |

## Quyết định kỹ thuật quan trọng

1. **Chunk 800 ký tự, overlap 150, recursive.** Ghi chú trong Task 4 chọn 800 vì phần lớn điều khoản tiếng Việt dài khoảng 300–900 ký tự; overlap giữ ngữ cảnh ở ranh giới. Đánh đổi là nhiều nội dung lặp và chi phí embedding cao hơn cấu hình chunk nhỏ/overlap thấp.

2. **Giữ chung embedding và schema nhưng để dense/BM25 chấm điểm riêng.** Task 4 mặc định dùng `BAAI/bge-m3` 1024 chiều; Task 5 gọi lại `embed_texts()` để query không lệch model hoặc dimension. Chroma cosine distance được chuyển thành similarity. Task 6 bổ sung bigram để bắt cụm từ tiếng Việt, còn Lucene-style IDF giữ trọng số từ phổ biến dương. Đổi lại BM25 có index token lớn hơn và score của hai bộ tìm kiếm khác thang đo; việc hợp nhất thứ hạng để Task 7 xử lý.

## Kiểm thử và kết quả

- Đã chạy `.\.venv\Scripts\python.exe -m pytest tests/test_contracts.py -q`: **15 passed**. Có một cảnh báo pytest không ghi được `.pytest_cache`; không có test thất bại. Test semantic search dùng embedding và collection giả, không gọi API ngoài.
- Đã kiểm tra collection `rag_documents`: 243/243 ID duy nhất, thuộc 12 tài liệu nguồn (136 chunk legal, 107 news); mọi chunk đọc lại đều có nội dung và metadata `source`, `title`, `doc_type`, `chunk_index`. Đã kiểm tra mẫu metadata có thêm `url`, `category`, `audience` khi nguồn cung cấp.

## Hạn chế và bước tiếp theo

- Thử cùng một nhóm query trên dense và BM25 trong `.venv`; ghi lại thứ hạng, metadata nguồn và metric đánh giá trước khi hiệu chỉnh retrieval hoặc chunking.

## Xác nhận đóng góp

Tôi xác nhận nội dung trên mô tả phần việc Task 4–6 được ghi nhận trong commit `ebb3ab7` trên branch `Minhz`; trạng thái kiểm thử phản ánh lần chạy khi hoàn thiện báo cáo này.

- **Ngày:** 2026-09-25
- **Tên thành viên:** Vũ Hải Minh 'dtc-z'
