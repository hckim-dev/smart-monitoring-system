# Smart Monitoring System

**STM32 · Qt · Android를 연결한 가스 위험 모니터링·원격 제어 시스템**

센서 입력과 현장 영상을 PC·모바일에서 확인하고, 위험 임계값에 따라 밸브·환기 팬·경보 LED를 제어합니다. 가변저항으로 가스 입력을, 서보 회전으로 밸브 동작을 모사한 프로토타입입니다.

**C99 / CMSIS · C++17 / Qt Widgets · Kotlin / Compose / CameraX · UART / TCP**

[시연](#시연) · [설계와 구현](#설계와-구현) · [기술 문서](docs/README.md) · [실행 방법](docs/08_setup_and_operation_guide.md) · [발표 자료](docs/smart_monitoring_system_presentation.pdf)

## 주요 기능

- **위험 감지·자동 차단** — 자동 차단을 켜면 ADC값이 임계값 이상일 때 Qt가 명령을 보내 서보·팬·LED를 함께 제어합니다.
- **세 경로의 수동 제어** — PC, Android 앱, 현장 물리 버튼에서 차단·복구할 수 있습니다.
- **영상·센서 통합 관제** — 스마트폰 카메라 영상, 센서 차트, 일별 CSV 기록을 제공합니다.

<p>
  <img src="images/qt_danger.png" alt="PC 관제: 현장 영상, 센서 차트, 자동 차단 로그" width="400">
  <img src="images/android_app_control.png" alt="모바일 제어: 센서값과 밸브 차단·복구 버튼" width="210">
</p>

## 전체 구조

~~~mermaid
flowchart LR
    MCU["STM32<br/>센서 측정 · 설비 구동"] <-->|"UART"| Qt["Qt 관제 서버<br/>위험 판정 · 표시 · 저장"]
    Qt <-->|"TCP"| App["Android<br/>현장 영상 송신 · 원격 제어"]
~~~

STM32가 200ms 간격으로 센서값을 보내고 **Qt가 자동 차단을 판단**합니다. Android는 카메라 영상을 Qt로 보내면서 센서값을 수신하고 제어 명령을 전달합니다.

## 설계와 구현

| 기술 과제 | 구현 방식 | 상세 |
| --- | --- | --- |
| 계측 중 원격 명령·현장 버튼 처리 | ISR에서 수신 정보를 저장하고 메인 루프에서 주기 작업과 구동 상태를 통합. 서보는 타이머 PWM으로 제어 | [펌웨어](docs/04_firmware_driver_guide.md) |
| UART와 TCP를 통한 관제·제어 연결 | Qt signal-slot으로 계측·차트·명령을 연결하고 차단 래치로 자동 명령의 반복 송신을 줄임 | [Qt](docs/05_qt_control_center.md) |
| 하나의 연결에서 JPEG와 명령 구분 | 길이 헤더·누적 수신 버퍼로 영상 경계를 복원하고 송신 Mutex로 명령과 영상의 바이트 혼합 방지 | [프로토콜](docs/02_protocol_specification.md) |
| 카메라 처리와 UI 응답 분리 | 전용 executor에서 프레임 변환, IO 코루틴에서 소켓 처리, StateFlow로 UI 상태 전달 | [Android](docs/06_android_streaming_client.md) |

## 시연

https://github.com/user-attachments/assets/efa6ab54-6c9d-45a7-a74a-6c195651bac4

[영상 파일 직접 보기](images/demo.mp4)

1. 가변저항으로 입력을 높여 **자동 차단과 팬·LED 연동**을 확인합니다.
2. 입력을 낮춘 뒤 PC 또는 Android에서 **수동 복구**합니다.
3. 현장 버튼으로 출력 상태를 바꾸고 **영상·센서 표시**를 확인합니다.

## 동작 범위

시연값은 `0~4095`의 ADC 원시값입니다. 자동 차단 후 복구는 수동으로 수행합니다. 현재 TCP 연결은 클라이언트 한 개를 유지하며, 물리 버튼으로 바뀐 MCU 상태를 Qt에 회신하는 기능은 없습니다.

상태 전이와 연결 해제 시 동작은 [아키텍처](docs/01_system_architecture.md#상태-소유권), 개발 중 문제와 판단은 [문제 해결 기록](docs/07_troubleshooting_log.md)에 정리했습니다.

## 더 살펴보기

| 목적 | 문서 |
| --- | --- |
| 구조·상태·통신 이해 | [아키텍처](docs/01_system_architecture.md) · [프로토콜](docs/02_protocol_specification.md) |
| 하드웨어·코드 확인 | [핀맵](docs/03_hardware_and_pinmap.md) · [펌웨어](docs/04_firmware_driver_guide.md) · [Qt](docs/05_qt_control_center.md) · [Android](docs/06_android_streaming_client.md) |
| 설계 판단·실행 재현 | [문제 해결 기록](docs/07_troubleshooting_log.md) · [설치·운용](docs/08_setup_and_operation_guide.md) |
