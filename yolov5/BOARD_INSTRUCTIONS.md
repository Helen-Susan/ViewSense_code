# Board Client — Setup & Run Instructions

This document explains how to set up and run `board_client.py` on the **Radxa board**. This script captures camera frames, streams them to the AI server over WebSocket, plays back the LLM's response using Piper TTS, and handles a physical GPIO button for switching modes (Object Detection / Currency / OCR).

---

## 1. Hardware Requirements

- Radxa board (with `periphery` GPIO support)
- USB camera connected and recognized as `/dev/video0`
- A physical push-button wired to GPIO chip `/dev/gpiochip0`, line `108` (PD12)
- Speaker/audio output connected (for TTS playback via `aplay`)

---

## 2. Software Prerequisites

### 2.1 System packages

Install audio playback and ALSA utilities (provides `aplay`):
```bash
sudo apt update
sudo apt install -y alsa-utils
```

Verify your speaker is detected:
```bash
aplay -l
```

### 2.2 Python environment

Create and activate a virtual environment (the script expects one at `./ven`, based on `PIPER_EXE` path):
```bash
cd /path/to/main-project-sw
python3 -m venv ven
source ven/bin/activate
```

### 2.3 Python dependencies

Install required packages:
```bash
pip install opencv-python websockets numpy python-periphery coloredlogs
```

> **Note:** `python-periphery` is the PyPI package name for the `periphery` module used for GPIO access — make sure to install `python-periphery`, not `periphery`.

### 2.4 Piper TTS

Install Piper inside the virtual environment (or ensure the binary is placed at `ven/bin/piper`):
```bash
pip install piper-tts
```
Or, if using the standalone Piper binary, place/symlink it at:
```
main-project-sw/ven/bin/piper
```

### 2.5 Piper voice model

Download the voice model referenced in the script (`en_GB-alba-medium.onnx`) and its accompanying `.json` config file, and place them at:
```
main-project-sw/backend/board/tts/models/en_GB-alba-medium.onnx
main-project-sw/backend/board/tts/models/en_GB-alba-medium.onnx.json
```
Voice models can be downloaded from the [Piper voices repository](https://github.com/rhasspy/piper/blob/master/VOICES.md).

---

## 3. Configuration

Open `board_client.py` and update the following constants to match your setup:

| Variable | Description | Example |
|---|---|---|
| `LAPTOP_IP` | IP address of the machine running `server.py` | `"172.20.10.2"` |
| `CAMERA_INDEX` | Path to the USB camera device | `"/dev/video0"` |
| `GPIO_CHIP` | GPIO chip device path | `"/dev/gpiochip0"` |
| `GPIO_LINE` | GPIO line number for the mode-switch button | `108` |

**Important:** `LAPTOP_IP` must be the **actual LAN IP address** of the server machine (not `localhost`), since the board and server are separate devices. Both devices must be on the same network.
--dev_video0 is the usual webcam path
To find the server machine's IP:
- **Windows:** `ipconfig` → look for IPv4 Address
- **Linux:** `ip addr show` or `hostname -I`

---

## 4. Running the Server First

Before starting the board client, the AI server (`server.py`) must already be running and reachable at the configured `LAPTOP_IP:8000`. See the server's own setup instructions to start it (typically `python server.py` from the `backend/` folder).

Verify connectivity from the Radxa board before running the full client:
```bash
ping <LAPTOP_IP>
curl http://<LAPTOP_IP>:8000
```

---

## 5. Running the Board Client

From the `main-project-sw` root (or wherever `board_client.py` resides), with the virtual environment activated:
```bash
source ven/bin/activate
python board_client.py
```

### What happens on startup:
1. Logging is initialized — logs print to console and are saved to `logs/board/board.log` (rotating, max 5MB × 3 backups).
2. The Piper TTS worker starts and "warms up" the ONNX engine (~2 seconds) so the first real response has no extra delay.
3. The USB camera is opened at `/dev/video0`.
4. The GPIO button is initialized for mode switching.
5. The client connects to the server's `/vision` WebSocket endpoint.
6. The main loop begins: capturing frames, sending them to the server, and playing back any streamed LLM text response via TTS.

### Controls:
- **Physical button press:** cycles through modes — `ObjectDetection → Currency → OCR → ObjectDetection...`
- **`q` key** (only works if a display/GUI is attached): closes the local preview window and exits the loop.
- **Ctrl+C:** gracefully stops the client (camera released, GPIO closed, TTS queue drained).

---

## 6. Headless Operation Notes

If running without a monitor attached to the Radxa board:
- The `cv2.imshow("Board Feed (Real-time)", frame)` call will fail to open a window. This is already caught by a `try/except` in the code, so **it will not crash the program** — it logs a one-time warning and continues in headless mode automatically.
- You do not need to modify the code for headless operation; just be aware the live preview simply won't be visible.

---

## 7. Logs

All runtime logs are written to:
```
main-project-sw/logs/board/board.log
```
Logs include frame transmission status, LLM response chunks received, TTS sync timing, and any connection or GPIO errors. Use these logs for debugging timing or connectivity issues between board and server.

---

## 8. Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `Could not open camera at /dev/video0` | Camera not connected, wrong device path, or permissions issue | Run `ls /dev/video*` to confirm the device path; add user to `video` group: `sudo usermod -aG video $USER` (then re-login) |
| `Error initializing GPIO` | Wrong chip/line number, or permissions issue | Confirm with `gpioinfo` which chip/line maps to your button; add user to `gpio` group if needed |
|check whether gpio is enabled in the board cmd:sudo rsetup
| `Connection Error` on startup | Server not running, wrong `LAPTOP_IP`, or firewall blocking port 8000 | Confirm server is running and reachable via `curl http://<LAPTOP_IP>:8000`; check firewall rules on server machine |
| No audio output | `aplay` misconfigured, wrong sound card, or Piper model/JSON missing | Test manually: `echo "hello" | piper --model <model_path> --output-raw | aplay -r 22050 -f S16_LE -t raw -` |
| TTS never starts speaking (`SIGNAL_READY` timeout in logs) | Piper failed to start or model path incorrect | Verify `PIPER_EXE` and `PIPER_MODEL` paths exist and are executable |

---

## 9. Process Flow Summary

```
Camera → Encode JPEG → Send over WebSocket (with mode byte)
                              ↓
                     Server runs AI detection + LLM
                              ↓
        Streamed LLM text chunks ← WebSocket ← Server
                              ↓
                  Queued into Piper TTS → aplay → Speaker
                              ↓
            "ready" signal sent back to server after speech finishes
```