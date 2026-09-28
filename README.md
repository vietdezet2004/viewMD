# SOFTWARE ARCHITECTURE DOCUMENT (SAD) — HỆ THỐNG DRIVERGUARD
## Tài Liệu Thiết Kế Kiến Trúc Phần Mềm Toàn Diện (Mô Hình C4 Chuẩn Hóa)

| Thuộc tính | Nội dung |
| :--- | :--- |
| **Dự án** | **DriverGuard** — Hệ thống Giám sát An toàn & Vận hành Đội xe Thông minh |
| **Quy mô mục tiêu** | 100.000 xe taxi (Điển hình: Đội xe Xanh SM) |
| **Phiên bản kiến trúc** | 2.0 (High-Readability Architecture Design) |
| **Chuẩn thiết kế** | C4 Model (Context $\rightarrow$ Container $\rightarrow$ Component $\rightarrow$ Code) |
| **Tài liệu tham chiếu** | PRD v1.0, Báo cáo đánh giá MVP, Biên bản Mentor Duty 01, 02, 03 |

---

## 📌 BẢNG TRA CỨU NHANH 4 CẤP ĐỘ C4

```mermaid
flowchart LR
    C1["🏛️ MỨC C1: CONTEXT<br><b>Bối cảnh Hệ thống</b><br>Hệ thống nằm ở đâu? Ai dùng?"] --> C2["📦 MỨC C2: CONTAINER<br><b>Các Khối Ứng Dụng</b><br>App, Backend, DB, Cloud S3"]
    C2 --> C3["🧩 MỨC C3: COMPONENT<br><b>Module Chi Tiết</b><br>Bên trong Edge App & Backend"]
    C3 --> C4["💻 MỨC C4: CODE<br><b>Class & Thuật Toán</b><br>RiskScorer, HITL, Contract"]

    style C1 fill:#1e293b,stroke:#38bdf8,stroke-width:2px,color:#f8fafc
    style C2 fill:#1e293b,stroke:#818cf8,stroke-width:2px,color:#f8fafc
    style C3 fill:#1e293b,stroke:#f472b6,stroke-width:2px,color:#f8fafc
    style C4 fill:#1e293b,stroke:#34d399,stroke-width:2px,color:#f8fafc
```

---

# 🏛️ MỨC C1: SYSTEM CONTEXT (BỐI CẢNH HỆ THỐNG)

> **Mục tiêu:** Định vị toàn bộ hệ thống DriverGuard trong môi trường thực tế; xác định rõ tương tác với **Tài xế**, **Fleet Manager** và các **Hệ thống doanh nghiệp bên ngoài**.

### 1. Sơ đồ Kiến trúc Bối cảnh (C1 Diagram)

