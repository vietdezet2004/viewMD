# DRIVERGUARD SOFTWARE ARCHITECTURE DOCUMENT (SAD)
## Tài Liệu Thiết Kế Kiến Trúc Phần Mềm Thống Nhất — Chuẩn C4 Model & Nền Tảng MQTT

| Thuộc tính | Nội dung |
| :--- | :--- |
| **Dự án** | **DriverGuard** — Nền tảng Giám sát An toàn Tài xế & Vận hành Đội xe |
| **Trọng tâm cốt lõi (Core Scope)** | **Safety Core:** Giám sát camera on-device, cảnh báo vi ngủ/buồn ngủ $\le 300\text{ ms}$, luồng duyệt an toàn 2 cấp (HITL) |
| **Phạm vi mở rộng (Phase 2 Roadmap)** | **Smart Fleet Dispatch:** Tối ưu hóa điều phối xe theo nhu cầu và doanh thu (Non-core) |
| **Quy mô mục tiêu** | 100.000 phương tiện kết nối đồng thời (Điển hình: Đội xe điện Xanh SM) |
| **Nền tảng công nghệ chính** | Android Edge AI (MediaPipe), **MQTT over TLS (QoS 0/1)**, Deterministic Rule Engine, PostgreSQL + S3 |
| **Phiên bản tài liệu** | **3.0 — Unified Master Edition** (Hợp nhất triết lý bài toán an toàn & hạ tầng kỹ thuật v2) |
| **Trạng thái** | Sẵn sàng cho Architecture & MVP Review |

---

## 1. BÀI TOÁN CỐT LÕI & NGUYÊN TẮC THIẾT KẾ (PROBLEM FRAMING)

### 1.1. Tuyên bố Bài toán & Sứ mệnh
Tài xế taxi vận hành theo ca dài (8–10 tiếng), đặc biệt ca tối hoặc đi cao tốc, thường bị suy giảm mức độ tỉnh táo sinh học dẫn đến hiện tượng **vi ngủ (micro-sleep)** trong 2–3 giây. Ở vận tốc $80\text{ km/h}$, 2 giây xe trôi tự do gần $50\text{ mét}$.

Hệ thống **DriverGuard** được xây dựng với sứ mệnh:
> **"Bảo vệ tính mạng tài xế và hành khách bằng hệ thống cảnh báo tức thời tại biên (On-device Edge AI), đồng thời chuyển đổi dữ liệu an toàn thành hành động can thiệp có kiểm soát của con người (Human-in-the-Loop) thông qua giao thức kết nối MQTT tối ưu cho 100.000 xe."**

### 1.2. Phân định Ranh giới Phạm vi (Scope Boundary)
* 🔴 **TRỌNG TÂM CỐT LÕI (CORE MVP):**
  1. *Edge Safety:* Camera AI phát hiện nhắm mắt kéo dài ($PERCLOS$), gục đầu, ngáp ngủ; còi hú cảnh báo tại chỗ $\le 300\text{ ms}$ (Offline-first, không phụ thuộc mạng).
  2. *Connected Fleet Transport:* Sử dụng **MQTT over TLS** để truyền telemetry và sự kiện an toàn nhẹ, bền bỉ, tiết kiệm pin.
  3. *Quy trình Duyệt 2 Cấp (2-Tier Approval):* Quản lý duyệt can thiệp an toàn $\rightarrow$ Tài xế tự nguyện chấp thuận dừng nghỉ.
  4. *Chính sách Dữ liệu An toàn:* Clip 5s tự hủy sau 24h, Audit log lưu trữ 3–5 năm phục vụ phân tích nhân quả và kiểm toán.
* ⚪ **TÍNH NĂNG MỞ RỘNG (PHASE 2 ROADMAP):**
  * Tự động dự báo nhu cầu khách (Demand Model) và gợi ý điều phối xe rảnh tăng doanh thu. (Module này tách riêng, không nằm trong luồng core MVP).

