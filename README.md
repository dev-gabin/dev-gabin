# Retrace

> 스마트 공간 블랙박스 기반 분실물 위치 기억 및 탐색 시스템

Retrace는 카메라를 이용해 공간 속 물건을 지속적으로 관찰하고,  
물건이 마지막으로 확인된 **위치·시간·이미지**를 기록하여 분실 시 빠르게 찾을 수 있도록 돕는 IoT 시스템입니다.

단순히 현재 물건을 탐지하는 것이 아니라, 물건이 시야에서 사라지더라도  
**마지막 목격 정보(Last Seen)** 를 기억하고 복원하는 것을 핵심 목표로 합니다.

**허브(Jetson Nano) + 노드** 구조로, 카메라 기반 비주얼 메모리를 중심으로  
**레이저 안내**, **스마트 서랍**, **스마트 현관등**, **BLE 부저**, **Web/PWA**를 연동합니다.

---

## 시스템 구성

| 노드 | 장치 | 역할 |
|---|---|---|
| **메인 유닛 (허브)** | Jetson Nano + 카메라 + STM32 | AI 객체 탐지, Last Seen, DB/이미지 저장, Web 서버, MQTT 브로커, 레이저 안내, PIR |
| **서랍 노드** | Arduino Uno + HC-06 | 서랍 LED ×6, 서랍 팝업 서보 ×6, 비상 버튼 |
| **출입 노드 (스마트 현관등)** | LOLIN D32 (ESP32) + PIR + LED | 외출 감지, 현관등 점등·경고 |
| **부저 태그** | ESP32-C3 + 부저 | 중요 물건 부저 호출 |
| **사용자 화면** | Web / PWA | 검색 및 시스템 제어 UI |

---

## 사용 기술

- Jetson Nano / Linux
- C / C++
- YOLO / OpenCV
- Arduino Uno
- STM32 NUCLEO-F411RE (STM32CubeMX / CMake)
- ESP32 (LOLIN D32, ESP32-C3)
- Wi-Fi / MQTT (Mosquitto)
- ntfy (폰 푸시 알림)
- Bluetooth (HC-06) / BLE
- GitHub / Jira

---

## 시스템 구조

```text
┌────────────── Main Unit (Hub) ──────────────┐
│                                             │
│  [ USB Camera ]                             │
│        │ USB                                │
│        v                                    │
│  [ Jetson Nano ]   Vision / Last Seen / DB  │
│        │           Web Server / MQTT Broker │
│        │ USB Serial                         │
│        v                                    │
│  [ STM32 Laser Head ]                       │
│     Pan/Tilt / Laser / PIR                  │
│                                             │
└────┬───────────┬───────────┬───────────┬────┘
     │           │           │           │
 Bluetooth     Wi-Fi        BLE        Wi-Fi
  (HC-06)     (MQTT)                  (HTTP)
     │           │           │           │
┌────┴────┐ ┌────┴────┐ ┌────┴────┐ ┌────┴────┐
│  Drawer │ │ Entrance│ │  Buzzer │ │ Web/PWA │
│   Node  │ │   Node  │ │   Tag   │ │         │
│ Arduino │ │LOLIN D32│ │ ESP32-C3│ │ Phone/PC│
│         │ │         │ │         │ │         │
│  LED x6 │ │   PIR   │ │  Buzzer │ │         │
│ Servo x6│ │   LED   │ │         │ │         │
│ SOS Btn │ │         │ │         │ │         │
└─────────┘ └─────────┘ └─────────┘ └─────────┘
```

| 노드 | 설명 |
|---|---|
| Main Unit (Hub) | Jetson Nano + 카메라 + STM32 레이저 헤드를 한 몸체로 구성 |
| Drawer Node | 스마트 서랍 + 비상 폰 찾기(사이렌) 버튼 (Arduino Uno + HC-06) |
| Entrance Node | 스마트 현관등 (LOLIN D32 + PIR + LED) |
| Buzzer Tag | BLE 부저 태그 (ESP32-C3) |
| Web/PWA | 사용자 화면 (스마트폰 / PC), 폰 사이렌은 ntfy 앱으로 수신 |

Jetson Nano가 허브로서 판단·기록·서비스를 담당하고,  
각 노드는 맡은 센서 및 구동 장치를 제어하며 허브와 통신합니다.

---

## 통신 구조

설치 위치와 용도에 따라 통신 방식을 나눴습니다.

