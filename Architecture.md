# Kiến Trúc Hệ Thống (Architecture Document) — Đề Tài DEV-02

> **Đề tài:** DriverGuard — Hệ thống Cảnh báo Ngủ gật và Mất tập trung Tài xế (Driver Monitoring System - DMS)  
> **Cột mốc:** Gate 2 — Architecture & Pipeline Design  
> **Đội thi:** P-136 (VinAI / AI20K)

---

## 1. Tổng Quan Hệ Thống (System Overview)

DriverGuard là hệ thống giám sát an toàn tài xế thông minh (DMS) kết hợp giữa **Edge AI (Inference trực tiếp trên thiết bị di động/camera gắn trên xe)** và **Cloud Management (Giám sát đội xe tập trung qua Backend FastAPI & MQTT Broker)**. 

Hệ thống theo dõi liên tục khuôn mặt tài xế trong thời gian thực qua Camera, trích xuất các điểm mốc sinh trắc học (Face Mesh Landmarks) để phát hiện sớm các dấu hiệu buồn ngủ (nhắm mắt kéo dài EAR, ngáp nhiều lần MAR, gục đầu Head Pitch), từ đó lập tức kích hoạt cảnh báo âm thanh đa tầng trên cabin xe và đồng bộ dữ liệu vi phạm về trung tâm điều hành.

---

## 2. Sơ Đồ Components (Các Thành Phần Hệ Thống)

Sơ đồ thể hiện đầy đủ các khối chức năng từ Input, Xử lý thị giác máy tính, Mô hình AI/Logic phán đoán, đến Output và Hệ thống ghi log.

```mermaid
graph TB
    subgraph Input_Module["1. INPUT MODULE"]
        CAM["📷 Camera Thiết Bị (CameraX / Webcam)"]
        VIDEO_IN["🎥 Video File (Test / Benchmark Dataset)"]
    end

    subgraph Processing_Module["2. PROCESSING MODULE"]
        FRAME_CAP["Frame Capture (30 FPS Stream)"]
        PRE_PROC["Tiền Xử Lý: Resize, Luma/Ánh Sáng, Format (MPImage/Bitmap)"]
        MEDIAPIPE["🧠 Google MediaPipe FaceLandmarker<br/>(468+ 3D Face Mesh Landmarks)"]
    end

    subgraph AI_Logic_Module["3. AI LOGIC & METRICS EXTRACTION"]
        FEATURE_EXT["Khối Trích Xuất Chỉ Số Hình Học<br/>(FaceFeatureExtractor)"]
        EAR["👁️ EAR (Eye Aspect Ratio)<br/>Mắt Trái & Mắt Phải"]
        MAR["👄 MAR (Mouth Aspect Ratio)<br/>Độ Mở Miệng (Ngáp)"]
        POSE["📐 Head Pose Estimation<br/>Pitch (Gục đầu), Yaw (Quay mặt), Roll"]
        PERCLOS["⏱️ PERCLOS_3s Window<br/>Tỷ Lệ Nhắm Mắt / 3 Giây"]
        
        DECISION["⚖️ Động Cơ Ra Quyết Định<br/>(DmsDecisionEngine / RiskEngine)"]
        THRESHOLDS["⚙️ Ngưỡng An Toàn (Thresholds)<br/>• EAR < 0.18 (Drowsiness)<br/>• MAR > 0.65 (Yawning)<br/>• Pitch < -20° (Head Nod)<br/>• PERCLOS >= 0.40"]
    end

    subgraph Output_Module["4. OUTPUT & INTERACTION MODULE"]
        UI_OVERLAY["🖥️ Giao Diện UI: FaceOverlayView<br/>(Bounding Box, Cảnh Báo Đỏ, Trạng Thái)"]
        AUDIO_BUZZER["🔊 Cảnh Báo Âm Thanh & Rung<br/>(Audio Alert / Tone Buzzer / Vibration)"]
        LOCAL_OUTBOX["💾 SQLite Local Outbox (QoS 1 Buffer)"]
    end

    subgraph Cloud_Logging_Module["5. LOGGING & CLOUD SYNC"]
        MQTT["📡 EMQX MQTT Broker (Topic: alert/driver/...)"]
        BACKEND["🚀 FastAPI Backend Core & HITL Review"]
        MINIO["🪣 Object Storage (MinIO / S3: Clip 5s Bằng Chứng)"]
        PHOENIX_LOG["📊 Phoenix Agent AI Logger (.ai-log Hook)"]
    end

    %% Luồng kết nối giữa các khối
    CAM --> FRAME_CAP
    VIDEO_IN --> FRAME_CAP
    FRAME_CAP --> PRE_PROC
    PRE_PROC --> MEDIAPIPE
    MEDIAPIPE --> FEATURE_EXT

    FEATURE_EXT --> EAR
    FEATURE_EXT --> MAR
    FEATURE_EXT --> POSE
    EAR --> PERCLOS

    EAR --> DECISION
    MAR --> DECISION
    POSE --> DECISION
    PERCLOS --> DECISION
    THRESHOLDS -.-> DECISION

    DECISION -->|Nguy Cơ / Vi Phạm| UI_OVERLAY
    DECISION -->|Nguy Cơ / Vi Phạm| AUDIO_BUZZER
    DECISION -->|Sự Kiện An Toàn| LOCAL_OUTBOX

    LOCAL_OUTBOX -->|Đồng bộ MQTT| MQTT
    MQTT --> BACKEND
    LOCAL_OUTBOX -->|Tải clip 5s| MINIO
    BACKEND -.-> PHOENIX_LOG
```

---

## 3. Sơ Đồ Luồng Dữ Liệu (Data Flow)