### 1.3. Nguyên tắc Bất biến (Architectural Invariants)
1. **An toàn tại biên là tối thượng:** Cảnh báo cứu mạng trên xe không bao giờ được chờ phản hồi từ Cloud hoặc kết nối mạng.
2. **Không stream video liên tục:** 99% dữ liệu là JSON metadata. Chỉ trích xuất clip ngắn 5 giây khi có vi phạm khẩn cấp.
3. **Quyền quyết định thuộc về con người:** AI chỉ phát hiện và đề xuất; Manager duyệt chiến thuật; Tài xế có quyền chấp thuận/từ chối hành động áp dụng cho bản thân.
4. **Kỷ luật bằng chứng:** Mọi cảnh báo, quyết định can thiệp và phản hồi của tài xế đều được lưu vết bất biến trong Audit Log.

---

## 2. KIẾN TRÚC MỨC C1: SYSTEM CONTEXT (BỐI CẢNH HỆ THỐNG)

Biểu đồ C1 định vị DriverGuard trong môi trường vận hành thực tế, tương tác với Tài xế, Quản lý đội xe và các hệ thống ngoài.

```mermaid
flowchart TB
    %% USERS
    subgraph USERS ["👥 TÁC NHÂN CON NGƯỜI"]
        Driver["👨‍✈️ <b>Tài xế Taxi (Driver)</b><br>• Lái xe trong cabin<br>• Nhận cảnh báo âm thanh/rung on-device (<=300ms)<br>• Phản hồi chấp thuận dừng nghỉ / Khiếu nại khi bị nhận diện sai"]
        Manager["👨‍💼 <b>Fleet Safety Manager (Cán bộ An toàn Đội xe)</b><br>• Giám sát rủi ro an toàn đội xe theo thời gian thực<br>• Xem clip 5s duyệt vi phạm nghiêm trọng (HITL)<br>• Phê duyệt đề xuất tạm dừng ca / Hỗ trợ khẩn cấp"]
    end

    %% CORE SYSTEM
    subgraph DRIVERGUARD_CORE ["🛡️ NỀN TẢNG DRIVERGUARD (CORE SYSTEM)"]
        DriverGuardPlatform["<b>DRIVERGUARD SAFETY PLATFORM</b><br>━━━━━━━━━━━━━━━━━━━━━━━━━━━━<br>• Edge AI: Phân tích thị giác on-device (20 FPS)<br>• Transport: MQTT Broker Cluster chịu tải 100k xe<br>• Rule Engine: Ràng buộc an toàn & giờ lái tối đa<br>• HITL Workflow: Quản lý luồng duyệt 2 cấp<br>• Báo cáo phân tích nhân quả (Causal Safety Reports)"]
    end

    %% FUTURE EXTENSION (NON-CORE)
    subgraph FUTURE_EXTENSION ["💡 TÍNH NĂNG MỞ RỘNG (PHASE 2 ROADMAP)"]
        DispatchEngine["🧭 <i>[Tương lai] Smart Fleet Dispatch</i><br>• Tối ưu điều phối xe rảnh theo nhu cầu & doanh thu"]
    end

    %% EXTERNAL SYSTEMS
    subgraph EXT_SYSTEMS ["🏢 HỆ THỐNG LIÊN KẾT BÊN NGOÀI"]
        FMS["🚗 <b>Hệ Thống Vận Hành Hãng (Xanh SM FMS)</b><br>• Quản lý thông tin tài xế, ca trực, cuốc khách<br>• Đồng bộ trạng thái tạm khóa nhận cuốc khi tài xế kiệt sức"]
        S3Storage["☁️ <b>Cloud Object Storage (AWS S3 / MinIO)</b><br>• Lưu trữ video clip 5s bằng chứng vi phạm<br>• Tự động xóa sau 24h qua S3 Lifecycle"]
        MapsAPI["🗺️ <b>Bản Đồ Số & Trạm Dừng Nghỉ</b><br>• Tọa độ nút giao cao tốc, cây xăng, trạm dừng nghỉ<br>• Điều hướng an toàn khi tài xế cần nghỉ khẩn cấp"]
    end

    %% RELATIONSHIPS
    Driver -->|"1. Camera soi khuôn mặt on-device<br>2. Nhận còi hú/giọng nói cảnh báo (<=300ms)<br>3. Bấm phản hồi / Khiếu nại"| DriverGuardPlatform
    Manager -->|"1. Xem bản đồ an toàn đội xe realtime<br>2. Duyệt clip 5s vi phạm nghiêm trọng<br>3. Phê duyệt chỉ thị can thiệp an toàn"| DriverGuardPlatform

    DriverGuardPlatform -->|"Đồng bộ ID tài xế, gửi lệnh tạm dừng phân cuốc"| FMS
    DriverGuardPlatform -->|"Upload clip 5s (Pre-signed URL, TTL 24h)"| S3Storage
    DriverGuardPlatform -->|"Truy vấn trạm dừng nghỉ gần nhất trên cao tốc"| MapsAPI

    DriverGuardPlatform -.-|"Tương lai: Dữ liệu an toàn hỗ trợ điều phối"| DispatchEngine
    DispatchEngine -.-|"Tương lai: Đề xuất điều xe"| FMS

    %% STYLING
    classDef userStyle fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc;
    classDef coreStyle fill:#1e1b4b,stroke:#a855f7,stroke-width:3px,color:#f8fafc;
    classDef extStyle fill:#1e293b,stroke:#94a3b8,stroke-width:2px,color:#e2e8f0;
    classDef futureStyle fill:#1e293b,stroke:#fbbf24,stroke-width:2px,stroke-dasharray: 5 5,color:#fef3c7;

    class Driver,Manager userStyle;
    class DriverGuardPlatform coreStyle;
    class FMS,S3Storage,MapsAPI extStyle;
    class DispatchEngine futureStyle;
```