| 연결 | 방식 | 선택 이유 |
|---|---|---|
| Jetson ↔ 카메라 | USB | 메인 유닛 내부 |
| Jetson ↔ STM32 | USB Serial (ST-LINK VCP) | 메인 유닛 내부, 카메라·레이저 위치 고정 |
| Jetson ↔ 서랍 노드 | **Bluetooth** (HC-06) | 같은 방 안 가구 — 근거리, 공유기 불필요 |
| Jetson ↔ 출입 노드 | **Wi-Fi** (MQTT) | 다른 공간(현관) — 공유기 경유 |
| Jetson ↔ 부저 태그 | **BLE** | 물건에 부착되는 저전력 장치 |
| Jetson ↔ Web / PWA | **Wi-Fi** (HTTP) | 사용자 화면 |
| Jetson → 스마트폰 알림 | **Wi-Fi** (ntfy) | 화면이 꺼진 폰도 받아야 하는 알림 (폰 사이렌, 외출 알림) |

> **같은 방 = Bluetooth / 다른 공간 = Wi-Fi / 물건에 붙는 장치 = BLE**

Jetson의 Bluetooth·BLE는 USB Bluetooth 4.0 동글(CSR8510)을 사용합니다.

### MQTT 토픽

Wi-Fi 노드는 Jetson의 MQTT 브로커(Mosquitto)를 통해 이벤트와 명령을 주고받습니다.  
새 노드는 Wi-Fi + MQTT 토픽 구독만으로 추가할 수 있습니다.

| 토픽 | 방향 | 내용 |
|---|---|---|
| `retrace/entrance/motion` | 출입 노드 → Jetson | 움직임 감지 |
| `retrace/entrance/light` | Jetson → 출입 노드 | `NORMAL` / `ALERT` |

> 메시지 세부 형식은 `docs/protocol.md`에서 확정 예정입니다.

---

## 하드웨어 역할

### 메인 유닛 – Jetson Nano

시스템의 허브(중앙 처리 장치)입니다.

담당 기능:

- 카메라 영상 입력
- YOLO 객체 탐지 / OpenCV 영상 처리
- Last Seen 판단 및 기록
- DB / 이미지 저장
- Web/PWA 서버
- MQTT 브로커
- ntfy 알림 서버 (폰 사이렌, 외출 알림)
- STM32 통신 (USB Serial)
- 서랍 노드 통신 (Bluetooth)
- 부저 태그 통신 (BLE)

---

### 메인 유닛 – STM32 NUCLEO-F411RE

카메라와 한 몸체로 고정된 **레이저 헤드**입니다.

담당 기능:

- Pan / Tilt 서보 제어
- 레이저 ON/OFF
- PIR 센서 입력
- Jetson과 통신 (USB Serial)

```text
STM32
├─ Pan Servo
├─ Tilt Servo
├─ Laser
└─ PIR
```

핀 배정은 [STM32 핀맵](docs/stm32_pinmap.md)을 참고합니다.

---

### 서랍 노드 – Arduino Uno

**스마트 서랍**을 담당합니다.

담당 기능:

- 서랍 LED ×6 제어
- 서랍 팝업 서보 ×6 제어
- 비상 버튼 입력
- Jetson과 통신 (HC-06 Bluetooth)

```text
Arduino
├─ Drawer LED ×6
├─ Drawer Popup Servo ×6
├─ Emergency Button
└─ HC-06 (Bluetooth)
```

> Arduino TX(5V) → HC-06 RX(3.3V) 사이에는 전압 분배 저항을 사용합니다.

---

### 출입 노드 – LOLIN D32 (ESP32)

**스마트 현관등**입니다.

담당 기능:

- PIR로 외출 감지
- 현관등 점등 (평소) / 경고 점등 (물건 두고 나갈 때)
- Jetson과 통신 (Wi-Fi, MQTT)

```text
LOLIN D32
├─ PIR
└─ LED (현관등)
```

---

### 부저 태그 – ESP32-C3

중요 물건에 부착하는 **BLE 부저 태그**입니다.

담당 기능:

- Jetson의 BLE 신호 수신
- 부저 울림

---

## 데이터 흐름

### 물건 감지

```text
Camera
   ↓
Jetson Nano (YOLO / OpenCV)
   ↓
객체 탐지
   ↓
Last Seen 갱신
   ↓
DB + 스냅샷 저장
```

### 레이저 위치 안내

```text
Web / PWA
   ↓ HTTP
Jetson Nano
   ↓ USB Serial
STM32
   ↓
Pan/Tilt 이동 → Laser ON
```

### 스마트 서랍

```text
Web / PWA
   ↓ HTTP
Jetson Nano
   ↓ Bluetooth
Arduino
   ↓
해당 서랍 LED ON → Drawer Servo 동작
   ↓
일정 시간 후 LED 자동 OFF
```

### PIR 활동 감지

```text
PIR
 ↓
STM32
 ↓ USB Serial
Jetson Nano
 ↓
분석 활성 / 대기 판단
```