```mermaid
flowchart TB
    %% ACTORS
    subgraph USERS ["👥 TÁC NHÂN NGƯỜI DÙNG"]
        Driver["👨‍✈️ <b>Tài xế Taxi (Driver)</b><br>• Lái xe trên cabin<br>• Nhận cảnh báo tức thì<br>• Khiếu nại khi bị nhận diện sai"]
        Manager["👨‍💼 <b>Fleet Manager (Điều phối viên)</b><br>• Giám sát an toàn đội xe<br>• Duyệt vi phạm clip 5s (HITL)<br>• Ra lệnh can thiệp / Dừng xe"]
    end

    %% CORE SYSTEM
    subgraph DRIVERGUARD_CORE ["🛡️ HỆ THỐNG TRỌNG TÂM: DRIVERGUARD PLATFORM"]
        DriverGuard["<b>DRIVERGUARD SYSTEM</b><br>━━━━━━━━━━━━━━━━━━━━━━<br>• Phân tích thị giác AI on-device (<=300ms)<br>• Đánh giá mức rủi ro tổng hợp (Risk Score)<br>• Cảnh báo phân cấp (Âm thanh/Giọng nói)<br>• Điều phối luồng duyệt vi phạm (HITL)<br>• Phân tích dữ liệu & Tối ưu vận hành"]
    end

    %% EXTERNAL SYSTEMS
    subgraph EXT_SYSTEMS ["🏢 HỆ THỐNG NGOÀI (EXTERNAL SYSTEMS)"]
        FMS["🚗 <b>Hệ Thống Vận Hành Hãng (Xanh SM FMS)</b><br>• Quản lý thông tin tài xế & cuốc xe<br>• Đồng bộ trạng thái khóa/mở nhận cuốc"]
        S3Cloud["☁️ <b>Cloud Storage (AWS S3 / MinIO)</b><br>• Lưu trữ tạm clip 5s vi phạm<br>• Tự động xóa sau 24h-48h"]
        MapsAPI["🗺️ <b>Bản Đồ & Trạm Dừng Nghỉ</b><br>• Vị trí trạm dừng nghỉ cao tốc<br>• Cung đường & tình hình kẹt xe"]
    end

    %% RELATIONSHIPS
    Driver -->|"1. Camera soi khuôn mặt (15-20 FPS)<br>2. Nhận còi hú/giọng nói cảnh báo (<=300ms)<br>3. Bấm phản hồi / Khiếu nại"| DriverGuard
    Manager -->|"1. Xem bản đồ rủi ro realtime<br>2. Xem clip 5s duyệt vi phạm (Xác nhận/Bác bỏ)<br>3. Gửi lệnh dừng xe / Khóa cuốc"| DriverGuard

    DriverGuard -->|"Đồng bộ mã tài xế, dữ liệu cuốc, lệnh khóa nhận cuốc"| FMS
    DriverGuard -->|"Upload clip 5s vi phạm (Pre-signed URL, TTL 24h)"| S3Cloud
    DriverGuard -->|"Truy vấn trạm dừng nghỉ gần nhất trên cao tốc"| MapsAPI

    %% STYLING
    classDef userStyle fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc;
    classDef coreStyle fill:#1e1b4b,stroke:#a855f7,stroke-width:3px,color:#f8fafc;
    classDef extStyle fill:#1e293b,stroke:#94a3b8,stroke-width:2px,color:#e2e8f0;

    class Driver,Manager userStyle;
    class DriverGuard coreStyle;
    class FMS,S3Cloud,MapsAPI extStyle;
```

### 2. Mô tả Trách nhiệm Tác nhân & Giao tiếp trong C1

| Đối tượng / Tác nhân | Vai trò trong hệ thống | Giao thức & Dữ liệu trao đổi |
| :--- | :--- | :--- |
| **Tài xế Taxi** | Người trực tiếp điều khiển phương tiện, nhận cảnh báo cứu mạng tức thời khi mất tập trung/buồn ngủ. | • **Input:** Khuôn mặt trước camera điện thoại.<br>• **Output:** Âm thanh còi hú, rung, khẩu lệnh giọng nói.<br>• **Phản hồi:** Chạm màn hình, hô *"Tôi tỉnh"* hoặc gửi khiếu nại. |
| **Fleet Manager** | Giám sát viên an toàn, giữ vai trò Human-in-the-Loop để bảo đảm AI không đánh giá oan cho tài xế. | • **Giao diện:** Web Dashboard (HTTPS / WSS).<br>• **Thao tác:** Xem clip 5s $\rightarrow$ Bấm Xác nhận / Bác bỏ $\rightarrow$ Gửi lệnh can thiệp. |
| **Hãng Taxi (Xanh SM FMS)** | Cung cấp thông tin nghiệp vụ và nhận chỉ thị điều phối từ DriverGuard. | • **REST API / Webhook:** Đồng bộ ID tài xế, biển số xe, trạng thái cuốc (`CHỞ KHÁCH` / `RẢNH`), thực thi lệnh khóa cuốc tạm thời. |
| **Cloud Object Storage (S3)** | Kho lưu trữ video bằng chứng vi phạm dung lượng tối ưu. | • **HTTPS S3 API:** Upload clip 5 giây khi có sự cố nghiêm trọng; tự hủy sau 24h theo chính sách S3 Lifecycle. |
| **Map & POI Service** | Bản đồ số hỗ trợ điều hướng an toàn. | • **REST API:** Định vị nút giao cao tốc, trạm xăng, trạm dừng nghỉ gần nhất khi tài xế kiệt sức. |

---

# 📦 MỨC C2: CONTAINER DIAGRAM (CÁC KHỐI ỨNG DỤNG)