---

## 3. KIẾN TRÚC MỨC C2: CONTAINER DIAGRAM (CÁC KHỐI ỨNG DỤNG)

Phân rã hệ thống thành các Container độc lập. Kiến trúc chuyển đổi toàn diện sang **MQTT over TLS** để giải quyết triệt để bài toán 100.000 xe.

```mermaid
flowchart TB
    subgraph VEHICLE_EDGE ["🚗 PHÍA PHƯƠNG TIỆN (VEHICLE EDGE CONTAINER)"]
        MobileApp["📱 <b>DriverGuard Android App</b><br><i>(Kotlin / MediaPipe C++ / SQLite)</i><br>━━━━━━━━━━━━━━━━━━━━━━━━━<br>• Thu nhận khung hình 20 FPS<br>• MediaPipe FaceMesh tính EAR, MAR, Pose<br>• Cửa sổ trượt 3s tính Risk Score<br>• Còi hú cảnh báo on-device (<=300ms)<br>• Cắt buffer clip 5s khi có sự cố Cấp 3<br>• SQLite Durable Outbox (Offline-first)"]
    end

    subgraph TRANSPORT_LAYER ["⚡ TẦNG KẾT NỐI & ĐIỀU PHỐI TIN NHẮN (MQTT TRANSPORT)"]
        MQTTBroker["📨 <b>MQTT Broker Cluster</b><br><i>(EMQX / HiveMQ / Mosquitto)</i><br>━━━━━━━━━━━━━━━━━━━━━━━━━<br>• Kết nối bảo mật MQTT over TLS (Port 8883)<br>• Topic GPS: `telemetry/gps` (QoS 0)<br>• Topic Vi phạm: `safety/alert` (QoS 1)<br>• Topic Lệnh: `command/driver` (QoS 1)<br>• Hỗ trợ Keep-alive & Quản lý Session 100k xe"]
    end

    subgraph BACKEND_PLATFORM ["🧠 TẦNG NGHIỆP VỤ AN TOÀN (BACKEND PLATFORM)"]
        IngestionWorker["📥 <b>Telemetry & Safety Ingestor</b><br><i>(Go / Node.js)</i><br>• Tiêu thụ sự kiện từ MQTT Broker<br>• Xác thực token & deduplicate theo `eventId`"]
        
        RuleEngine["⚖️ <b>Deterministic Safety Rule Engine</b><br>• Áp dụng luật cứng: Lái liên tục > 4 tiếng<br>• Đánh giá ngưỡng rủi ro tích lũy (Cumulative Risk)"]
        
        SafetyHITLService["🛡️ <b>Safety Core & HITL Service</b><br>• Quản lý hàng đợi duyệt clip 5s cho Manager<br>• Điều phối luồng 2 cấp chấp thuận (2-Tier Approval)<br>• Xử lý khiếu nại (Dispute) của tài xế"]
        
        CausalReportEngine["📊 <b>Causal Analytics Engine</b><br>• Phân tích tương quan: Giờ lái ↔ Tần suất buồn ngủ<br>• Tương quan: Điểm an toàn ↔ Rating/Doanh thu cuốc"]
        
        RetentionWorker["🧹 <b>Data Retention Janitor</b><br>• Quét & Xóa clip S3 sau 24h-48h<br>• Nén Cold Storage log sau 90 ngày"]
    end

    subgraph STORAGE_LAYER ["💾 TẦNG LƯU TRỮ DỮ LIỆU (PERSISTENCE)"]
        PostgresDB[("🗄️ <b>PostgreSQL Database</b><br>• Hồ sơ tài xế, xe, chuyến đi<br>• <b>Audit Log duyệt an toàn (3-5 năm)</b><br>• Lịch sử khiếu nại")]
        TelemetryDB[("📈 <b>Time-Series Telemetry Store</b><br>• Lịch sử GPS, vận tốc, risk scores<br>• Dữ liệu phục vụ báo cáo nhân quả")]
        S3Bucket[("☁️ <b>AWS S3 / MinIO Object Storage</b><br>• Video clip 5s bằng chứng vi phạm<br>• <b>S3 Lifecycle Rule tự xóa sau 24h</b>")]
    end

    subgraph WEB_DASHBOARD ["💻 PHÍA QUẢN LÝ (WEB CONTAINER)"]
        ManagerUI["🖥️ <b>Fleet Safety Dashboard</b><br><i>(React / Next.js + TailwindCSS)</i><br>• Bản đồ an toàn đội xe thời gian thực<br>• Popup video clip 5s duyệt vi phạm (HITL)<br>• Nút gửi khuyến nghị dừng nghỉ / Gọi khẩn cấp"]
    end

    %% FLOWS
    MobileApp -->|"1. Publish GPS (QoS 0) & Safety Event (QoS 1)"| MQTTBroker
    MQTTBroker -->|"2. Subscribe lệnh can thiệp / Đề xuất nghỉ (QoS 1)"| MobileApp
    MobileApp -->|"3. Upload clip 5s khi kích hoạt Cấp 3 (HTTPS)"| S3Bucket

    MQTTBroker -->|"Chuyển tiếp message"| IngestionWorker
    IngestionWorker -->|"Dữ liệu an toàn"| RuleEngine
    RuleEngine -->|"Sự kiện vượt ngưỡng"| SafetyHITLService
    IngestionWorker -->|"Lưu telemetry"| TelemetryDB

    SafetyHITLService -->|"Ghi nhận vi phạm & Audit Trail"| PostgresDB
    SafetyHITLService -->|"Push cảnh báo đỏ realtime (WebSocket/SSE)"| ManagerUI
    ManagerUI -->|"Manager duyệt đề xuất (HTTPS REST)"| SafetyHITLService
    SafetyHITLService -->|"Publish lệnh dừng nghỉ tới xe (MQTT QoS 1)"| MQTTBroker

    CausalReportEngine -->|"Truy vấn phân tích dữ liệu lớn"| TelemetryDB
    CausalReportEngine -->|"Cung cấp biểu đồ vận hành"| ManagerUI
    RetentionWorker -->|"Xóa clip hết hạn 24h"| S3Bucket

    %% STYLING
    classDef edgeStyle fill:#064e3b,stroke:#34d399,stroke-width:2px,color:#ecfdf5;
    classDef mqttStyle fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff;
    classDef svcStyle fill:#312e81,stroke:#a78bfa,stroke-width:2px,color:#f5f3ff;
    classDef dbStyle fill:#1f2937,stroke:#9ca3af,stroke-width:2px,color:#f9fafb;
    classDef webStyle fill:#701a75,stroke:#f472b6,stroke-width:2px,color:#fdf2f8;

    class MobileApp edgeStyle;
    class MQTTBroker mqttStyle;
    class IngestionWorker,RuleEngine,SafetyHITLService,CausalReportEngine,RetentionWorker svcStyle;
    class PostgresDB,TelemetryDB,S3Bucket dbStyle;
    class ManagerUI webStyle;
```

