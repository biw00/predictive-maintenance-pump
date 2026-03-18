# Vibration Analysis & ISO 10816-3 Trend Monitoring

> **Linear Regression (FFT Features)** — พยากรณ์และจำแนก Zone ความเสียหายของเครื่องจักรหมุน ตามมาตรฐาน ISO 10816-3

---

## ภาพรวมโปรเจกต์

โปรเจกต์นี้วิเคราะห์สัญญาณการสั่นสะเทือน (Vibration) ของเครื่องจักรหมุน โดยแปลงข้อมูล Acceleration (G-s) จากไฟล์ .txt ให้กลายเป็นค่า Velocity RMS (mm/s) ผ่านกระบวนการ FFT Integration จากนั้นนำไปจำแนก Zone ตามมาตรฐาน ISO 10816-3 และพยากรณ์แนวโน้มในอนาคต

**เครื่องจักรที่วิเคราะห์:**

| เครื่องจักร | รหัส | ความเร็วรอบ |
|---|---|---|
| Motor Compressor | CH-06 A (NAA_1490) | ~1,490 RPM |
| Cooling Pump | OAH-02 (M1H_1480) | ~1,480 RPM |
| Jockey Pump | M1A (2925) | ~2,925 RPM |

**ช่วงเวลาข้อมูล:** มิถุนายน / กันยายน / ตุลาคม 2024

---

## โครงสร้างไฟล์

```
📁 project-folder/
├── 📁 dataset/
│   ├── A_CH-06 A_NAA_1490__Jun24.txt
│   ├── A_CH-06 A_NAA_1490__Sep24.txt
│   ├── A_CH-06 A_NAA_1490__Oct24.txt
│   ├── A_Cooling Pump OAH 02_M1H_1480_Jun24.txt
│   ├── A_Cooling Pump OAH 02_M1H_1480_Sep24.txt
│   ├── A_Cooling Pump OAH 02_M1H_1480_Oct24.txt
│   ├── A_Jockey pump_M1A_2925__Jun24.txt
│   ├── A_Jockey pump_M1A_2925__Sep24.txt
│   └── A_Jockey pump_M1A_2925__Oct24.txt
├── 📁 saved_models/      ← สร้างอัตโนมัติเมื่อรัน
├── 📁 saved_plots/       ← สร้างอัตโนมัติเมื่อรัน
└── vibration_iso10816.ipynb
```

---

## การติดตั้ง

**Python 3.11+** และติดตั้ง dependencies ดังนี้:

```bash
pip install numpy pandas matplotlib scikit-learn joblib scipy
```

---

## Pipeline

```
Raw .txt (Acceleration G-s)
    │
    ├── Step 1 : Import Libraries & กำหนด Folder
    │
    ├── Step 2 : parse_vibration_file()
    │             → DataFrame [Time_ms, Amplitude]
    │             (DC offset removal + dup timestamp check)
    │
    ├── Step 3 : extract_fft_features()  (windowed, 256 samples, 50% overlap)
    │             X = [peak_freq_hz, accel_rms,
    │                  band_low/mid/high_power, spectral_centroid,
    │                  kurtosis, crest_factor]
    │             y = velocity_rms (mm/s)  ← scipy integration per window
    │
    ├── Step 4 : train_model()
    │             Group-based split: train=Jun+Sep / test=Oct
    │             CV: GroupKFold (ไม่มี data leakage)
    │             Best model: Linear Regression (Ridge)
    │             Best CV R²: 0.8406
    │             → บันทึก *_fft.pkl
    │
    ├── Step 4.5 : predict_rms_from_gb()
    │             Gradient Boosting — ใช้กับไฟล์ใหม่ในอนาคต
    │
    ├── Step 5 : accel_to_velocity_rms_fft()  (full signal)
    │             → velocity_rms ค่าเดียวต่อไฟล์ (mm/s)
    │
    ├── Step 6 : classify_zone()
    │             → ISO 10816-3 Zone A / B / C / D
    │
    ├── Step 7 : Trend Analysis Plots
    │             Bar Chart + Grouped Bar + Line Chart
    │
    ├── Step 8 : สรุปผล + คำแนะนำตาม Zone
    │
    └── Step 9 : Hybrid Trend Forecast → Jan 2025
                  X = month_order (0, 1, 2, ...)
                  y = velocity_rms จาก Step 5 (FFT historical)
                      + GB predictions (new months)
                  แสดง 95% Prediction Interval บนกราฟ
```

---

## ISO 10816-3 Zone Classification

