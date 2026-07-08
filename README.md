# Smart Interactive Virtual Guide | AI Museum Robot

An AI-Powered, cost-effective Museum Guide Robot designed to provide real-time exhibit identification, remote operator control, and interactive verbal/visual guidance to museum visitors. Developed as a final-year B.E. Capstone Project under the Department of Electronics & Telecommunication Engineering, Sipna College of Engineering and Technology.

---

## 📌 Project Overview

This project presents an **AI-powered museum guide robot** built with a total Bill of Materials (BOM) cost of **under $30 USD** (contrasting with commercial alternatives ranging from $10,000 to $30,000 USD). By utilizing cloud-based Artificial Intelligence (Groq and ElevenLabs APIs) instead of expensive local GPU hardware, the robot achieves high accuracy in computer vision and voice recognition with sub-10-second latencies.

### 🌟 Key Highlights
* **Budget-Friendly:** Full physical prototype built under $30.
* **Separation of Concerns:** Dual ESP32 microcontroller design for efficient task distribution.
* **Cloud AI Vision:** Real-time visual scene analysis using the Groq API (LLaMA-4 Vision model).
* **Voice Interaction:** Speech-to-Text transcription powered by ElevenLabs Scribe API.
* **Multimodal Feedback:** Narration via digital audio and scrolling visual description on an OLED screen.
* **Locomotion Control:** operator dashboard with a custom watchdog timer for locomotion safety.

---

## ⚙️ System Architecture & Working Principle

To handle the memory and processing constraints of low-cost microcontrollers, the system splits visual/mobility workloads from audio/display tasks using two independent microcontrollers connected over a physical UART link.

```
       +-------------------+               +----------------------+
       |     ESP32-CAM     |  UART Link    |   ESP32 DevKit V1    |
       | (Vision & Drive)  |<------------->|   (Voice & Output)   |
       +---------+---------+  115,200 baud +----------+-----------+
                 |                                    |
     +-----------+-----------+            +-----------+-----------+
     |                       |            |                       |
+----+----+             +----+----+  +----+----+             +----+----+
| OV2640  |             |  L298N  |  | INMP441 |             |SH1106   |
| Camera  |             | Driver  |  | I2S Mic |             |OLED Disp|
+---------+             +---------+  +---------+             +---------+
                                              |                   |
                                         +----+----+         +----+----+
                                         |MAX98357 |         |LittleFS |
                                         | I2S DAC |         |Storage  |
                                         +----+----+         +---------+
                                              |
                                         +----+----+
                                         | Speaker |
                                         +---------+
```

### 1. 📷 ESP32-CAM Subsystem (Vision & Locomotion)
The ESP32-CAM manages high-frequency visual streaming and motor control.
* **MJPEG Video Server:** Streams live video feed over Port 81 using direct DMA buffer reading.
* **Cloud Visual Uplink:** Pauses the live stream, captures a frame at QVGA resolution (320x240), serializes the binary JPEG into Base64, and streams it to the Groq Vision endpoint in 2,048-byte chunks to prevent heap overflow.
* **Locomotion Control:** Hosts an operator web panel on Port 80, driving an H-bridge L298N motor driver using PWM signals.
* **Watchdog Protection:** Employs a heartbeat mechanism. If the operator's movement control packet is not received for over 500ms, the watchdog timer halts all motors immediately to prevent physical collision.
* **UART Configuration:** Communicates with the audio board via SoftwareSerial (TX: GPIO 12, RX: GPIO 13) at 115,200 baud.

### 2. 🎤 ESP32 DevKit V1 Subsystem (Voice & Multimodal Output)
The ESP32 DevKit V1 handles direct audio processing and visual OLED displays.
* **Voice Activity Detection (VAD):** Samples audio from the INMP441 MEMS microphone over the I2S protocol (16 kHz, 24-bit). The microcontroller continuously calculates the Root Mean Square (RMS) energy. When the signal exceeds 45 dB SPL for 8 consecutive frames (512 samples each), voice recording is triggered.
* **Speech-to-Text (STT):** Uploads recorded binary audio clips (approx. 5 seconds) to ElevenLabs Scribe over secure HTTPS.
* **Narrator playback:** Decodes pre-recorded MP3 files from the local flash memory (via LittleFS filesystem) and plays them back through the MAX98357A I2S DAC Amplifier.
* **GUI scroll display:** Drives the 1.3-inch SH1106 OLED screen over Hardware I2C (SDA: GPIO 21, SCL: GPIO 22) to render scrolling textual summaries.

---

## 🔌 Hardware Configurations & Pin Mapping

