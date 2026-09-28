# BẢN BRIEF TRÌNH BÀY & BỘ CÂU HỎI XIN ĐỊNH HƯỚNG MENTOR
## Dự Án: DriverGuard — Hệ Thống Giám Sát An Toàn & Vận Hành Đội Xe

| Thuộc tính | Nội dung |
| :--- | :--- |
| **Nhóm thực hiện** | Nhóm AI Thực Chiến (Phát hiện tài xế mất tập trung — Đội xe Xanh SM) |
| **Thời lượng trình bày** | 3 – 5 phút (Cô đọng, tập trung vào kết quả & giải pháp) |
| **Mục đích** | Báo cáo tiến độ chuẩn hóa kiến trúc, giải quyết toàn bộ bài toán của Duty 01-03 và xin định hướng chốt phạm vi MVP tuần tới |

---

# PHẦN 1: BẢN BRIEF TRÌNH BÀY 5 PHÚT (PITCH BRIEF)
*(Cấu trúc chuẩn theo 5 yêu cầu cốt lõi Mentor đã giao ở Duty 02 & Duty 03)*

---

### 1. Bối cảnh Doanh nghiệp & Bài toán (Biz Context & Pain Point)
* **Bối cảnh:** Hãng taxi điện (như Xanh SM) vận hành quy mô lớn (~100.000 xe), tài xế thường chạy ca kéo dài 8–10 tiếng hoặc chạy ca đêm/thời tiết xấu.
* **Nỗi đau thực tế (Pain Points):**
  1. *Phía Tài xế:* Mệt mỏi sinh học tích tụ dẫn đến vi ngủ (micro-sleep 2–3 giây). Tự nhận thức bản thân thường đến quá muộn.
  2. *Phía Vận hành (Fleet Ops):* Dữ liệu camera nếu stream liên tục sẽ làm sập hạ tầng và tốn hàng tỷ đồng tiền mạng; nếu chỉ thu thập ảnh mà không phân tích nhân quả với doanh thu và cuốc chạy thì dữ liệu trở nên vô dụng.

---

### 2. Mô hình Lợi ích Kinh tế (Value Proposition & ROI)
Nhóm không chỉ tiếp cận dưới góc độ "thuật toán nhận diện khuôn mặt", mà chuyển đổi thành giá trị kinh tế:
1. **Giảm thiểu thiệt hại trực tiếp:** Giảm rủi ro tai nạn nghiêm trọng (chi phí sửa chữa xe điện, đền bù bảo hiểm, gián đoạn kinh doanh phương tiện).
2. **Bảo vệ chỉ số hài lòng (CSAT & Rating):** Phân tích cho thấy tài xế tỉnh táo có rating sao cao hơn, hạn chế tối đa khiếu nại của khách hàng đi taxi.
3. **Mục tiêu tương lai (North Star Hypothesis):** Tối ưu hóa **Doanh thu trên mỗi giờ xe online (Revenue per Online Vehicle Hour)** thông qua việc cảnh báo kịp thời và điều chuyển ca an toàn.

---

### 3. Giải pháp Kỹ thuật & Đáp ứng Ràng buộc của Mentor (Technical Architecture)
Nhóm đã tiếp thu toàn bộ góp ý kỹ thuật từ 3 buổi Mentor Duty trước để chốt kiến trúc:

| Vấn đề Mentor đã chỉ ra | Giải pháp nhóm đã chốt & hiện thực hóa |
| :--- | :--- |
| **Tải 100.000 xe đồng thời (Duty 02)** | **Bỏ hoàn toàn WebSocket liên tục.** Chuyển sang kiến trúc **Event-Driven JSON siêu nhẹ** (chỉ gửi khi có vi phạm) đệm qua Message Broker (Kafka/MQTT). |
| **Ngưỡng nhận diện đơn lẻ gây báo sai (Duty 03)** | Chuyển sang **Thuật toán Ngưỡng tổng hợp cửa sổ trượt 3s (Sliding-window Risk Score)**: Kết hợp PERCLOS + Góc gục đầu + Bù trừ vận tốc GPS để lọc xe dừng đèn đỏ, ngáp ngắn, nói chuyện. |
| **Bảo vệ riêng tư & Chi phí lưu trữ (Duty 03)** | Áp dụng chính sách **Retention 24h đối với video clip ngắn 5s** (tự xóa qua S3 Lifecycle); dữ liệu lưu dài hạn 3–5 năm là **Audit Log dạng chữ** phục vụ kiểm toán. |
| **Độ trễ cứu mạng trên xe (Duty 01)** | Mô hình AI MediaPipe FaceMesh chạy **100% on-device trên điện thoại**, đảm bảo còi hú cứu mạng trong $\le 300\text{ ms}$ ngay cả khi mất sóng 4G trên cao tốc. |
| **Tránh phạt oan tài xế** | Thiết kế quy trình **Human-in-the-Loop (HITL)** trên Web Dashboard kèm nút **Khiếu nại 1-chạm** trên app tài xế để đối soát chéo viễn thông. |

---

### 4. Phạm vi MVP Tuần Tới (Core MVP vs. Non-core Roadmap)
Để đảm bảo tiến độ "1 tuần nữa phải có MVP chạy được" theo cảnh báo của Mentor, nhóm phân định dứt khoát:
* **TRỌNG TÂM MVP (Làm thật & Đo thật):**
  1. *Mobile Edge App:* Nhận diện khuôn mặt $\ge 15\text{ FPS}$, phát còi báo on-device $\le 300\text{ ms}$, cắt buffer clip 5s.
  2. *Data Contract:* Chuẩn hóa payload JSON sự kiện giữa Mobile và Backend.
  3. *Manager Dashboard:* Màn hình xem bản đồ đội xe, popup video 5s để Manager bấm Xác nhận / Bác bỏ (HITL).