---

## 4. KIẾN TRÚC MỨC C3: COMPONENT DIAGRAM (CHI TIẾT NỘI BỘ)

### 4.1. Nội bộ Mobile Safety App (On-Vehicle Edge)
Trách nhiệm xử lý camera on-device, tính Risk Score và điều phối còi báo cục bộ $\le 300\text{ ms}$:

```mermaid
flowchart TB
    subgraph MOBILE_APP_COMPONENTS ["📱 NỘI BỘ CONTAINER: MOBILE SAFETY APP"]
        FrameCapturer["📷 <b>Frame Capture Manager</b><br>• Camera2 API điều tiết 15 - 20 FPS<br>• Cân bằng sáng WDR chống chói"]
        FaceDetector["👁️ <b>MediaPipe FaceMesh Detector</b><br>• Trích xuất 468 điểm mốc khuôn mặt<br>• Chạy trên Mobile GPU (TFLite C++)"]
        BiometricExtractor["📐 <b>Biometric Feature Extractor</b><br>• Tính EAR (Mắt), MAR (Miệng)<br>• Tính góc gục đầu (Pitch), quay mặt (Yaw)"]
        RiskEngine["🧮 <b>Sliding-Window Risk Scorer</b><br>• Cửa sổ trượt 3 giây (60 frames)<br>• Tính PERCLOS, kết hợp đa tín hiệu<br>• Bù trừ vận tốc xe (lọc dừng đèn đỏ)"]
        AlertDispatcher["🔊 <b>Local Audio Alert Dispatcher</b><br>• Cấp 1 (Score>=40): Beep nhẹ / Rung<br>• Cấp 2 (Score>=70): Còi hú lớn (<=300ms)<br>• Cấp 3 (Score>=85): Giọng nói ép phản xạ"]
        VideoBuffer["📼 <b>Rolling Video Buffer</b><br>• Lưu cuốn chiếu 10s video trong RAM<br>• Cắt clip 5s (2s trước + 3s sau vi phạm)"]
        DurableOutbox["💾 <b>SQLite Outbox & MQTT Client</b><br>• Lưu sự kiện offline khi mất sóng cao tốc<br>• Tự động publish lại qua MQTT khi có mạng"]
    end

    FrameCapturer --> FaceDetector
    FaceDetector --> BiometricExtractor
    BiometricExtractor --> RiskEngine
    RiskEngine --> AlertDispatcher
    RiskEngine -->|"Kích hoạt khi đạt Cấp 3"| VideoBuffer
    RiskEngine --> DurableOutbox
    VideoBuffer --> DurableOutbox

    classDef c3Style fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc;
    class FrameCapturer,FaceDetector,BiometricExtractor,RiskEngine,AlertDispatcher,VideoBuffer,DurableOutbox c3Style;
```

