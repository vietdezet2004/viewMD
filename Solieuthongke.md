# TÀI LIỆU CHI TIẾT: CÁC CHỈ SỐ, THUẬT TOÁN TÍNH TOÁN VÀ CƠ CHẾ AI TRONG MODULE BÁO CÁO THỐNG KÊ

> **Hệ thống**: DriverGuard AI — Fleet Safety Management System  
> **Module**: Phân hệ Báo cáo & Phân tích An toàn Điều hành (`/reports`)  
> **Tài liệu tham chiếu mã nguồn**:  
> - Backend API: `src/api/reports.py`  
> - Dịch vụ LLM: `src/services/llm.py`  
> - Schema Pydantic: `src/schemas/reports.py`  
> - Giao diện Web: `web/src/components/dashboard/reports/ExecutiveA4Document.jsx`

---

## 1. TỔNG QUAN KIẾN TRÚC VÀ 2 TẦNG AI CỦA HỆ THỐNG

Phân hệ Báo cáo và Giám sát của DriverGuard AI được vận hành dựa trên sự kết hợp giữa **2 tầng Trí tuệ Nhân tạo**:

```
+-------------------------------------------------------------+
| TẦNG 1: COMPUTER VISION & EDGE AI (Tại Buồng Lái / Thiết Bị) |
| - MediaPipe Face Mesh, YOLO, OpenCV                         |
| - Đo nhịp chớp mắt (EAR), ngáp (MAR), góc quay đầu, điện thoại|
| - Phát hiện: Buồn ngủ (Drowsiness), Mất tập trung...       |
+------------------------------+------------------------------+
                               | Gửi Telemetry & Alerts qua MQTT
                               v
+-------------------------------------------------------------+
| CƠ SỞ DỮ LIỆU TRUNG TÂM (PostgreSQL & MinIO Evidence)       |
| - safety_alert_logs, hitl_cases, vehicle_telemetry_logs...  |
+------------------------------+------------------------------+
                               | ETL & Aggregation Queries
                               v
+-------------------------------------------------------------+
| TẦNG 2: GENERATIVE AI & LLM ADVISOR (Tại Backend FastAPI)   |
| - Model: OpenAI gpt-4o-mini (LangChain Integration)         |
| - Đóng vai trò: Chuyên gia Cố vấn An toàn & Điều hành Đội xe |
| - Tự động suy luận rủi ro sinh học & xuất khuyến nghị quản lý|
+-------------------------------------------------------------+
```

---

## 2. NGUỒN DỮ LIỆU GỐC TRONG DATABASE (LẤY Ở ĐÂU?)

Hệ thống truy vấn trực tiếp từ 5 bảng dữ liệu quan hệ trong PostgreSQL:

| Tên Bảng (ORM Model) | Cột dữ liệu khai thác | Mục đích sử dụng |
| :--- | :--- | :--- |
| **`safety_alert_logs`** | `id`, `timestamp`, `severity`, `violation_type`, `risk_score`, `vehicle_id`, `driver_id`, `precursor_events` | Bảng lõi ghi nhận toàn bộ vi phạm an toàn từ IoT / DMS buồng lái. |
| **`hitl_cases`** | `id`, `alert_log_id`, `status` (`CONFIRMED`, `REJECTED`, `PENDING`), `reviewer_notes` | Nhật ký điều phối viên (con người) thẩm định và can thiệp thực tế. |
| **`vehicle_telemetry_logs`**| `vehicle_id`, `timestamp`, `latitude`, `longitude`, `speed_kph`, `address_label` | Nhật ký định vị GPS, dùng để xác định tuyến đường và điểm đen tai nạn. |
| **`users` & `driver_profiles`** | `id`, `full_name`, `role`, `license_number`, `assigned_vehicle_id` | Định danh tài xế, mã giấy phép lái xe (GPLX) và phân ca. |
| **`vehicles`** | `id`, `license_plate`, `model`, `fleet_group` | Danh sách phương tiện, biển kiểm soát và chủng loại xe. |

---

## 3. CHI TIẾT TỪNG CHỈ SỐ, CÔNG THỨC TOÁN HỌC & THUẬT TOÁN

### 3.1. Nhóm Chỉ số Điều hành Cốt lõi (Key Executive KPIs)