### ESP32-CAM Pin Connections
| Peripheral | Component Pin | ESP32-CAM GPIO | Description |
| :--- | :--- | :--- | :--- |
| **L298N Driver** | IN1 | GPIO 12 | Motor Left Direction A |
| | IN2 | GPIO 13 | Motor Left Direction B |
| | IN3 | GPIO 15 | Motor Right Direction A |
| | IN4 | GPIO 14 | Motor Right Direction B |
| **Serial Link** | RX | GPIO 13 | Software UART Receive |
| | TX | GPIO 12 | Software UART Transmit |

### ESP32 DevKit V1 Pin Connections
| Peripheral | Component Pin | ESP32 DevKit V1 GPIO | Description |
| :--- | :--- | :--- | :--- |
| **INMP441 Microphone**| SCK | GPIO 26 | I2S Serial Clock |
| | WS | GPIO 25 | I2S Word Select / LRC |
| | SD | GPIO 33 | I2S Serial Data |
| **MAX98357A Amp** | BCLK | GPIO 14 | I2S Bit Clock |
| | LRC | GPIO 27 | I2S Left/Right Clock |
| | DIN | GPIO 12 | I2S Data Input |
| **SH1106 OLED** | SDA | GPIO 21 | Hardware I2C Data |
| | SCL | GPIO 22 | Hardware I2C Clock |
| **Serial Link** | RX2 | GPIO 16 | Hardware UART2 Receive |
| | TX2 | GPIO 17 | Hardware UART2 Transmit |

---

## 🛠️ Operational Pipeline

```
[OV2640 Capture] --> [Base64 Chunks] --> [Groq Vision LLaMA-4] 
                                                  |
                                                  v
[I2S Audio Output] <-- [UART Play Command] <-- [Exhibit Match]
        |
        +--> [MAX98357A Narration] & [SH1106 Scrolling Description]
```

1. **Scene Acquisition:** The operator drives the robot close to an exhibit. The camera subsystem captures the scene.
2. **AI Inference:** The image is sent to the Groq API. LLaMA-4 Vision processes context, shapes, and colors.
3. **Keyword Parsing:** The textual response from Groq is parsed for target tags (e.g., "Mona Lisa", "Ganesha").
4. **Trigger Signal:** Once a matching exhibit is identified, a UART frame (e.g., `PLAY_GANESHA\n`) is transmitted.
5. **Guidance Experience:** The audio board catches the UART string, fetches the matching MP3 narrative from LittleFS, outputs it to the I2S speaker, and scrolls the metadata text across the SH1106 OLED screen.

---

## 📈 Technical Analysis

### Advantages (Pros)
* **Extremely Low Cost:** Complete system BOM cost under $30 USD.
* **Practical AI Integration:** Leverages cloud API computation, eliminating the need for expensive edge GPUs.
* **High Reliability:** Reached ~90% accuracy in speech transcription and ~85% accuracy in vision parsing.
* **Locomotion Watchdog:** Hardware halts immediately on packet drop to prevent structural damage.
* **Low Idle Draw:** Idle consumption of ~140mA allows extended operation on simple Li-ion battery arrays.

### Limitations (Cons)
* **Internet Dependency:** Offline operation is currently not supported; requires active Wi-Fi.
* **Response Latency:** Cloud roundtrips lead to a 5-10 second response delay for visual/voice commands.
* **Manual Positioning:** Lacks autonomous path planning or automatic LIDAR-based obstacle avoidance.
* **Single Language:** Speech interaction is currently limited to English keywords.

---

## 🚀 Future Scopes
* **Autonomous Navigation:** Integration of ultrasonic sensors or lightweight LIDAR for self-guided tours.
* **Dynamic Text-to-Speech (TTS):** Real-time generation of custom voice answers utilizing ElevenLabs API to answer arbitrary queries.
* **Edge AI Deployment:** Transitioning to offline visual recognition using TensorFlow Lite Micro (YOLOv8-nano) on the ESP32.
* **Content Management System (CMS):** Connect the LittleFS narrative files to a centralized database server for curator dashboard updates.
* **Multilingual support:** Support local regional Indian languages such as Hindi, Marathi, etc.

---

## 👥 Project Team

* **Department:** Electronics & Telecommunication Engineering (EXTC) Capstone Project 2025-26
* **Institution:** Sipna College of Engineering and Technology, Amravati, Maharashtra, India
* **Project Guide:** Dr. A.V.Malviya

**Development Team:**
* **Ram Kaitwas** (Team Leader) · GitHub: [@Ramjianonmyous](https://github.com/Ramjianonmyous)
* **Shrihit Bandawar**
* **Snehal Kalkar**
* **Sampada Dumas**
* **Sahil Palaskar**
