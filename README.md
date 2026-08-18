# BÁO CÁO BÀI TẬP BUỔI 05 (SS05 - BÀI 1)
## Đề tài: Phân tích & Lựa chọn - Thiết kế mô tả Tool (Metadata Description) tối ưu trong Spring AI

---

### 👨‍💻 THÔNG TIN HỌC VIÊN
* **Họ và tên:** Phạm Huỳnh Tiến Đạt
* **Mã học viên:** PTIT-HCM-043
* **Môn học:** AI Integration Action (Spring AI Function Calling & Tool Calling)
* **Bài tập:** SS05_HW01 (Bài 1 - Mức độ Khá)

---

## 📝 KHUNG LÀM BÀI CỦA SINH VIÊN

### 1. Phương án lựa chọn:
👉 **Phương án B (Mô tả chi tiết và tường minh) là phương án tối ưu nhất.**

---

### 2. Phân tích phương án chọn (Phương án B):

Trong kiến trúc **Spring AI Function/Tool Calling**, mô hình LLM không trực tiếp nhìn thấy mã nguồn Java hay logic xử lý bên trong hàm. Thay vào đó, Spring AI tự động trích xuất `@Description` (hoặc `description` trong `@Tool`) cùng cấu trúc tham số để biên dịch thành một bản **JSON Schema (Tool Definition / Function Specification)** gửi kèm trong payload request tới mô hình AI.

**Phương án B là thiết kế tối ưu vượt trội vì đáp ứng trọn vẹn 4 nguyên lý thiết kế Tool Metadata cấp doanh nghiệp:**

1. **Định hình mục đích nghiệp vụ rõ ràng (Semantic Precision & Intent Disambiguation):**
   * Mô tả của `getRoomAvailability` nêu rõ nhiệm vụ là *kiểm tra tình trạng còn phòng trống* và *trả về đơn giá mỗi đêm*.
   * Mô tả của `calculateTotalPrice` phân định rạch ròi nhiệm vụ là *tính toán tổng chi phí lưu trú*.
   * Sự phân tách ranh giới ngữ nghĩa rõ ràng này giúp cơ chế **Tool Selection** của LLM không bao giờ bị nhầm lẫn giữa việc "tra cứu phòng" và "tính tổng tiền".

2. **Chỉ dẫn định dạng dữ liệu và giá trị mẫu (Format Guidance & Value Constraints):**
   * Chỉ dẫn rõ ràng định dạng ngày tháng `yyyy-MM-dd` cho `checkInDate` và `checkOutDate`. Khi người dùng nhập ngôn ngữ tự nhiên (ví dụ: *"đặt phòng ngày mai đến cuối tuần"*), LLM sẽ tự động quy đổi chính xác về chuỗi định dạng ISO chuẩn (vd: `2026-08-19`), giúp tầng Java Backend parse qua `LocalDate.parse()` thành công mà không bao giờ gặp `DateTimeParseException`.
   * Cung cấp giá trị mẫu (`Deluxe`, `Standard`) giúp LLM chuẩn hóa tham số `roomType` thay vì sinh ra các chuỗi tự do.

3. **Xác lập tiền điều kiện và thứ tự gọi hàm (Execution Preconditions & Workflow Sequencing):**
   * Trong mô tả của `calculateTotalPrice` có chỉ dẫn ràng buộc: *"Công cụ này chỉ được gọi sau khi đã xác định được loại phòng (roomType) và tổng số ngày lưu trú thực tế của khách hàng (numberOfDays, phải lớn hơn 0)"*.
   * Đây là kỹ thuật **Prompt-Driven State Machine / Precondition Enforcement**, giúp LLM hiểu được luồng nghiệp vụ (Workflow): Phải gọi `getRoomAvailability` trước để biết phòng còn hay không và xác định số ngày, sau đó mới được gọi `calculateTotalPrice`. Điều này ngăn chặn hoàn toàn việc LLM gọi bừa hàm tính tiền khi chưa có thông tin phòng.

4. **Ràng buộc logic giá trị tham số (Business Rule Guardrail):**
   * Yêu cầu `numberOfDays` phải lớn hơn 0 ngăn chặn các trường hợp tính tiền với số ngày bằng 0 hoặc số âm khi người dùng vô tình nhập ngày trả phòng trước ngày nhận phòng.

---

### 3. Phân tích các phương án loại trừ (Phương án A và Phương án C):

#### ❌ Phân tích Phương án A (Mô tả tối giản):
* **Mô tả hiện tại:**
  * `getRoomAvailability`: `"Check phòng trống khách sạn"`
  * `calculateTotalPrice`: `"Tính toán giá tiền phòng"`