#### ① Tổng số cảnh báo an toàn (`total_alerts`)
- **Nguồn lấy**: Đếm số bản ghi trong bảng `safety_alert_logs` có thời gian nằm trong kỳ lọc:
  $$\text{timestamp} \in [\text{start\_dt}, \text{end\_dt}]$$
- **Ý nghĩa**: Tổng lượng rủi ro an toàn phát sinh từ buồng lái trong chu kỳ.

#### ② Cảnh báo Nghiêm trọng (CRITICAL) & Tỷ lệ CRITICAL (`crit_rate`)
- **Nguồn lấy**: Đếm các sự kiện có mức độ `severity = SeverityLevel.CRITICAL` (ví dụ: ngủ gật sâu microsleep, ngất/gục đầu, buông cả hai tay).
- **Công thức tính tỷ lệ**:
  $$\text{crit\_rate} = \begin{cases} \left( \dfrac{\text{crit\_count}}{\text{total\_alerts}} \times 100 \right) \% & \text{nếu } \text{total\_alerts} > 0 \\ 0.0\% & \text{nếu } \text{total\_alerts} = 0 \end{cases}$$

#### ③ Vi phạm Buồn ngủ & Tỷ lệ Buồn ngủ (`drowsy_rate`)
- **Nguồn lấy**: Đếm các vi phạm có `violation_type` thuộc tập:
  $$\{\text{"MICROSLEEP"}, \text{"YAWN"}, \text{"DROWSINESS"}, \text{"SAFETY\_DROWSINESS\_ESCALATION\_LV3"}\}$$
- **Công thức tính tỷ lệ**:
  $$\text{drowsy\_rate} = \frac{\text{drowsy\_count}}{\text{total\_alerts}} \times 100\%$$

#### ④ Điểm An toàn Đội xe (`fleet_safety_score`) & Xếp hạng
- **Nguồn lấy**: Cột `risk_score` (thang từ $0.0$ đến $100.0$) trong bảng `safety_alert_logs`.
- **Thuật toán tính toán**:
  1. Tính điểm rủi ro bình quân toàn đội:
     $$\text{avg\_risk} = \frac{\sum_{i=1}^{N} \text{risk\_score}_i}{N} \quad (N = \text{số cảnh báo có điểm rủi ro})$$
  2. Quy đổi sang Điểm An toàn (Safety Score):
     $$\text{fleet\_safety\_score} = \max(100.0 - \text{avg\_risk}, 0.0)$$
  3. Phân hạng an toàn chuẩn hóa:
     - $\ge 90.0$: **Hạng A (Rất tốt)** — Đội xe kiểm soát an toàn tối ưu.
     - $\ge 75.0$: **Hạng B+ (Khá)** — Đội xe an toàn, có một số vi phạm cảnh báo sớm.
     - $\ge 60.0$: **Hạng C (Trung bình)** — Mức độ rủi ro cần chú ý, xuất hiện vi phạm nghiêm trọng.
     - $< 60.0$: **Hạng D (Báo động / Nguy cơ cao)** — Yêu cầu can thiệp điều hành khẩn cấp.

#### ⑤ Tỷ lệ Xác thực Thẩm định HITL Dispatcher (`hitl_rate`)
- **Nguồn lấy**: Bảng `hitl_cases` kết nối với các cảnh báo an toàn trong kỳ.
- **Ý nghĩa**: Đo lường hiệu quả quy trình Human-In-The-Loop. Tỷ lệ cảnh báo được điều phối viên xác nhận là đúng thực tế (không phải báo động giả).
- **Công thức**:
  $$\text{hitl\_rate} = \frac{\text{Số ca có status = CONFIRMED}}{\text{Tổng số ca HITL}} \times 100\%$$

---

### 3.2. Phân tích Khung giờ Sinh học & Giờ Vàng Nguy Hiểm (Circadian Rhythm)

Hệ thống tự động quy đổi toàn bộ mốc thời gian UTC sang **Múi giờ Việt Nam (UTC+7)**:
$$\text{local\_hour} = (\text{timestamp.hour} + 7) \pmod{24}$$

#### ① Khung giờ đêm khuya (Night Biological Lull)
- **Tập giờ phân tích**: Giờ đêm $\in \{0, 1, 2, 3, 4, 5\}$.
- **Cửa sổ đêm rủi ro cao (`night_window_cnt`)**:
  $$\text{night\_window\_cnt} = \sum_{h \in \{1, 2, 3, 4\}} \text{Alerts}(h)$$
  *(Khung giờ từ 01:00 đến 04:00 là thời điểm phản xạ lái xe suy giảm sâu nhất theo nhịp sinh học tự nhiên)*.