> **Mục tiêu:** Phân rã hệ thống thành các khối phần mềm (Container) độc lập về mặt triển khai và công nghệ. Giải quyết bài toán chịu tải **100.000 xe** mà không làm nghẽn mạng hay tràn chi phí server.

### 1. Sơ đồ Kiến trúc Container (C2 Diagram)

```mermaid
flowchart TB
    subgraph EDGE_LAYER ["🚗 PHÍA TRÊN PHƯƠNG TIỆN (VEHICLE EDGE CONTAINER)"]
        MobileApp["📱 <b>DriverGuard Mobile App</b><br><i>(Android Kotlin / iOS Swift + MediaPipe C++)</i><br>━━━━━━━━━━━━━━━━━━━━━━━━━<br>• Camera AI on-device (20 FPS)<br>• Tính điểm Risk Score tức thì<br>• Phát còi hú cảnh báo (<=300ms)<br>• Cắt clip buffer 5s khi có sự cố<br>• SQLite Offline-First Queue"]
    end

    subgraph INGESTION_LAYER ["⚡ TẦNG TIẾP NHẬN & XẾP HÀNG (INGESTION & BROKER)"]
        APIGateway["🚪 <b>API Gateway / Ingestion Service</b><br><i>(Go / Fastify)</i><br>• Tiếp nhận Event JSON qua HTTPS REST<br>• Nhận Heartbeat WSS mỗi 30s<br>• Rate Limiting & Auth Token"]
        MsgBroker["📨 <b>Event Stream Broker</b><br><i>(Apache Kafka / RabbitMQ)</i><br>• Topic: `critical-safety-events`<br>• Topic: `telemetry-stream`<br>• Đảm bảo không mất dữ liệu 100k xe"]
    end

    subgraph BACKEND_SERVICES ["🧠 TẦNG NGHIỆP VỤ & PHÂN TÍCH (BACKEND SERVICES)"]
        SafetyHITLSvc["🛡️ <b>Safety Core & HITL Service</b><br><i>(Node.js / Python)</i><br>• Quản lý hàng đợi duyệt vi phạm<br>• Xử lý logic can thiệp khẩn cấp<br>• Tính điểm an toàn tài xế (Safety Score)"]
        ReportEngine["📊 <b>BI & Causal Analytics Engine</b><br><i>(Python / FastAPI)</i><br>• Báo cáo tương quan: Giờ lái ↔ Buồn ngủ<br>• Tương quan: Điểm an toàn ↔ Rating/Doanh thu<br>• Phát hiện khu vực rủi ro cao"]
        RetentionWorker["🧹 <b>Data Retention & Janitor Worker</b><br><i>(Go Cron Job)</i><br>• Quét & Xóa clip S3 hết hạn 24h<br>• Nén Cold Storage log sau 90 ngày"]
    end

    subgraph DATA_STORES ["💾 TẦNG LƯU TRỮ DỮ LIỆU (PERSISTENCE)"]
        PostgresDB[("🗄️ <b>Operational & Audit DB</b><br><i>(PostgreSQL)</i><br>• Tài xế, Xe, Cuốc xe<br>• <b>Audit Log duyệt (3-5 năm)</b><br>• Lịch sử khiếu nại")]
        ClickHouseDB[("📈 <b>Time-Series Telemetry DB</b><br><i>(ClickHouse / TimescaleDB)</i><br>• Tọa độ GPS, vận tốc theo giây<br>• Dữ liệu EAR/MAR/Risk Score<br>• Báo cáo BI")]
        S3Storage[("☁️ <b>Object Storage</b><br><i>(AWS S3 / MinIO)</i><br>• Video clip 5s vi phạm<br>• <b>TTL: 24h - 48h tự hủy</b>")]
    end

    subgraph WEB_LAYER ["🖥️ PHÍA QUẢN LÝ (WEB CONTAINER)"]
        WebDashboard["💻 <b>Fleet Manager Dashboard</b><br><i>(React / Next.js + TailwindCSS)</i><br>• Bản đồ giám sát 100.000 xe realtime<br>• Popup video clip 5s duyệt HITL<br>• Nút gọi điện khẩn cấp / Dừng xe"]
    end

    %% FLOWS
    MobileApp -->|"1. Gửi Event Metadata JSON (REST/HTTPS)"| APIGateway
    MobileApp -.->|"2. Gửi Heartbeat định kỳ 30s (WSS)"| APIGateway
    MobileApp -->|"3. Upload clip 5s khi sự cố Cấp 3 (HTTPS)"| S3Storage
    MobileApp -->|"4. Mất mạng: Tự lưu vào SQLite; Có mạng: Tự sync"| MobileApp

    APIGateway -->|"Đẩy event thô vào Kafka"| MsgBroker
    MsgBroker -->|"Tiêu thụ sự kiện an toàn"| SafetyHITLSvc
    MsgBroker -->|"Đẩy telemetry vào Data Warehouse"| ClickHouseDB

    SafetyHITLSvc -->|"Ghi log vi phạm & Audit"| PostgresDB
    SafetyHITLSvc -->|"Push cảnh báo đỏ realtime (WebSocket/SSE)"| WebDashboard
    WebDashboard -->|"Manager duyệt: Xác nhận / Bác bỏ (REST)"| SafetyHITLSvc

    ReportEngine -->|"Truy vấn phân tích dữ liệu lớn"| ClickHouseDB
    ReportEngine -->|"Cung cấp biểu đồ vận hành"| WebDashboard

    RetentionWorker -->|"Kích hoạt xóa clip hết hạn 24h"| S3Storage
    RetentionWorker -->|"Nén dữ liệu cũ"| PostgresDB

    %% STYLING
    classDef edgeStyle fill:#064e3b,stroke:#34d399,stroke-width:2px,color:#ecfdf5;
    classDef ingestStyle fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff;
    classDef svcStyle fill:#312e81,stroke:#a78bfa,stroke-width:2px,color:#f5f3ff;
    classDef dbStyle fill:#1f2937,stroke:#9ca3af,stroke-width:2px,color:#f9fafb;
    classDef webStyle fill:#701a75,stroke:#f472b6,stroke-width:2px,color:#fdf2f8;

    class MobileApp edgeStyle;
    class APIGateway,MsgBroker ingestStyle;
    class SafetyHITLSvc,ReportEngine,RetentionWorker svcStyle;
    class PostgresDB,ClickHouseDB,S3Storage dbStyle;
    class WebDashboard webStyle;
```