Mô tả chi tiết 6 bước xử lý từ khi camera thu nhận frame ảnh cho đến khi kích hoạt cảnh báo và đồng bộ nhật ký:

```mermaid
sequenceDiagram
    autonumber
    actor Driver as 👤 Tài Xế
    participant Cam as 📷 CameraX Stream
    participant Pre as 🛠️ Tiền Xử Lý
    participant MP as 🧠 MediaPipe Mesh
    participant Ext as 📐 Trích Xuất EAR/MAR
    participant Engine as ⚖️ Decision Engine
    participant UI as 🚨 Loa / Màn Hình
    participant Cloud as ☁️ Backend & Phoenix Log

    Note over Cam: 1. CAPTURE
    Cam->>Pre: Frame ảnh (RGB, 30 FPS)
    
    Note over Pre: 2. PRE-PROCESS
    Pre->>Pre: Kiểm tra độ sáng (Luma >= 35), Chuẩn hóa MPImage
    Pre->>MP: Đưa frame vào pipeline suy luận
    
    Note over MP: 3. DETECTION
    MP->>Ext: 468+ Face Landmarks (Mắt 33, 133, 160, 144... Miệng 61, 291...)

    Note over Ext: 4. CALCULATION
    Ext->>Ext: Tính EAR_Left, EAR_Right -> Average EAR
    Ext->>Ext: Tính MAR (Độ mở miệng) & Head Pitch (Độ nghiêng đầu)
    Ext->>Engine: Gửi gói chỉ số FeatureSample(t, EAR, MAR, Pitch)

    Note over Engine: 5. DECISION
    rect rgb(255, 240, 240)
        Engine->>Engine: Kiểm tra ngưỡng liên tục (Continuous Window):<br/>• EAR < 0.18 trong >= 1.5s -> Cảnh báo ngủ gật<br/>• MAR > 0.65 trong >= 2.0s -> Cảnh báo ngáp mệt mỏi<br/>• Pitch < -20° trong >= 1.0s -> Cảnh báo gục đầu
    end

    Note over UI, Cloud: 6. ACTION & LOGGING
    alt Phát Hiện Buồn Ngủ / Mất Tập Trung (Triggered)
        Engine->>UI: Kích hoạt Buzzer/Audio âm lượng cao + Viền đỏ UI
        Engine->>Cloud: Đẩy Safety Alert Event qua MQTT & Upload Clip 5s
        Engine->>Cloud: Ghi log phiên vào AI Log (Phoenix Logger)
    else Trạng Thái Bình Thường (Normal)
        Engine->>UI: Hiển thị trạng thái an toàn (Xanh lá)
    end
```

---

## 4. Bảng Chi Tiết Từng Khối Chức Năng (Component Details)

| Module | Công nghệ / Thư viện | Nhiệm vụ chính trong code |
| :--- | :--- | :--- |
| **Input Module** | Android CameraX / Camera2 / Video File | Cung cấp luồng khung hình độ trễ thấp (30 FPS), tự động điều chỉnh tiêu cự và phơi sáng. |
| **Processing Module** | Google MediaPipe Vision Tasks (`face_landmarker.task`), BitmapImageBuilder | Phát hiện vùng mặt, trích xuất 468 tọa độ điểm 3D (Landmarks) trong thời gian thực (< 25ms/frame trên mobile). |
| **AI Logic / Feature Extractor** | `FaceFeatureExtractor.java`, `RiskEngine.java` | • **EAR (Eye Aspect Ratio):** Tính toán độ mở mi mắt qua khoảng cách Euclidean.<br/>• **MAR (Mouth Aspect Ratio):** Tính tỷ lệ mở miệng theo chiều dọc/ngang.<br/>• **Head Pose:** Trích xuất góc nghiêng (Pitch) qua ma trận biến đổi không gian. |
| **Decision Module** | `DmsDecisionEngine.java` | Áp dụng cửa sổ trượt thời gian (Sliding Window / PERCLOS_3s) để lọc bỏ chớp mắt tự nhiên (~200ms) và chỉ báo động khi nhắm mắt kéo dài. |
| **Output Module** | `FaceOverlayView.java`, Android SoundPool / MediaPlayer | Phát tín hiệu âm thanh tần số cao (Buzzer/Tone), rung phản hồi và vẽ bounding box trực quan cho tài xế. |
| **Cloud & Logging** | Paho MQTT v5, FastAPI, SQLite Outbox, Phoenix Hook | Đảm bảo tính tin cậy (QoS 1, lưu SQLite khi mất mạng) và gửi toàn bộ dữ liệu phiên chạy về hệ sinh thái chấm điểm Phoenix. |

---

## 5. Bảng Tham Số Ngưỡng Nghiệp Vụ (Safety Thresholds)

Các thông số này được cấu hình tập trung tại backend và đồng bộ xuống ứng dụng:

```json
{
  "THRESHOLD_EAR_DROWSY": 0.18,
  "THRESHOLD_MAR_YAWN": 0.65,
  "THRESHOLD_HEAD_PITCH_NOD": -20.0,
  "THRESHOLD_PERCLOS_3S": 0.40,
  "SPEED_EXPRESSWAY_MIN": 70.0,
  "S3_VIDEO_TTL_HOURS": 24
}
```

* **Mắt nhắm (Drowsiness):** $\text{EAR} < 0.18$ trong hơn $1.5$ giây hoặc $\text{PERCLOS} \ge 40\%$ trong cửa sổ 3 giây.
* **Ngáp (Yawn):** $\text{MAR} > 0.65$ duy trì hơn $2.0$ giây.
* **Gục đầu (Head Nod):** $\text{Pitch} < -20^\circ$ (đầu chúi xuống vô lăng).