---

## 5. THIẾT KẾ MỨC C4: CODE LEVEL DESIGN & DATA CONTRACTS

### 5.1. Thuật toán Cốt lõi: Cửa Sổ Trượt Tính Risk Score (`AggregateRiskScorer.ts`)

```typescript
export interface FrameBiometrics {
  timestampMs: number;
  ear: number;            // Eye Aspect Ratio (Mắt mở bình thường ~0.3; Nhắm mắt < 0.20)
  mar: number;            // Mouth Aspect Ratio (Miệng bình thường < 0.35; Ngáp > 0.60)
  headPitchDeg: number;   // Góc cúi gục đầu (Gục xuống: < -20 độ)
  headYawDeg: number;     // Góc ngoảnh mặt khỏi đường (Quay trái/phải: > 30 độ)
  isPhoneDetected: boolean;
  vehicleSpeedKmh: number;// Vận tốc xe từ GPS
}

export enum SafetyAlertLevel {
  NORMAL = "NORMAL",                     // 0 - 39 điểm
  LEVEL_1_WARNING = "LEVEL_1_WARNING",   // 40 - 69 điểm (Beep nhẹ on-device)
  LEVEL_2_CRITICAL = "LEVEL_2_CRITICAL", // 70 - 84 điểm (Còi to on-device <=300ms)
  LEVEL_3_ESCALATION = "LEVEL_3_HITL"    // 85 - 100 điểm (Gửi Clip 5s duyệt HITL)
}

export class AggregateRiskScorer {
  private readonly WINDOW_FRAMES = 60; // 3 giây ở tần số 20 FPS
  private buffer: FrameBiometrics[] = [];

  public processFrame(current: FrameBiometrics): { score: number; level: SafetyAlertLevel } {
    this.buffer.push(current);
    if (this.buffer.length > this.WINDOW_FRAMES) {
      this.buffer.shift();
    }

    // 1. Tính toán PERCLOS (% thời gian mắt nhắm trong 3s)
    const closedEyesFrames = this.buffer.filter(f => f.ear < 0.20).length;
    const perclos = closedEyesFrames / this.buffer.length;

    // 2. Kiểm tra gục đầu hoặc quay mặt kéo dài
    const headDeviationFrames = this.buffer.filter(
      f => f.headPitchDeg < -20 || Math.abs(f.headYawDeg) > 30
    ).length;
    const headDeviationRatio = headDeviationFrames / this.buffer.length;

    // 3. Hệ số bù trừ tốc độ GPS (Lọc kẹt xe / dừng đèn đỏ)
    const speedFactor = current.vehicleSpeedKmh < 5.0 ? 0.35 : 1.0;

    // 4. Cộng dồn thang điểm rủi ro đa biến (0 - 100)
    let score = 0;
    if (perclos >= 0.40) score += 40;
    else if (perclos >= 0.25) score += 20;

    if (headDeviationRatio >= 0.50) score += Math.round(30 * speedFactor);
    if (current.mar > 0.60 && perclos > 0.15) score += 15; // Ngáp thật khi mắt nheo
    if (current.isPhoneDetected) score += Math.round(25 * speedFactor);

    score = Math.min(100, Math.max(0, score));

    // 5. Xác định cấp độ cảnh báo
    let level = SafetyAlertLevel.NORMAL;
    if (score >= 85) level = SafetyAlertLevel.LEVEL_3_ESCALATION;
    else if (score >= 70) level = SafetyAlertLevel.LEVEL_2_CRITICAL;
    else if (score >= 40) level = SafetyAlertLevel.LEVEL_1_WARNING;

    return { score, level };
  }
}
```