| Zone | Velocity RMS (mm/s) | ความหมาย | การดำเนินการ |
|:---:|---|---|---|
| **A** | ≤ 2.3 | ใหม่หรือสมบูรณ์มาก | ปกติ |
| **B** | 2.3 – 4.5 | ยอมรับได้สำหรับการใช้งานระยะยาว | เฝ้าติดตาม |
| **C** | 4.5 – 7.1 | น่าเป็นห่วง | วางแผนซ่อมบำรุง |
| **D** | > 7.1 | อันตราย | หยุดเครื่องทันที |

---

## Features ที่ใช้ในโมเดล

| Feature | สูตร | ความหมาย |
|---|---|---|
| `peak_freq_hz` | argmax(power[f > 2 Hz]) | ความถี่ที่มีพลังงานสูงสุด |
| `accel_rms` | √mean(x²) | ขนาดการสั่นสะเทือนรวม |
| `band_low_power` | Σpower[2–10 Hz] | ย่านความถี่ต่ำ (imbalance) |
| `band_mid_power` | Σpower[10–100 Hz] | ย่านความถี่กลาง (misalignment) |
| `band_high_power` | Σpower[>100 Hz] | ย่านความถี่สูง (bearing fault) |
| `spectral_centroid` | Σ(f×power)/Σpower | จุดศูนย์ถ่วงของ spectrum |
| `kurtosis` | scipy.stats.kurtosis | ตรวจจับ impulse / impact |
| `crest_factor` | peak / RMS | ความแหลมของสัญญาณ |

---

## ผลลัพธ์โมเดล

| รายการ | ค่า |
|---|---|
| Split strategy | Group-based (train = Jun+Sep, test = Oct) |
| Cross-validation | GroupKFold (ไม่มี data leakage) |
| Window size | 256 samples, hop 128 (overlap 50%) |
| Best model | Linear Regression (Ridge) |
| Best CV R² | **0.8406** |
| Forecast target | Jan 2025 |
| Uncertainty | 95% Prediction Interval |

---

## กราฟที่สร้างขึ้น (`saved_plots/`)

| ไฟล์ | เนื้อหา |
|---|---|
| `regression_actual_vs_pred.png` | Actual vs Predicted scatter พร้อม ISO boundaries |
| `gb_vs_fft_validation.png` | เปรียบเทียบ GB prediction กับ FFT integration |
| `iso_zone_all_files.png` | Zone จำแนกตามไฟล์ทั้งหมด |
| `iso_trend_by_machine_month.png` | Grouped bar chart ตามเครื่องจักรและเดือน |
| `iso_trend_line.png` | Line chart แนวโน้ม velocity RMS |
| `iso_hybrid_forecast.png` | Hybrid forecast พร้อม 95% PI ถึง Jan 2025 |

---

## หมายเหตุทางเทคนิค

**ทำไมต้องใช้ FFT Integration แทน cumsum?**
ข้อมูลแต่ละเดือนมี sampling rate ต่างกัน (Jun/Oct: 7,680 Hz, Sep: 2,560 Hz) ดังนั้น FFT integration จึงถูกต้องกว่าในเชิงฟิสิกส์เพราะรองรับ `dt` ที่ไม่สม่ำเสมอได้

**Hybrid Forecast (Step 9)**
Trend LR ใช้ FFT ทางอ้อม — ค่า `y` ที่ใช้ fit คือ velocity RMS จาก FFT integration (Step 5) แต่ตัว Trend LR เองรับแค่ `month_order` → `velocity_rms` โดยไม่เห็น spectrum โดยตรง เมื่อมีข้อมูลใหม่ สามารถเพิ่ม GB predicted point เข้า timeline ได้โดยไม่ต้องมีไฟล์ .txt ใหม่

---

## การอัปเดตในเวอร์ชันปัจจุบัน

- เพิ่ม regex parser + DC offset removal + duplicate timestamp check (Step 2)
- เพิ่ม Hann window + Kurtosis + Crest Factor เป็น features (Step 3)
- เปลี่ยนเป็น `build_windowed_dataset` (model เดียวทุกเครื่อง) (Step 4)
- Group-based split + GroupKFold CV (Step 4)
- ใช้ `month_order` เป็น x-axis ในกราฟ Trend (Step 7)
- เพิ่ม 95% Prediction Interval บน Forecast chart (Step 9)
- รองรับ Hybrid timeline: FFT (historical) + GB (new months) (Step 9)

---

## มาตรฐานอ้างอิง

**ISO 10816-3** — *Mechanical vibration — Evaluation of machine vibration by measurements on non-rotating parts — Part 3: Industrial machines with nominal power above 15 kW and nominal speeds between 120 r/min and 15,000 r/min when measured in situ*