### 비상 버튼

```text
Emergency Button
   ↓
Arduino
   ↓ Bluetooth
Jetson Nano (ntfy 서버)
   ↓ Wi-Fi
스마트폰 ntfy 앱
   ↓
폰 사이렌 (알림음 반복)
```

### 외출 알림 (스마트 현관등)

```text
출입 노드 PIR 감지
   ↓ MQTT (retrace/entrance/motion)
Jetson Nano
   ↓
차키 Last Seen 확인 → 아직 책상에 있음
   ├─ MQTT (retrace/entrance/light: ALERT) → 현관등 경고 점등
   ├─ USB Serial → STM32 레이저로 차키 안내
   └─ ntfy → 폰 알림 "차키 챙기셨나요?"
```

### BLE 부저

```text
Web / PWA
   ↓ HTTP
Jetson Nano
   ↓ BLE
부저 태그 (ESP32-C3)
   ↓
부저 울림
```

---

## 프로젝트 구조

```text
Retrace-Project/
│
├─ arduino/                          # 서랍 노드 (Arduino Uno)
│  ├─ arduino.ino                    # setup() / loop(), 전체 흐름
│  └─ src/
│     ├─ drawer/
│     │  └─ DrawerController.h/.cpp  # 서랍 LED ×6, 팝업 서보 ×6 제어
│     ├─ button/
│     │  └─ EmergencyButton.h/.cpp   # 비상 버튼 입력
│     └─ communication/
│        └─ Communication.h/.cpp     # HC-06 Bluetooth ↔ Jetson
│
├─ esp32/                            # ESP32 노드 (Arduino IDE)
│  ├─ entrance_node/                 # 출입 노드 · 스마트 현관등 (LOLIN D32)
│  │  └─ entrance_node.ino           # PIR · LED · Wi-Fi · MQTT
│  └─ buzzer_tag/                    # 부저 태그 (ESP32-C3)
│     └─ buzzer_tag.ino              # BLE 수신 · 부저
│
├─ jetson_nano/                      # 허브 (C/C++)
│  ├─ main.cpp
│  ├─ CMakeLists.txt
│  ├─ vision/
│  │  └─ Detector.h/.cpp             # 객체 탐지 · 영상 처리
│  ├─ record/
│  │  └─ LastSeen.h/.cpp             # 마지막 목격 정보 생성 · 관리
│  ├─ storage/
│  │  ├─ Database.h/.cpp             # DB 저장
│  │  └─ ImageStorage.h/.cpp         # 스냅샷 저장
│  ├─ communication/
│  │  ├─ Stm32Link.h/.cpp            # USB Serial → AIM · 레이저 명령 / ← PIR 이벤트
│  │  ├─ ArduinoLink.h/.cpp          # Bluetooth → 서랍 명령 / ← 비상 버튼 이벤트
│  │  ├─ MqttLink.h/.cpp             # MQTT ↔ 출입 노드 (움직임 · 현관등)
│  │  ├─ BleBuzzer.h/.cpp            # BLE → 부저 태그 호출
│  │  └─ PhoneNotifier.h/.cpp        # ntfy → 폰 사이렌 · 외출 알림
│  └─ server/
│     └─ Server.h/.cpp               # Web/PWA 요청 처리
│
├─ stm32/                            # 레이저 헤드 (CubeMX + CMake)
│  ├─ Core/
│  │  ├─ Inc/                        # 헤더 (아래 Src와 짝)
│  │  └─ Src/
│  │     ├─ main.c                   # CubeMX 생성 (초기화 · 메인 루프)
│  │     ├─ pir_sensor.c             # PIR 입력 처리
│  │     ├─ pan_tilt.c               # Pan/Tilt 서보 PWM
│  │     ├─ laser.c                  # 레이저 ON/OFF
│  │     └─ ...                      # 그 외 CubeMX 생성 파일
│  ├─ Drivers/                       # HAL · CMSIS (CubeMX 생성)
│  ├─ cmake/
│  ├─ CMakeLists.txt
│  ├─ CMakePresets.json
│  ├─ Retrace_STM32.ioc              # CubeMX 설정
│  ├─ startup_stm32f411xe.s
│  └─ STM32F411xx_FLASH.ld
│
├─ web/                              # 사용자 Web / PWA
├─ docs/                             # 회로도 · 구성도 · 개발 문서
├─ .github/
│  └─ CODEOWNERS
├─ .gitattributes
├─ .gitignore
└─ README.md
```

> 프로젝트 구조는 개발 진행에 따라 변경될 수 있습니다.

---

## 개발 문서

- [STM32 핀맵](docs/stm32_pinmap.md)

---

## 진행 상태

**현재 개발 진행 중**