### 2. Bảng Phân Tích Công Nghệ & Trọng Trách Container (C2)

| Container | Công nghệ đề xuất | Trách nhiệm chính | Giải quyết bài toán gì của Mentor? |
| :--- | :--- | :--- | :--- |
| **Mobile Safety App** | Kotlin/Swift, MediaPipe Tasks, SQLite | Chạy AI on-device 20 FPS, còi báo $\le 300\text{ ms}$, cắt clip buffer 5s, lưu hàng đợi offline. | **Không phụ thuộc 4G/Internet**, xử lý cứu mạng tức thời, không lo mất sóng cao tốc. |
| **API Gateway** | Golang / Fastify | Nhận kết nối REST/WebSocket, định tuyến, kiểm tra tính hợp lệ của token thiết bị. | Đảm bảo throughput cao, tiêu tốn ít RAM/CPU khi tiếp nhận kết nối từ hàng chục ngàn xe. |
| **Message Broker** | Apache Kafka | Đệm và xếp hàng bất đồng bộ mọi sự kiện an toàn và dữ liệu GPS. | Tránh hiện tượng quá tải hệ thống Backend vào giờ cao điểm hoặc sự cố diện rộng. |
| **Safety Core & HITL** | Node.js / Python | Quản lý quy trình duyệt của Manager, phân phối cảnh báo đỏ, xử lý khiếu nại của tài xế. | Hiện thực hóa mô hình **Human-in-the-Loop**, loại trừ nguy cơ phạt oan do AI. |
| **Object Storage (S3)** | AWS S3 / MinIO | Lưu trữ tạm các video clip 5s làm bằng chứng cho Manager duyệt. | **Giải bài toán chi phí:** Chỉ lưu clip khi có vi phạm; áp dụng **S3 Lifecycle 24h tự xóa**. |
| **Audit DB (PostgreSQL)** | PostgreSQL | Lưu vết toàn bộ lịch sử vi phạm, biên bản phê duyệt của Manager. | Lưu trữ bất biến phục vụ kiểm toán, pháp lý trong **3–5 năm**. |
| **Telemetry DB** | ClickHouse | Lưu trữ chuỗi thời gian GPS, vận tốc, PERCLOS, dữ liệu lái xe liên tục. | Phục vụ **Báo cáo phân tích nhân quả (Causal Analysis)** và BI với tốc độ query mili-giây. |

