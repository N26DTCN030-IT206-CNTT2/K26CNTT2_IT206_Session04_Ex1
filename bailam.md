# BÁO CÁO BÀI TẬP VỀ NHÀ - BUỔI 4
**Môn học:** Kỹ năng Prompting với AI (IT206-K26)  
**Chủ đề:** Thiết kế giải pháp Prompting ứng dụng thực tế cho Kỹ thuật Prompting Nâng cao (Logic & Chuỗi)  
**Sinh viên thực hiện:** Lê Hồ Thanh Thảo  
**MSSV:** N26DTCN030  

---

## 1. Mục tiêu và Bối cảnh bài toán

### 1.1. Mục tiêu
* Ứng dụng các kỹ thuật Prompting nâng cao gồm **Chain-of-Thought (CoT)**, **Prompt Chaining** và **Xử lý đứt gãy ngữ cảnh (Continue Generating)** để tự động hóa quy trình nghiệp vụ thực tế.
* Rèn luyện tư duy phân rã bài toán, kiểm soát dữ liệu đầu vào/đầu ra (I/O) và bẫy lỗi ngoại lệ trước khi triển khai thành mã nguồn.

### 1.2. Bối cảnh thực tế
Một cửa hàng bán lẻ đa kênh mỗi ngày nhận hàng trăm tin nhắn/phản hồi từ khách hàng (hỏi hàng, báo lỗi sản phẩm, khiếu nại giao chậm, yêu cầu đổi trả) dưới dạng văn bản thô không đồng nhất. Nhân viên trực page thường xuyên bỏ sót đơn gấp, nhập sai mã đơn hoặc mất nhiều thời gian tổng hợp báo cáo cuối ngày.

Giải pháp đặt ra là xây dựng một **chuỗi Prompt tự động (Prompt Chaining)** gồm 3 mắt xích nối tiếp nhau để trích xuất dữ liệu, suy luận mức độ ưu tiên (CoT) và xuất báo cáo tổng hợp cho quản lý cửa hàng, đồng thời xử lý được tình huống dữ liệu thiếu/sai lệch hoặc hội thoại quá dài bị ngắt quãng.

---

## 2. Bước 1: Thiết kế giải pháp (Input / Output & Ràng buộc dữ liệu)

### 2.1. Đặc tả Input và Output của hệ thống

| Thành phần | Mô tả chi tiết | Định dạng dữ liệu |
| :--- | :--- | :--- |
| **Input (Đầu vào)** | Danh sách tin nhắn/phản hồi thô của khách hàng trong ngày (bao gồm: tên khách, số điện thoại, nội dung phản hồi, mã đơn hàng nếu có, thời gian gửi). | Văn bản thô (Plain Text) bọc trong thẻ `<raw_messages>` |
| **Output Bước 1 (Trích xuất & Làm sạch)** | Danh sách các bản ghi đã chuẩn hóa trường thông tin, gắn cờ cảnh báo nếu thiếu dữ liệu hoặc sai định dạng. | Bảng Markdown hoặc JSON chuẩn |
| **Output Bước 2 (Suy luận CoT & Phân loại)** | Phân tích nguyên nhân từng ca, xác định mức độ ưu tiên (`P1 - Khẩn cấp`, `P2 - Trung bình`, `P3 - Thông thường`) và đề xuất hướng xử lý. | Văn bản có cấu trúc từng bước suy luận + Bảng phân loại |
| **Output Bước 3 (Báo cáo tổng hợp)** | Báo cáo tổng kết cuối ngày gửi quản lý kèm danh sách các ca P1 cần gọi điện xử lý ngay. | Báo cáo Markdown hoàn chỉnh |

### 2.2. Ràng buộc và bẫy dữ liệu lường trước (Edge Cases)
Để tránh việc AI tự suy diễn (Hallucination) hoặc bỏ sót lỗi khi dữ liệu thực tế bị nhiễu, hệ thống thiết lập các quy tắc điều kiện xử lý lỗi như sau:

| Tình huống dữ liệu lỗi | Quy tắc điều kiện bắt buộc cho AI |
| :--- | :--- |
| **Thiếu mã đơn hàng hoặc SĐT** | Không được tự bịa mã số. Ghi rõ giá trị là `MISSING_DATA` và gắn cờ `[CẦN BỔ SUNG THÔNG TIN]`. |
| **Nội dung tin nhắn rỗng hoặc vô nghĩa (spam/ký tự lạ)** | Bỏ qua bước phân tích nghiệp vụ, phân loại vào nhóm `INVALID_INPUT` để nhân viên lọc sau. |
| **Dữ liệu sai lệch/mâu thuẫn (VD: báo chưa nhận hàng nhưng mã đơn ghi đã hoàn tiền)** | Gắn cờ `[CONFLICT_ALERT]` và đẩy thẳng lên mức ưu tiên `P1` để quản lý kiểm tra thủ công. |
| **Đứt gãy ngữ cảnh (Output quá dài bị cắt giữa chừng)** | Áp dụng cơ chế đánh số thứ tự bản ghi (`ID_01`, `ID_02`...) và dừng ở điểm ngắt an toàn; khi nhận lệnh `TIẾP TỤC`, AI phải nối tiếp đúng từ ID kế tiếp mà không lặp lại phần cũ. |

---

## 3. Bước 2: Thiết kế chuỗi Prompt hoàn chỉnh (Prompt Chaining)

### Prompt 1: Làm sạch dữ liệu & Kiểm tra ràng buộc lỗi (Data Extraction & Validation)

```text
[VAI TRÒ]
Bạn là Chuyên viên Xử lý Dữ liệu Vận hành của cửa hàng bán lẻ.

[NHIỆM VỤ]
Đọc dữ liệu tin nhắn thô trong cặp thẻ <raw_messages> dưới đây, trích xuất thông tin từng khách hàng và kiểm tra tính hợp lệ của dữ liệu.

[QUY TẮC XỬ LÝ LỖI BẮT BUỘC]
1. Nếu tin nhắn rỗng, chỉ có emoji hoặc ký tự spam vô nghĩa: Gắn trạng thái "INVALID_INPUT" và không trích xuất các trường khác.
2. Nếu thiếu Mã đơn hàng hoặc Số điện thoại: Tuyệt đối KHÔNG tự suy đoán, điền chính xác chuỗi "MISSING_DATA".
3. Nếu phát hiện thông tin mâu thuẫn (ví dụ: khách báo chưa nhận hàng nhưng ghi chú hệ thống đã hoàn tất/hoàn tiền): Gắn cờ "[CONFLICT_ALERT]".
4. Đánh số thứ tự từng bản ghi theo định dạng MSG-01, MSG-02... để chống đứt gãy ngữ cảnh. Nếu dữ liệu quá dài, hãy dừng lại ở bản ghi hoàn chỉnh cuối cùng và ghi "[CHỜ LỆNH TIẾP TỤC]".

[ĐỊNH DẠNG ĐẦU RA]
Trình bày kết quả dưới dạng bảng Markdown gồm các cột:
| Mã bản ghi | Tên khách | SĐT | Mã đơn | Nội dung tóm tắt | Trạng thái dữ liệu |

[DỮ LIỆU ĐẦU VÀO]
<raw_messages>
1. Chị Lan (0912345678): Đơn DH102 giao từ hôm qua mà mở hộp ra bị vỡ bình giữ nhiệt rồi em ơi, hỗ trợ đổi gấp cho chị nhé!
2. Minh Tuấn: Cho mình hỏi mẫu áo khoác đen size L còn ở chi nhánh Quận 1 không shop?
3. ...
4. Khách ẩn danh (0988777666): Đơn DH105 báo đã giao thành công và hoàn tiền nhưng thực tế anh chưa nhận được hàng lẫn tiền?!
5. 0909111222: Giao hàng chậm quá đi mất, đặt 4 ngày rồi chưa thấy đâu.
</raw_messages>