- **Đỉnh đêm (`peak_night_h`, `peak_night_cnt`)**: Khung giờ đêm có số vụ vi phạm cao nhất ($\text{argmax}$).

#### ② Khung giờ trũng sinh học sau bữa trưa (Post-Lunch Dip)
- **Tập giờ ban ngày**: Giờ ngày $\in [6, 23]$.
- **Số vụ vi phạm sau bữa trưa (`post_lunch_cnt`)**:
  $$\text{post\_lunch\_cnt} = \text{Alerts}(13\text{h}) + \text{Alerts}(14\text{h}) + \text{Alerts}(15\text{h})$$
  *(Hiện tượng cơ thể tiêu hóa bữa ăn trưa gây buồn ngủ tạm thời kết hợp với nhiệt độ mặt đường cao)*.
- **Đỉnh ngày (`peak_day_h`, `peak_day_cnt`)**: Khung giờ ban ngày có số vụ vi phạm cao nhất.

---

### 3.3. Nhận diện Tuyến đường & Điểm đen Rủi ro (Geographical Blackspots)

Thuật toán thực hiện liên kết thời không gian (Spatial-Temporal Correlation):
1. **Truy vấn**: Lấy danh sách tọa độ GPS và tên địa danh (`address_label`) từ bảng `vehicle_telemetry_logs`.
2. **Khớp cặp (Matching)**: Với mỗi cảnh báo trong `safety_alert_logs`, thuật toán tìm điểm viễn thông của chính phương tiện đó có độ lệch thời gian nhỏ nhất:
   $$\Delta t = \min |\text{timestamp}_{\text{telemetry}} - \text{timestamp}_{\text{alert}}|$$
3. **Thống kê tần suất**: Đếm số cảnh báo theo từng `address_label` (ví dụ: *Quận Long Biên, Tuyến Vành đai 3, Cầu Thanh Trì, Cầu Giấy...*).
4. **Đầu ra**: Trích xuất Top 4 tuyến đường/địa bàn có mật độ vi phạm cao nhất kèm số vụ thực tế để đưa vào báo cáo.

---

### 3.4. Bảng điểm Tài xế & Danh sách Vi phạm Lặp lại (Repeat Offenders)

Hệ thống gom nhóm toàn bộ cảnh báo theo từng `driver_id`:

#### ① Các chỉ số tính toán trên từng tài xế:
- **Số vụ vi phạm (`violation_count`)**: Tổng số lần vi phạm của tài xế trong kỳ.
- **Lỗi chủ đạo (`primary_violation_type`)**: Loại lỗi chiếm tần suất cao nhất (Mode) của tài xế đó:
  - *Ngủ gật (Microsleep)*: Phát hiện mắt nhắm $> 1.5\text{s}$.
  - *Ngáp lặp lại (Yawn)*: Miệng mở rộng vượt ngưỡng nhiều lần trong thời gian ngắn.
  - *Mất tập trung (Distraction)*: Đầu quay sang hướng khác hoặc dùng điện thoại.
- **Điểm an toàn cá nhân (`safety_score`)**:
  $$\text{driver\_score} = \max \left(100.0 - \frac{\sum \text{risk\_score}_{\text{của tài xế}}}{\text{Số vụ}}, 0.0 \right)$$

#### ② Thuật toán khuyến nghị chế tài tự động (Rule-Based Matrix):
- **Trường hợp 1**: Nếu $\text{violation\_count} \ge 50$ HOẶC $\text{safety\_score} < 70$:
  $$\rightarrow \text{"Tạm đình chỉ ca chạy 48h & Đào tạo lại an toàn bắt buộc"}$$
- **Trường hợp 2**: Nếu $\text{violation\_count} \ge 20$:
  $$\rightarrow \text{"Cảnh cáo văn bản & Trừ điểm thưởng chuyên cần tháng"}$$
- **Trường hợp 3**: Nếu $\text{violation\_count} < 20$:
  $$\rightarrow \text{"Huấn luyện lại kỹ năng phòng chống buồn ngủ & Giám sát 1:1"}$$

---