---

# 🧩 MỨC C3: COMPONENT DIAGRAM (CHI TIẾT MODULE NỘI BỘ)

> **Mục tiêu:** Nhìn sâu vào bên trong 2 Container quan trọng nhất: **Mobile Safety App** và **Backend Safety & HITL Service**.

---

### 3.1. Sơ đồ Component: Mobile Safety App (On-Vehicle Edge)

```mermaid
flowchart TB
    subgraph MOBILE_EDGE ["📱 NỘI BỘ CONTAINER: MOBILE SAFETY APP"]
        subgraph SENSING_PIPELINE ["TẦNG THU NHẬN & THỊ GIÁC (INPUT & VISION)"]
            CameraModule["📷 <b>Frame Capture Manager</b><br>• Thu nhận khung hình camera trước<br>• Điều tiết cố định 15 - 20 FPS<br>• Tự động cân bằng sáng WDR"]
            FaceMeshAI["👁️ <b>MediaPipe FaceMesh Detector</b><br>• Trích xuất 468 điểm mốc khuôn mặt<br>• Chạy trên Mobile GPU (TFLite)<br>• Nhận diện tọa độ mắt, miệng, sống mũi"]
            FeatureExtractor["📐 <b>Biometric Metric Extractor</b><br>• Tính EAR (Eye Aspect Ratio)<br>• Tính MAR (Mouth Aspect Ratio)<br>• Ước lượng góc xoay đầu: Pitch, Yaw"]
        end

        subgraph DECISION_PIPELINE ["TẦNG QUYẾT ĐỊNH & CẢNH BÁO (DECISION ENGINE)"]
            RiskScorer["🧮 <b>Sliding-Window Risk Scorer</b><br>• Cửa sổ trượt 3 giây (60 frames)<br>• Tính PERCLOS liên tục<br>• Cộng điểm rủi ro đa biến (0-100)<br>• Bù trừ vận tốc xe (Lọc kẹt xe/đèn đỏ)"]
            AlertManager["🔊 <b>Local Alert Dispatcher</b><br>• Cấp 1 (Score>=40): Beep nhẹ / Rung<br>• Cấp 2 (Score>=70): Còi hú lớn (<=300ms)<br>• Cấp 3 (Score>=85): Giọng nói ép phản xạ"]
        end

        subgraph BUFFER_STORAGE ["TẦNG BẰNG CHỨNG & ĐỒNG BỘ (EVIDENCE & SYNC)"]
            CircularBuffer["📼 <b>Rolling Video Buffer</b><br>• Lưu cuốn chiếu 10s video trong RAM<br>• Khi có Cấp 3: Trích xuất clip 5s (2s trước + 3s sau)"]
            OfflineSync["💾 <b>SQLite Queue & Sync Worker</b><br>• Lưu trữ JSON sự kiện khi mất mạng<br>• Tự động retry đồng bộ khi có 4G/Wifi"]
        end
    end

    %% PIPELINE LINKS
    CameraModule -->|"Khung hình thô"| FaceMeshAI
    FaceMeshAI -->|"Tọa độ 468 landmarks"| FeatureExtractor
    FeatureExtractor -->|"Dữ liệu EAR, MAR, Pitch, Yaw"| RiskScorer

    RiskScorer -->|"Score >= 40/70/85"| AlertManager
    RiskScorer -->|"Kích hoạt khi đạt Cấp 3"| CircularBuffer
    RiskScorer -->|"Metadata sự kiện"| OfflineSync
    CircularBuffer -->|"File clip 5s"| OfflineSync

    %% STYLING
    classDef sensStyle fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc;
    classDef decStyle fill:#1e1b4b,stroke:#f43f5e,stroke-width:2px,color:#fff1f2;
    classDef bufStyle fill:#064e3b,stroke:#34d399,stroke-width:2px,color:#ecfdf5;

    class CameraModule,FaceMeshAI,FeatureExtractor sensStyle;
    class RiskScorer,AlertManager decStyle;
    class CircularBuffer,OfflineSync bufStyle;
```