* **Nhược điểm & Rủi ro kỹ thuật:**
  1. **Ảo giác và sai lệch định dạng tham số (Parameter Hallucination & Format Mismatch):** Mô tả không hề hướng dẫn định dạng ngày tháng cho `checkInDate` và `checkOutDate`. LLM có thể truyền `19/08/2026`, `19-08`, hoặc chuỗi chữ `"ngày mai"`. Khi Jackson deserialize hoặc Java parse sang `LocalDate`, hệ thống sẽ sập (`IllegalArgumentException` / `DateTimeParseException`).
  2. **Gặp hiện tượng xung đột công cụ (Tool Ambiguity / Misrouting):** Khi người dùng hỏi *"Tôi muốn đặt phòng Deluxe 3 đêm, giá bao nhiêu và còn phòng không?"*, do cả 2 mô tả đều liên quan đến "phòng" và "tiền phòng", LLM rất dễ gọi sai công cụ (vd: chỉ gọi `calculateTotalPrice` mà bỏ qua việc kiểm tra phòng trống `getRoomAvailability`), hoặc gọi cả hai hàm cùng lúc mà thiếu tham số.
  3. **Không kiểm soát được biên giá trị:** Không có ràng buộc `numberOfDays > 0`, dễ dẫn đến việc LLM truyền `numberOfDays = 0` làm sai lệch kết quả tính toán.

---

#### ❌ Phân tích Phương án C (Mô tả kỹ thuật nội bộ) - Anti-Pattern:
* **Mô tả hiện tại:**
  * `getRoomAvailability`: `"Thực thi phương thức getRoomAvailability từ class BookingService, kết nối tới bảng room_status trong MySQL DB qua JPA để đếm số bản ghi thỏa mãn điều kiện checkin và checkout."`
  * `calculateTotalPrice`: `"Gọi hàm calculateTotalPrice nhận vào String roomType và int numberOfDays, trả về kiểu double đại diện cho tích của đơn giá phòng nhân với số ngày."`
* **Nhược điểm & Rủi ro kỹ thuật:**
  1. **Vi phạm nguyên lý Trừu tượng hóa & Lộ chi tiết kỹ thuật nội bộ (Leaking Internal Implementation Details):** LLM là mô hình ngôn ngữ tương tác ở tầng giao tiếp người dùng và điều phối nghiệp vụ. LLM **hoàn toàn không thể và không cần biết** về `class BookingService`, `bảng room_status trong MySQL DB qua JPA`, hay kiểu dữ liệu `double`.
  2. **Gây nhiễu ngữ cảnh và lãng phí Token (Context Window Pollution & Attention Dilution):** Các từ khóa kỹ thuật (`MySQL`, `JPA`, `class`, `double`) làm loãng sự chú ý (*Attention mechanism*) của LLM vào nội dung thực sự của cuộc trò chuyện, làm giảm độ chính xác khi trích xuất thông tin từ câu nói của khách hàng.
  3. **Hoàn toàn thiếu hướng dẫn định dạng nghiệp vụ cho LLM:** Dù mô tả rất dài nhưng lại không cung cấp định dạng ngày `yyyy-MM-dd`, không có giá trị mẫu `Deluxe, Standard`, và không có tiền điều kiện thực thi. Do đó, Phương án C vẫn mắc trọn vẹn tất cả các lỗi runtime về sai lệch định dạng tham số tương tự như Phương án A.

---

### 📊 BẢNG TỔNG HỢP SO SÁNH 3 PHƯƠNG ÁN

| Tiêu chí đánh giá | Phương án A (Tối giản) | Phương án B (Tường minh - Tối ưu) | Phương án C (Kỹ thuật nội bộ) |
| :--- | :--- | :--- | :--- |
| **Phân biệt mục đích công cụ (Intent)** | ⚠️ Kém, dễ nhầm lẫn | ✅ **Rất rõ ràng, chính xác 100%** | ⚠️ Bị nhiễu bởi từ khóa kỹ thuật |
| **Định dạng tham số (`yyyy-MM-dd`)** | ❌ Không có (Dễ crash Runtime) | ✅ **Có chỉ dẫn định dạng & mẫu** | ❌ Không có (Dễ crash Runtime) |
| **Ràng buộc thứ tự thực thi (Sequence)**| ❌ Không có | ✅ **Rõ ràng (Gọi sau khi tra cứu)** | ❌ Không có |
| **Hiệu quả sử dụng Token** | Tiết kiệm nhưng không dùng được | ✅ **Tối ưu, tập trung vào nghiệp vụ** | ❌ Lãng phí token cho chi tiết thừa |
| **Mức độ sẵn sàng cho Production** | ❌ Rủi ro cao | 🏆 **Đạt chuẩn Enterprise Production** | ❌ Anti-pattern |