---

### 5.2. Data Contract: MQTT Payload Sự Kiện An Toàn (`safety/alert`)

```json
{
  "eventId": "evt_98f4c2e1-7b0a-4a21-9d22-12f5a8901bca",
  "eventType": "safety.drowsiness.escalation.v1",
  "occurredAt": "2026-09-28T15:30:00.120Z",
  "deviceId": "DEV_ANDROID_8841",
  "driverId": "TX_HN_1042",
  "vehiclePlate": "29E-888.99",
  "tripId": "TRIP_20260928_88412",
  "telemetry": {
    "latitude": 21.028511,
    "longitude": 105.854444,
    "speedKmh": 85.5,
    "roadType": "HIGHWAY_EXPRESSWAY"
  },
  "metrics": {
    "perclos3s": 0.55,
    "headPitchDeg": -24.5,
    "headYawDeg": 3.8,
    "riskScore": 88
  },
  "evidence": {
    "hasVideoClip": true,
    "clipDurationSec": 5,
    "clipS3Url": "https://s3.fleet.driverguard.io/clips/20260928/TX_HN_1042_evt_98f4c2e1.mp4",
    "clipExpiresAt": "2026-09-29T15:30:00.120Z"
  }
}
```

---

## 6. SƠ ĐỒ LUỒNG RUNTIME: QUY TRÌNH DUYỆT 2 CẤP (2-TIER HITL APPROVAL)