### 3.5. Thống kê Tiền đề Rủi ro (Precursor Events)

- **Nơi lấy**: Cột định dạng JSON `precursor_events` trong từng bản ghi `safety_alert_logs`.
- **Ý nghĩa khoa học**: Các vi hành vi xuất hiện trước $30\text{s} - 60\text{s}$ khi xảy ra một sự cố nguy hiểm (ví dụ: ngáp ngắn 2 lần liên tục, tần suất nháy mắt gia tăng, chuyển động giật đầu nhẹ).
- **Thuật toán**: Đếm tỷ lệ cảnh báo có tiền đề (`alerts_with_precursors`) và xếp hạng loại tiền đề phổ biến nhất nhằm hỗ trợ đào tạo nhận thức sớm cho lái xe.

---

## 4. CƠ CHẾ HOẠT ĐỘNG CỦA GENERATIVE AI (OPENAI LLM)

### 4.1. Cách thức hoạt động
Khi tham số `use_ai = true` được gửi lên từ Frontend:
1. Backend nạp toàn bộ số liệu thống kê thực tế ở các phần trên vào Prompt có cấu trúc.
2. Thiết lập vai trò chuyên gia cho LLM:
   > *"Bạn là Chuyên gia Cao cấp về An toàn Giao thông và Quản trị Đội xe Vận tải Thông minh."*
3. Yêu cầu LLM trả về định dạng **Pure JSON Schema**:
   ```json
   {
     "night_analysis": "Phân tích rủi ro khung giờ đêm khuya và nhịp sinh học (1-2 câu)",
     "day_analysis": "Phân tích rủi ro ban ngày và hiệu ứng hạ đường huyết sau bữa trưa (1-2 câu)",
     "recommendations": [
       "Biện pháp can thiệp 1 (định lượng cụ thể theo số liệu thực)",
       "Biện pháp can thiệp 2",
       "Biện pháp can thiệp 3",
       "Biện pháp can thiệp 4"
     ]
   }
   ```
4. Backend parse JSON và đưa vào **Mục V (Đánh giá Khung giờ)** và **Mục VI (Khuyến nghị Điều hành)** trong Báo cáo A4.

### 4.2. Cơ chế Fallback an toàn (Đảm bảo 100% không văng lỗi)
- Toàn bộ khối gọi OpenAI được bao bọc trong khối `try...except Exception`.
- Nếu chưa có `OPENAI_API_KEY`, API key hết quota, hoặc mạng timeout: Hệ thống tự động kích hoạt **Bộ quy tắc tính toán dự phòng nội bộ (Deterministic Rule Engine)**.
- Báo cáo vẫn được sinh ra đầy đủ số liệu chính xác mà người dùng không gặp bất kỳ lỗi gián đoạn nào.

---

## 5. HƯỚNG DẪN CẤU HÌNH MÔI TRƯỜNG & BẢO MẬT API KEY KHI DEPLOY

### 5.1. Bảo mật trên GitHub
- File `.env` và các file chứng chỉ SSH (`*.pem`) đã được đưa vào `.gitignore` (dòng 16).
- **Tuyệt đối không** commit hoặc push file `.env` lên GitHub repository.

### 5.2. Cấu hình trên Server đã Deploy (Docker Compose / EC2)
1. **Kết nối SSH vào server**:
   ```bash
   ssh -i AWS-KeyPair.pem ubuntu@<IP_SERVER>
   ```
2. **Mở file `.env` trên thư mục dự án**:
   ```bash
   cd ~/P-136
   nano .env
   ```
3. **Thêm hoặc sửa API Key**:
   ```env
   OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxxxxxxxxxxxxx
   ```
   *(Nhấn `Ctrl + O` $\rightarrow$ `Enter` để lưu, `Ctrl + X` để thoát)*.
4. **Khởi động lại backend để nạp cấu hình mới**:
   ```bash
   docker compose restart backend
   ```
5. **Kiểm tra log vận hành**:
   ```bash
   docker compose logs -f backend --tail 50
   ```
6. **Bảo toàn khi CI/CD cập nhật**: Quy trình CI (`.github/workflows/ci.yml`) đã được thiết lập để tự động đọc file `.env` trên máy chủ chủ quyền (`$HOME/P-136/.env`) nên các lần push code mới sẽ **không bao giờ** làm mất hay đè lên API key của bạn.