---

### 3.2. Sơ đồ Component: Backend Safety & HITL Service

```mermaid
flowchart TB
    subgraph BACKEND_SAFETY ["🧠 NỘI BỘ CONTAINER: BACKEND SAFETY & HITL SERVICE"]
        EventConsumer["📥 <b>Safety Event Ingestion Worker</b><br>• Lắng nghe Kafka topic `safety-events`<br>• Xác thực schema JSON & chữ ký thiết bị"]
        
        HitlOrchestrator["⚖️ <b>HITL Workflow Engine</b><br>• Đưa vi phạm vào hàng đợi `PENDING_REVIEW`<br>• Điều phối trạng thái: CONFIRMED / REJECTED<br>• Tự động hủy nếu quá hạn (Auto-dismiss)"]
        
        RealtimeDispatcher["🚨 <b>Realtime Alert Hub (WebSocket)</b><br>• Đẩy thông báo đỏ tức thời tới Manager Dashboard<br>• Gửi lệnh can thiệp khẩn cấp (Dừng xe/Khóa cuốc)"]
        
        DriverScoreManager["📈 <b>Driver Score & Profile Service</b><br>• Trừ điểm an toàn tài xế khi xác nhận vi phạm<br>• Lưu baseline cá nhân hóa (mắt một mí, đeo kính)"]
        
        DisputeHandler["📝 <b>Dispute & Appeal Service</b><br>• Tiếp nhận khiếu nại của tài xế khi bị Admin duyệt sai<br>• Đóng băng clip 5s & Mở phiên phúc khảo cấp 2"]
        
        AuditLogger["🔒 <b>Immutable Audit Writer</b><br>• Ghi vết vĩnh viễn: Ai duyệt, duyệt lúc nào, lý do gì"]
    end

    %% CONNECTIONS
    EventConsumer -->|"Sự kiện hợp lệ"| HitlOrchestrator
    HitlOrchestrator -->|"Bắn popup khẩn"| RealtimeDispatcher
    HitlOrchestrator -->|"Ghi vết quyết định"| AuditLogger
    HitlOrchestrator -->|"Cập nhật vi phạm"| DriverScoreManager
    DisputeHandler -->|"Đóng băng sự kiện"| HitlOrchestrator

    %% STYLING
    classDef compStyle fill:#1e1b4b,stroke:#a855f7,stroke-width:2px,color:#f8fafc;
    class EventConsumer,HitlOrchestrator,RealtimeDispatcher,DriverScoreManager,DisputeHandler,AuditLogger compStyle;
```

---

# 💻 MỨC C4: CODE LEVEL DESIGN (THIẾT KẾ MÃ NGUỒN & HỢP ĐỒNG)

> **Mục tiêu:** Cung cấp mã nguồn hiện thực hóa các thuật toán mấu chốt, cấu trúc dữ liệu JSON Contract và Class Diagram chi tiết.

### 4.1. Thuật toán Cốt lõi: Chấm Điểm Rủi Ro Trượt (Sliding-Window Risk Scorer)

Dưới đây là mã nguồn TypeScript mô tả chuẩn xác logic chạy on-device trên điện thoại:

```typescript
/**
 * Cấu trúc dữ liệu đầu vào trích xuất từ từng frame hình ảnh
 */
export interface FrameBiometrics {
  frameIndex: number;
  timestampMs: number;
  ear: number;            // Eye Aspect Ratio (Mắt mở bình thường ~0.3; Nhắm mắt < 0.20)
  mar: number;            // Mouth Aspect Ratio (Miệng bình thường < 0.35; Ngáp > 0.60)
  headPitchDeg: number;   // Góc gục đầu (Gục xuống: < -20 độ)
  headYawDeg: number;     // Góc ngoảnh mặt (Quay trái/phải: > 30 độ)
  isPhoneDetected: boolean;
  vehicleSpeedKmh: number;// Vận tốc xe lấy từ GPS
}

export enum SafetyAlertLevel {
  NORMAL = 0,             // 0 - 39 điểm: Bình thường
  LEVEL_1_WARNING = 1,    // 40 - 69 điểm: Nhắc nhở nhẹ (Beep/Rung on-device)
  LEVEL_2_CRITICAL = 2,   // 70 - 84 điểm: Nguy hiểm cao (Còi hú lớn on-device)
  LEVEL_3_HITL = 3        // 85 - 100 điểm: Khẩn cấp (Upload clip 5s cho Manager duyệt)
}

export class AggregateRiskScorer {
  private readonly WINDOW_SIZE = 60; // Cửa sổ 3 giây ở tần số 20 FPS
  private buffer: FrameBiometrics[] = [];

  public evaluateFrame(current: FrameBiometrics): { score: number; level: SafetyAlertLevel } {
    this.buffer.push(current);
    if (this.buffer.length > this.WINDOW_SIZE) {
      this.buffer.shift();
    }

    // 1. Tính toán PERCLOS (% thời gian mắt nhắm trong 3 giây)
    const closedEyesFrames = this.buffer.filter(f => f.ear < 0.20).length;
    const perclos = closedEyesFrames / this.buffer.length;

    // 2. Kiểm tra gục đầu hoặc quay mặt kéo dài
    const headDownFrames = this.buffer.filter(f => f.headPitchDeg < -20 || Math.abs(f.headYawDeg) > 30).length;
    const headDeviationRatio = headDownFrames / this.buffer.length;

    // 3. Hệ số bù trừ tốc độ (Lọc False Positive khi dừng đèn đỏ hoặc kẹt xe)
    const isStoppedOrCreeping = current.vehicleSpeedKmh < 5.0;
    const speedMultiplier = isStoppedOrCreeping ? 0.35 : 1.0;

    // 4. Cộng dồn thang điểm rủi ro đa biến (0 - 100)
    let score = 0;

    // Trọng số PERCLOS (Tối đa 40 điểm)
    if (perclos >= 0.40) score += 40;
    else if (perclos >= 0.25) score += 20;

    // Trọng số Gục đầu / Ngoảnh mặt (Tối đa 30 điểm)
    if (headDeviationRatio >= 0.50) {
      score += Math.round(30 * speedMultiplier);
    }

    // Trọng số Ngáp ngủ thật (Tối đa 15 điểm - chỉ tính khi mắt nheo)
    if (current.mar > 0.60 && perclos > 0.15) {
      score += 15;
    }

    // Trọng số Sử dụng điện thoại (Tối đa 25 điểm)
    if (current.isPhoneDetected) {
      score += Math.round(25 * speedMultiplier);
    }

    score = Math.min(100, Math.max(0, score));

    // 5. Phân định cấp độ cảnh báo
    let level = SafetyAlertLevel.NORMAL;
    if (score >= 85) level = SafetyAlertLevel.LEVEL_3_HITL;
    else if (score >= 70) level = SafetyAlertLevel.LEVEL_2_CRITICAL;
    else if (score >= 40) level = SafetyAlertLevel.LEVEL_1_WARNING;

    return { score, level };
  }
}
```

---

### 4.2. Data Contract: Payload JSON Sự Kiện Vi Phạm (Mobile $\rightarrow$ Server)

```json
{
  "$schema": "https://json-schema.driverguard.io/v2/safety-event.json",
  "eventId": "evt_98f4c2e1-7b0a-4a21-9d22-12f5a8901bca",
  "driverId": "TX_HN_1042",
  "vehiclePlate": "29E-888.99",
  "tripId": "TRIP_20260928_88412",
  "timestamp": "2026-09-28T15:30:00.120Z",
  "telemetry": {
    "latitude": 21.028511,
    "longitude": 105.854444,
    "speedKmh": 85.5,
    "roadCategory": "HIGHWAY_EXPRESSWAY",
    "tripStatus": "OCCUPIED"
  },
  "aiMetrics": {
    "perclos3s": 0.55,
    "headPitchDeg": -24.5,
    "headYawDeg": 3.8,
    "aggregateRiskScore": 88
  },
  "violationType": "DROWSINESS_CONFIRMED",
  "severityLevel": "LEVEL_3_HITL",
  "evidence": {
    "clipDurationSec": 5,
    "clipS3Url": "https://s3.fleet.driverguard.io/clips/20260928/TX_HN_1042_evt_98f4c2e1.mp4",
    "clipExpiresAt": "2026-09-29T15:30:00.120Z"
  }
}
```