Minh họa quy trình xử lý nhân văn và chặt chẽ khi tài xế ngủ gật trên cao tốc:

```mermaid
sequenceDiagram
    autonumber
    actor Driver as 👨‍✈️ Tài xế Taxi
    participant App as 📱 Android Edge App
    participant S3 as ☁️ AWS S3 Storage
    participant MQTT as 📨 MQTT Broker
    participant Svc as 🛡️ Safety HITL Service
    actor Mgr as 👨‍💼 Fleet Safety Manager

    Driver->>App: Mắt nhắm > 40% trong 3s + Gục đầu (Tốc độ 85 km/h)
    Note over App: Risk Scorer tính: Score = 88 (Vượt ngưỡng Cấp 3)

    par ⚡ Xử lý cứu mạng tại chỗ (<= 300ms)
        App->>Driver: 🔊 Phát còi hú khẩn cấp + Giọng nói: "Bác tài buồn ngủ, bật Hazard!"
    and 📦 Đóng gói bằng chứng & gửi MQTT
        App->>S3: Upload clip ngắn 5s (Pre-signed URL)
        App->>MQTT: Publish topic `safety/alert` (QoS 1, kèm link clip S3)
    end

    MQTT->>Svc: Deliver safety event
    Svc->>Mgr: 🚨 Bắn popup đỏ khẩn cấp lên Dashboard (WebSocket/SSE)
    Mgr->>Mgr: Xem clip 5s xác thực hành vi

    alt 🔴 Cấp duyệt 1: Manager bấm [Xác nhận vi phạm & Đề xuất dừng nghỉ]
        Mgr->>Svc: Xác nhận vi phạm
        Svc->>MQTT: Publish topic `command/driver/TX_HN_1042` (Lệnh an toàn)
        MQTT->>App: Giao diện App chuyển sang đề xuất: "Hệ thống khuyến nghị bạn nghỉ 30 phút"
        
        alt 🟢 Cấp duyệt 2: Tài xế bấm [Chấp thuận nghỉ ngơi]
            Driver->>App: Bấm "Đồng ý dừng xe"
            App->>Driver: Điều hướng tới Trạm dừng nghỉ gần nhất (cách 2km)
            App->>MQTT: Gửi ACK "ACCEPTED"
            Svc->>Svc: Ghi Audit Log, kích hoạt thời gian nghỉ an toàn (30 phút)
        else 🟡 Cấp duyệt 2: Tài xế bấm [Khiếu nại / Tôi tỉnh táo]
            Driver->>App: Bấm "Khiếu nại (Chói nắng)"
            App->>MQTT: Gửi ACK "DISPUTED"
            Svc->>Svc: Đóng băng clip 5s, chuyển phiên phúc khảo cấp 2 (Không phạt tài xế)
        end
    else 🟢 Manager bấm [Bác bỏ - False Positive]
        Mgr->>Svc: Bác bỏ cảnh báo
        Svc->>S3: Xóa clip ngay lập tức
        Svc->>Svc: Gắn nhãn dữ liệu để tinh chỉnh mô hình AI
    end
```

