# 시스템 아키텍처

[문서 안내](README.md) · [프로토콜](02_protocol_specification.md) · [실행 방법](08_setup_and_operation_guide.md)

## 역할 분담

| 구성 | 책임 | 주요 입출력 |
| --- | --- | --- |
| STM32 | 센서 측정, 현장 버튼, 액추에이터 구동 | ADC · GPIO · PWM · UART |
| Qt | 위험 판정, 자동·수동 제어 중계, 영상·차트·파일 처리 | UART · TCP · CSV · INI |
| Android | 현장 영상 송신, 센서값 표시, 원격 명령 입력 | CameraX · TCP · Compose |

**자동 차단 정책은 Qt에 있습니다.** Android도 수신값과 임계값을 비교해 위험을 표시하지만 자동 명령을 생성하지 않습니다.

~~~mermaid
flowchart TB
    subgraph Mobile["Android"]
        Camera["CameraX · ImageUtils"] --> VM["MonitoringViewModel"]
        Screen["MonitoringScreen"] <--> VM["MonitoringViewModel"]
        VM <--> Socket["TcpSocketClient"]
    end
    subgraph Desktop["Qt"]
        TCP["TcpStreamServer"] <--> MW["MainWindow"]
        Serial["SerialManager"] <--> MW
        MW --> Chart["ChartManager"]
        MW <--> Settings["SettingsManager"]
        MW --> CSV["CSV 파일"]
    end
    subgraph Device["STM32"]
        ADC["ADC1 · PA6"] --> Loop["메인 루프"]
        Keys["현장 버튼"] --> Loop
        RX["USART2 RX ISR"] --> Loop
        Loop --> Act["서보 · 팬 · LED"]
    end
    Socket <-->|TCP| TCP
    Loop -->|"ADC / UART"| Serial
    Serial -->|"명령 / UART"| RX
~~~

## 데이터 흐름

| 경로 | 순서 |
| --- | --- |
| 계측 | ADC → MCU UART → SerialManager → 차트·CSV·Android |
| 자동 차단 | Qt 임계값 비교 → UART 명령 → MCU 출력 상태 변경 |
| 원격 수동 제어 | Android 버튼 → TCP → Qt 제어 슬롯 → UART → MCU |
| 현장 제어 | 물리 버튼 → MCU의 같은 출력 상태 함수 |
| 영상 | CameraX → JPEG → TCP → QPixmap → Qt 화면 |

MCU는 200ms 간격으로 ADC값을 전송합니다. Qt는 유효한 값을 받을 때 화면·파일·모바일을 갱신하고 자동 차단 조건을 검사합니다. 이 간격은 계측 작업의 설정값이며 실제 구동 완료까지의 지연과는 다릅니다.

MCU의 처리 순서는 UART 명령 → 키 스캔 → ADC 전송입니다. 같은 루프 회차에 원격 명령과 키 이벤트를 처리하면 뒤의 키 이벤트가 최종 상태를 정할 수 있습니다. 키 두 개의 눌림 edge가 동시에 검출되면 차단을 먼저 선택합니다.

## 상태 소유권

| 상태 | 소유 위치 | 의미 |
| --- | --- | --- |
| `g_valve_state` | STM32 | 마지막으로 적용한 액추에이터 논리 상태 |
| `m_isValveClosed` | Qt | 차단 명령 송신 결과를 바탕으로 기억하는 래치 |
| NORMAL / DANGER | Qt·Android | 마지막 ADC값과 임계값의 비교 |
| 연결·영상 전송 상태 | 통신 객체·Android ViewModel | 세션 상태와 사용자 선택 |

물리 위치 피드백과 명령 ACK가 없으므로 논리 상태·송신 결과·실제 위치를 구분합니다.

| 조건·입력 | 현재 동작 |
| --- | --- |
| 위험 + 자동 차단 ON + 래치 false | 차단 송신을 시도하고 성공 반환 시 래치 true |
| 위험 지속 + 래치 true | 자동 차단 재송신 생략 |
| ADC가 임계값 미만으로 하락 | 정상 배지로 변경, 출력 목표와 래치는 유지 |
| Qt·Android에서 복구 | 복구 송신 성공 반환 시 래치 false |
| 원격 복구 후 위험 지속 | 다음 샘플에서 다시 자동 차단 가능 |
| 물리 버튼으로 복구 / MCU reset | MCU 상태만 변경, Qt 래치와 자동 동기화하지 않음 |

Qt가 자동 차단한 뒤 현장에서 복구하면 Qt의 래치가 남아 재차단 명령을 생략할 수 있습니다. 상태 회신은 후속 확장의 주요 항목입니다.

## 동시성과 연결 수명

| 구성 | 실행 방식 | 부하가 걸리는 지점 |
| --- | --- | --- |
| STM32 | ISR + 단일 메인 루프 | ADC·UART 송신 polling |
| Qt | GUI 이벤트 루프 | JPEG 디코딩·스케일링·차트·파일 쓰기 |
| Android | UI + 카메라 executor + IO 코루틴 | 프레임 변환·직렬화된 소켓 송신 |

UART와 TCP는 독립적으로 연결합니다. TCP는 클라이언트 하나를 유지하고 새 연결 시 기존 연결을 교체합니다.

통신 해제 후 MCU는 마지막 출력 목표를 유지하며 버튼 처리를 계속합니다. Qt는 마지막 값·배지·차단 래치를 보존합니다. Android도 마지막 센서값을 유지하며 수신 timeout은 10초입니다. Qt의 UART 수신이 멈추면 영상 전송 중에도 읽기 timeout 경로로 들어갈 수 있습니다.

현재 자동 재연결·heartbeat·통신 두절 시 독립 차단은 없습니다. Tailscale은 필요할 때 앱 밖에서 구성하는 VPN이며, 애플리케이션 자체의 인증·암호화 구현과 구분합니다.

## 코드로 확인하기

[MCU 제어](../stm32-firmware/main.c) · [Qt 이벤트 연결](../qt-server/mainwindow.cpp) · [TCP 서버](../qt-server/tcpstreamserver.cpp) · [Android 상태](../android/app/src/main/java/com/hocheol/smartmonitoringsystem/ui/MonitoringViewModel.kt)
