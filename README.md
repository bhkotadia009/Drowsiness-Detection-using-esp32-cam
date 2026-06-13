# Drowsiness-Detection-using-esp32-cam
# Driver Drowsiness Detection System

A real-time drowsiness detection system using an **ESP32-CAM** for video streaming and a **Python/dlib** pipeline for Eye Aspect Ratio (EAR) based detection. When drowsiness is detected, an alert is sent over Wi-Fi to trigger a buzzer on the ESP32.

---

## How It Works

```
ESP32-CAM  ──(MJPEG stream)──►  Python (EAR detection)  ──(HTTP GET /alert)──►  ESP32 buzzer
```

1. ESP32-CAM streams MJPEG video over Wi-Fi on port 81.
2. Python reads the stream, detects faces using dlib, and computes EAR per frame.
3. If EAR drops below threshold for N consecutive frames, an alert request is fired to the ESP32.
4. ESP32 activates the buzzer for 2 seconds.

---

## Hardware Requirements

| Component | Details |
|-----------|---------|
| ESP32-CAM | AI-Thinker module |
| FTDI/USB-TTL programmer | For flashing (3.3 V logic) |
| Buzzer | Active buzzer on GPIO 12 |
| Power supply | 5 V / 2 A recommended |

**Wiring (buzzer):**
```
ESP32 GPIO12  →  Buzzer (+)
ESP32 GND     →  Buzzer (-)
```

---

## Project Structure

```
.
├── esp32_cam/
│   └── esp32_cam.ino        # Arduino sketch for ESP32-CAM
├── detection/
│   └── detect.py            # Python drowsiness detection client
├── shape_predictor_68_face_landmarks.dat   # dlib landmark model (download separately)
└── README.md
```

---

## ESP32-CAM Setup

### Prerequisites
- Arduino IDE with **ESP32 board package** installed
- Board: `AI Thinker ESP32-CAM`

### Configuration

In `esp32_cam.ino`, set your Wi-Fi credentials:
```cpp
const char* ssid     = "YOUR_WIFI_NAME";
const char* password = "YOUR_WIFI_PASSWORD";
```

Default GPIO for buzzer is `12`. Change `BUZZER_PIN` if needed.

### Flash Instructions

1. Wire FTDI programmer to ESP32-CAM (TX→RX0, RX→TX0, GND→GND, 5V→5V).
2. Pull **GPIO0 to GND** before powering on to enter flash mode.
3. Upload via Arduino IDE.
4. Remove GPIO0-GND jumper, press reset.
5. Open Serial Monitor at **115200 baud** — note the printed IP address.

### Endpoints

| Endpoint | Port | Description |
|----------|------|-------------|
| `GET /video` | 81 | MJPEG stream |
| `GET /alert` | 80 | Triggers buzzer for 2 s |

---

## Python Client Setup

### Prerequisites

- Python 3.7+
- CMake (required to build dlib)

### Install Dependencies

```bash
pip install opencv-python imutils scipy dlib requests
```

> **Note:** `dlib` requires CMake. On Ubuntu: `sudo apt install cmake`. On Windows, install [CMake](https://cmake.org/download/) and Visual Studio Build Tools.

### Download Landmark Model

```bash
wget http://dlib.net/files/shape_predictor_68_face_landmarks.dat.bz2
bzip2 -d shape_predictor_68_face_landmarks.dat.bz2
```

Or download manually from [dlib.net](http://dlib.net/files/shape_predictor_68_face_landmarks.dat.bz2) and extract.

### Configuration

In `detect.py`, set the ESP32 IP (printed on Serial Monitor after boot):
```python
ESP32_IP = "192.168.x.x"
```

Tunable parameters:
```python
EAR_THRESH  = 0.25   # EAR below this = eye closed (lower = less sensitive)
FRAME_CHECK = 6      # Consecutive closed-eye frames to trigger alert
```

### Run

```bash
python detect.py
```

Press `q` to quit the detection window.

---

## EAR — Eye Aspect Ratio

EAR is computed from 6 eye landmark points:

```
EAR = (||p2-p6|| + ||p3-p5||) / (2 * ||p1-p4||)
```

- Open eye → EAR ≈ 0.3
- Closed eye → EAR ≈ 0.0–0.2
- Threshold of `0.25` works for most lighting conditions; tune as needed.

---

## Tuning Tips

| Issue | Fix |
|-------|-----|
| Too many false alerts | Increase `EAR_THRESH` slightly (e.g. 0.27) or increase `FRAME_CHECK` |
| Misses drowsiness | Decrease `EAR_THRESH` or decrease `FRAME_CHECK` |
| Low FPS / lag | Set `FRAMESIZE_QVGA` (already default); reduce `jpeg_quality` to 15–20 |
| Alert spamming | Increase `alert_cooldown` frames in `detect.py` (default: 30) |
| Stream drops | Check Wi-Fi signal; ESP32-CAM is 2.4 GHz only |

---

## Known Limitations

- Detection runs on the host machine — ESP32 only streams and actuates.
- EAR-based detection is sensitive to head angle; works best with a front-facing camera.
- `alert_handler` on ESP32 uses `delay(2000)`, which blocks the HTTP task during buzzer activation. Long alert durations can cause the stream server to miss frames.
- No HTTPS; all communication is plain HTTP on local network.

---

## License

MIT