---

## 7. CHÍNH SÁCH LƯU TRỮ DỮ LIỆU & BẢO VỆ RIÊNG TƯ (RETENTION POLICY)

| Loại dữ liệu | Vị trí lưu trữ | Vòng đời (TTL) | Cơ chế xóa / chuyển vùng | Mục đích sử dụng |
| :--- | :--- | :--- | :--- | :--- |
| **Video Clip ngắn (5s)** | AWS S3 / MinIO | **24 giờ – 48 giờ** | Tự động xóa (Auto-expire qua S3 Lifecycle) | Phục vụ Manager xác thực vi phạm, xóa ngay để bảo vệ quyền riêng tư |
| **Raw Telemetry (GPS thô)** | Time-Series DB | **30 ngày** | Sau 30 ngày tự động nén hoặc downsampling | Phục vụ đối soát sự cố và đánh giá ca chạy |
| **Audit Log (Lịch sử duyệt & Khiếu nại)** | PostgreSQL (Append-only) | **3 – 5 năm** | Không xóa tự động; lưu vết ID tài xế, quyết định duyệt | Phục vụ kiểm toán an toàn, pháp lý giao thông |
| **Aggregated Metrics** | Data Warehouse | **Vĩnh viễn** | Dữ liệu dạng số đã ẩn danh: heat-map mệt mỏi, điểm an toàn tuần | Phục vụ nghiên cứu nhân quả và báo cáo vận hành |

---

## 8. NĂNG LỰC HỆ THỐNG VỚI KỊCH BẢN 100.000 XE (CAPACITY SCENARIO)

* **Giao thức MQTT:** 100.000 kết nối thường trực duy trì keep-alive (60 giây/lần) chỉ tiêu tốn khoảng **$1.667\text{ pings/giây}$**, RAM trên cụm EMQX/HiveMQ chỉ mất khoảng **$1.5 - 2\text{ GB RAM}$** (hoàn toàn chịu tải nhẹ nhàng so với việc duy trì 100k kết nối WebSocket truyền thống).
* **Băng thông Video:** Chỉ có tối đa $0.5\% - 1\%$ số xe vi phạm Cấp 3 mỗi ngày. Trung bình 1 ngày hệ thống chỉ tiếp nhận khoảng $500 - 1.000$ clip 5s (mỗi clip ~1.5MB $\approx 1.5\text{ GB/ngày}$), chi phí lưu trữ S3 chỉ tốn vài chục nghìn VNĐ/tháng.

---

## 9. ĐÁNH GIÁ ĐÁP ỨNG TOÀN DIỆN CÁC RÀNG BUỘC CỦA MENTOR

1. **Khắc phục triệt để lỗi WebSocket quá tải:** Sử dụng **MQTT over TLS (Port 8883)** phân chia rõ ràng QoS 0 cho GPS và QoS 1 cho vi phạm an toàn.
2. **Khắc phục lỗi cảnh báo phiền hà (False Alarms):** Áp dụng thuật toán **Cửa sổ trượt 3 giây** tính điểm rủi ro tổng hợp ($PERCLOS + Head Pose + GPS Speed$).
3. **Giải quyết bài toán Chi phí & Quyền riêng tư:** Áp dụng cơ chế **Buffer 5s upload S3 tự hủy sau 24h** thay cho WebRTC cồng kềnh.
4. **Bảo đảm quyền con người:** Luồng **HITL 2 cấp chấp thuận** đảm bảo hệ thống không độc đoán phạt oan tài xế.
5. **Tiến độ khả thi cho MVP tuần tới:** Tập trung 100% tài nguyên làm xong **Safety Core** chạy thông luồng từ Mobile $\rightarrow$ MQTT $\rightarrow$ Dashboard.