* **ROADMAP TƯƠNG LAI (Non-core):**
  * Module *Smart Fleet Dispatch* (dự báo nhu cầu khách để điều xe) được tách riêng thành phase mở rộng sau khi Safety Core chạy ổn định.

---

### 5. Phân công Nhóm & Cam kết Deliverables
* **Phan Hoàng Vũ:** Phụ trách On-Device AI Model (MediaPipe/TFLite, tối ưu FPS & Latency trên Android).
* **Nguyễn Công Duẩn:** Phụ trách Backend API, Data Contract & Xử lý hàng đợi sự kiện.
* **Đỗ Thành Đạt:** Phụ trách Web Dashboard cho Fleet Manager (Giao diện bản đồ & luồng duyệt HITL clip 5s).
* **Phùng Quốc Việt:** Phụ trách Kiến trúc hệ thống tổng thể, Benchmark kịch bản thực tế & Báo cáo sản phẩm.

---

# PHẦN 2: BỘ CÂU HỎI CHIẾN LƯỢC XIN ĐỊNH HƯỚNG TỪ MENTOR
*(Dành cho phần Q&A cuối buổi — Hỏi đúng tầm để Mentor thấy nhóm có chiều sâu tư duy)*

---

### ❓ Nhóm 1: Về Ranh giới Phạm vi MVP (Scope & Prioritization)
> **Câu hỏi 1:** *"Thưa Mentor, ở buổi Duty 02, Mentor có gợi mở bài toán kết hợp dữ liệu mệt mỏi với doanh thu để tối ưu điều phối xe. Để đảm bảo tiến độ thứ 7 tuần tới có sản phẩm chạy thông luồng, nhóm em đã tách bài toán thành 2 giai đoạn: **Giai đoạn 1 (Tuần này) tập trung 100% vào Safety Core (Cảnh báo on-device + Manager duyệt HITL)**, còn **Giai đoạn 2 mới demo mô phỏng (Simulation) bài toán điều phối doanh thu**. Mentor thấy việc thu hẹp phạm vi như vậy đã hợp lý và an toàn cho tiến độ MVP chưa ạ?"*
* **Mục đích hỏi:** Thể hiện nhóm biết quản trị rủi ro tiến độ, không tham lam ôm đồm, xin Mentor "bật đèn xanh" cho việc tập trung làm chắc phần an toàn.

---

### ❓ Nhóm 2: Về Nguồn Dữ liệu & Môi trường Thử nghiệm (Data & Simulation)
> **Câu hỏi 2:** *"Vì nhóm không có quyền truy cập trực tiếp vào hệ thống dữ liệu cuốc xe và doanh thu thời gian thực của Xanh SM, cho buổi demo MVP tới, Mentor kỳ vọng nhóm chứng minh giá trị phân tích nhân quả bằng cách nào:*
> * Phương án A: Dựng một bộ kịch bản dữ liệu mô phỏng (Synthetic/Replay Data) dựa trên bản đồ taxi Hà Nội?
> * Hay Phương án B: Chỉ cần tập trung đo kiểm số liệu nhận diện thực tế (Precision, Recall, Latency) trên thiết bị thật với tài xế thật?"*
* **Mục đích hỏi:** Làm rõ tiêu chí chấm điểm của Mentor, tránh trường hợp nhóm bỏ công làm mô phỏng nhưng Mentor lại chỉ muốn xem benchmark AI thực tế trên điện thoại.

---

### ❓ Nhóm 3: Về Trải nghiệm Vận hành Thực tế (Operational UX & Edge Cases)
> **Câu hỏi 3:** *"Trong thiết kế luồng duyệt HITL của Manager, nhóm đang áp dụng quy tắc: **Nếu tài xế vi phạm trên cao tốc hoặc không phản hồi chuông báo quá 5 giây thì hệ thống mới kích hoạt báo động đỏ khẩn cấp và đẩy clip 5s về cho Manager**. Theo kinh nghiệm vận hành thực tế của Mentor tại các doanh nghiệp vận tải, tần suất đẩy cảnh báo về điều phối viên như vậy đã đủ chặt chẽ để tránh gây quá tải thông báo (Alert Fatigue) cho người quản lý chưa ạ?"*
* **Mục đích hỏi:** Kéo Mentor vào góc nhìn chuyên gia vận hành thực tế, chứng minh nhóm đã suy nghĩ rất sâu về bài toán 100.000 xe và vấn đề ngợp thông báo.

---

### ❓ Nhóm 4: Về Tiêu chí Nghiệm thu MVP Tuần Tới (Pass/Fail Criteria)
> **Câu hỏi 4:** *"Để chuẩn bị tốt nhất cho buổi check-in thứ 4 và buổi nghiệm thu MVP thứ 7 tuần tới, Mentor có thể cho nhóm xin **3 tiêu chí quan trọng nhất mà Mentor sẽ dùng để đánh giá đạt/chưa đạt** của một luồng hệ thống thông suốt không ạ?"*
* **Mục đích hỏi:** Nắm chắc "đáp án trong đề thi" của Mentor để dồn toàn lực hoàn thiện đúng 3 điểm mấu chốt đó trong tuần này.