---

# 🔄 SƠ ĐỒ LUỒNG TOÀN TRÌNH (SEQUENCE DIAGRAM)

Quy trình phản ứng khi tài xế có dấu hiệu ngủ gật trên cao tốc:

```mermaid
sequenceDiagram
    autonumber
    actor Driver as 👨‍✈️ Tài xế Taxi
    participant App as 📱 Mobile Edge App
    participant S3 as ☁️ AWS S3 Storage
    participant API as 🚪 API Gateway & Kafka
    participant Svc as 🛡️ Safety HITL Service
    actor Mgr as 👨‍💼 Fleet Manager

    Driver->>App: Mắt nhắm > 40% trong 3s + Gục đầu (Tốc độ 85 km/h)
    Note over App: Risk Scorer tính: Score = 88 (Vượt ngưỡng Cấp 3)

    par ⚡ Xử lý tức thời on-device (<= 300ms)
        App->>Driver: 🔊 Phát còi hú khẩn cấp + Giọng nói: "Bác tài buồn ngủ, bật Hazard!"
    and 📦 Đóng gói bằng chứng & tải lên nền
        App->>S3: Upload clip ngắn 5s (Pre-signed URL)
        App->>API: Gửi Event Metadata JSON kèm link clip S3
    end

    API->>Svc: Đẩy sự kiện vào hàng đợi xử lý
    Svc->>Mgr: 🚨 Bắn popup đỏ khẩn cấp lên Dashboard (WebSocket)
    Mgr->>Mgr: Xem clip 5s xác thực hành vi

    alt 🔴 Trường hợp 1: Manager bấm [Xác nhận vi phạm]
        Mgr->>Svc: Xác nhận vi phạm + Chọn "Dừng xe an toàn"
        Svc->>App: Gửi lệnh yêu cầu tấp xe vào trạm dừng nghỉ
        Svc->>Driver: App tự bật dẫn đường tới Trạm dừng gần nhất (cách 2km)
        Svc->>Svc: Ghi Audit Log (Lưu 3 năm), khóa nhận cuốc tiếp theo
    else 🟢 Trường hợp 2: Manager bấm [Bác bỏ - False Positive]
        Mgr->>Svc: Bác bỏ (Lý do: Tài xế chói nắng/nheo mắt)
        Svc->>S3: Kích hoạt xóa clip ngay lập tức
        Svc->>Svc: Ghi nhận dữ liệu tinh chỉnh model AI
        Note over Driver: Tài xế không bị trừ điểm an toàn
    end
```

---

# 🎯 ĐÁNH GIÁ ĐÁP ỨNG CÁC RÀNG BUỘC CỦA MENTOR

| Ràng buộc Mentor đặt ra | Giải pháp thiết kế trong tài liệu SAD này | Vị trí thể hiện |
| :--- | :--- | :--- |
| **Quy mô 100.000 xe** | Không dùng WebSocket stream video liên tục; sử dụng kiến trúc **Event-Driven JSON siêu nhẹ qua Kafka**; chỉ upload clip 5s khi sự cố. | Mức C2 (Container) |
| **Độ trễ cảnh báo on-device** | Pipeline thị giác MediaPipe + Risk Scorer chạy cục bộ trên mobile với thời gian phản hồi $\le 300\text{ ms}$, hoạt động cả khi mất mạng. | Mức C3.1 (Mobile App) |
| **Chi phí lưu trữ & Quyền riêng tư** | Áp dụng chính sách **Retention 24h đối với clip S3** (tự xóa qua Lifecycle Rule); Audit Log lưu 3–5 năm dạng bản ghi chữ. | Mức C1, C2 & Data Contract |
| **Tránh báo động phiền hà (False Alarm)** | Thuật toán **Cửa sổ trượt 3 giây** kết hợp bù trừ vận tốc GPS (loại trừ xe dừng đèn đỏ, ngáp giả, nói chuyện). | Mức C4 (Thuật toán Risk Scorer) |
| **Đảm bảo công bằng cho tài xế** | Thiết kế quy trình **Human-in-the-Loop (HITL)** kèm cơ chế Khiếu nại 1 chạm và đối soát dữ liệu viễn thông. | Mức C3.2 & Sequence Diagram |
