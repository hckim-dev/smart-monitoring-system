# 통신 프로토콜

[문서 안내](README.md) · [시스템 상태](01_system_architecture.md#상태-소유권)

이 문서는 현재 송신기와 수신기가 사용하는 형식을 정의합니다. `\r`은 CR(`0x0D`), `\n`은 LF(`0x0A`)를 뜻합니다.

## 경로 요약

| 방향 | 매체 | 메시지 |
| --- | --- | --- |
| STM32 → Qt | UART | ADC 정수 + CRLF |
| Qt → STM32 | UART | 차단 `0x31` / 복구 `0x30` |
| Android → Qt | TCP | 4바이트 길이 + JPEG 또는 명령 + LF |
| Qt → Android | TCP | `GAS:<ADC>:<THRESHOLD>\n` |

MCU와 Android 사이의 제어·계측 중계는 Qt에서 이루어집니다. 영상은 Android에서 Qt로만 전송합니다.

## UART

### 연결 설정

| 항목 | 설정 |
| --- | --- |
| MCU 주변장치 | USART2 |
| 핀 | PA2 TX / PA3 RX, AF7 |
| 통신 | 115200bps, 8 data bits, no parity, 1 stop bit, no flow control |
| PC 연결 | 시연 장치의 ST-LINK Virtual COM Port |
| MCU 기준 클럭 | PCLK1 48MHz |

Qt에서 다른 baud rate를 선택할 수 있지만 현재 MCU는 `115200`으로 초기화합니다. 양쪽 설정을 맞춰야 합니다.

### ADC 전송

MCU는 200ms 간격으로 `printf("%d\n", adc_val)`을 호출합니다. `syscalls.c`의 송신 함수가 LF 앞에 CR을 추가하므로 선로상의 예시는 다음과 같습니다.

~~~text
표현: 1265\r\n
Hex:  31 32 36 35 0D 0A
~~~

Qt는 `canReadLine()`이 참일 때만 줄을 읽고 `trimmed()`를 적용합니다. 숫자 변환 성공과 `0 <= ADC <= 4095`를 모두 만족한 줄만 `dataReceived(int)`로 전달합니다. 따라서 부팅 메시지와 빈 줄은 센서값으로 처리되지 않습니다.

이 검증은 형식·범위 검증입니다. 범위 안의 잘못된 값이나 측정 노이즈를 검출하는 CRC·통계 필터는 없습니다.

### 제어

| ASCII | Hex | MCU 출력 목표 |
| --- | --- | --- |
| `'1'` | `31` | 서보 90°, 팬 ON, LED ON |
| `'0'` | `30` | 서보 0°, 팬 OFF, LED OFF |

Qt의 `sendChar()`는 개행 없이 1바이트를 씁니다. MCU ISR은 받은 바이트를 공유 슬롯에 저장하고, 메인 루프는 `'1'`·`'0'`만 해석합니다.

ACK, 상태 응답, 순번, 재전송은 없습니다. 수신 슬롯은 FIFO가 아니므로 연속 명령을 모두 보존하지 않습니다. ISR의 갱신과 메인 루프의 flag 해제 사이에 새 명령이 도착하면 처리 기회를 잃을 수도 있습니다.

## TCP

### 세션

| 항목 | 설정 |
| --- | --- |
| 서버 | Qt `QTcpServer` |
| 바인딩 | `QHostAddress::AnyIPv4` |
| 기본 포트 | `8080`, Qt UI에서 변경 가능 |
| 클라이언트 | Android `Socket` |
| 연결 timeout | 5000ms |
| Android 수신 timeout | 10000ms |
| 동시 연결 | 한 개 유지, 새 연결 시 기존 연결 교체 |

Android 기본 IP는 `192.168.137.1`입니다. 고정 탐색이나 Tailscale API 연동은 없으며 사용자가 실제 서버 주소를 입력합니다.

### Android → Qt: JPEG

~~~text
offset  size       내용
0       4 bytes    JPEG 바이트 수 N, Big-Endian
4       N bytes    JPEG 본문
~~~

길이 `1000`의 헤더는 `00 00 03 E8`입니다. Android는 `ByteArray.size`를 4바이트로 나눠 쓰고, Qt는 `qint32`로 읽어 `1 <= N <= 10 * 1024 * 1024` 범위를 허용합니다. 허용 범위의 길이는 모두 최상위 바이트가 `0x00`입니다.

JPEG 품질 인자는 기본 `50`이며 CameraX는 4:3 비율을 우선 선택합니다. 고정 해상도나 전송 FPS는 지정하지 않습니다.

### Android → Qt: 제어 명령

~~~text
차단: "1\n"  → 31 0A
복구: "0\n"  → 30 0A
~~~

Android `sendText()`는 `PrintWriter.println()`을 사용합니다. Qt는 줄을 trim한 뒤 정확히 `"1"` 또는 `"0"`일 때만 명령 signal을 발생시킵니다.

Qt에는 수신 버퍼가 정확히 1바이트인 `'1'`·`'0'`도 즉시 처리하는 보조 경로가 있습니다. 개행 없는 명령을 여러 개 붙이거나 영상과 합쳐 보내는 일반 형식은 정의하지 않습니다. 현재 Android처럼 개행으로 끝내야 합니다.

### Qt → Android: 텔레메트리

~~~text
GAS:1265:3000\n
~~~

| 필드 | 의미 |
| --- | --- |
| `GAS` | 메시지 식별자 |
| `1265` | ADC 원시값 |
| `3000` | Qt의 현재 임계값 |

Qt는 유효한 UART 샘플을 받을 때 연결된 클라이언트에 전송합니다. 연결 직후나 임계값 변경 직후에 별도 상태 snapshot을 보내지 않습니다. 변경된 임계값도 다음 센서 샘플에서 전달됩니다.

Android는 `readLine()`으로 수신한 문자열을 `:`로 분리합니다. 필드가 3개 이상이면 앞의 두 수치 필드를 정수로 변환하고, 변환 실패 시 기존 값을 유지합니다. 별도의 범위 검증은 없습니다.

## 수신 버퍼와 메시지 경계

TCP는 메시지 경계를 보존하지 않습니다. Qt의 `m_buffer`와 `m_imageSize`는 다음 상태로 동작합니다.

~~~mermaid
flowchart TD
    Read["readAll 결과를 버퍼에 추가"] --> Start{"JPEG 본문 대기 중인가?"}
    Start -->|아니오| Kind{"첫 바이트가 0인가?"}
    Kind -->|아니오| Text["LF까지 명령 처리 · 미완성 줄 대기<br/>단일 바이트 명령은 보조 경로로 처리"]
    Kind -->|예| Header["4바이트 헤더 대기 · 길이 검사"]
    Header --> Body["N바이트 본문 대기"]
    Start -->|예| Body
    Body --> Decode["JPEG 디코딩 · 표시 signal"]
    Decode --> Next{"남은 데이터가 있는가?"}
    Text --> Next
    Next -->|예| Start
~~~

- 헤더가 4바이트 미만이면 다음 수신을 기다립니다.
- 본문이 N바이트 미만이면 크기 상태를 유지합니다.
- 한 번에 여러 프레임이 도착하면 반복문으로 계속 처리합니다.
- 본문 수신 중에는 JPEG 내부의 LF나 `'1'`을 명령으로 해석하지 않습니다.
- 길이가 0 이하 또는 10MiB 초과이면 현재 버퍼와 길이 상태를 초기화합니다.
- JPEG 디코딩에 실패한 프레임은 표시하지 않고 다음 메시지로 진행합니다.

길이 제한이 전체 수신 버퍼의 상한을 뜻하지는 않습니다. LF 없는 텍스트 입력의 누적 제한, 본문 완성 timeout, 오류 후 명확한 재동기화 식별자는 없습니다.

## 송신 순서와 보장 범위

Android의 `sendRaw()`와 `sendText()`는 같은 `Mutex`를 사용합니다. 정상적인 송신에서 JPEG의 헤더·본문 사이로 텍스트가 끼어드는 것을 막습니다. 프레임 송신 완료까지 제어 명령이 기다릴 수 있으며, 제어 명령 우선 큐는 없습니다.

Qt `QSerialPort::write()`와 `QTcpSocket::write()`는 송신 데이터를 버퍼에 맡깁니다. 현재 `sendChar()`는 반환값이 `-1`이 아닌지 확인하며, 수신측 처리 완료는 확인하지 않습니다. Android의 명령 송신 로그 역시 구동 완료 ACK가 아닙니다.

## 소스 안내

| 역할 | 파일 |
| --- | --- |
| UART 출력과 수신 | [syscalls.c](../stm32-firmware/syscalls.c), [uart.c](../stm32-firmware/uart.c), [isr.c](../stm32-firmware/isr.c) |
| Qt 송수신 | [serialmanager.cpp](../qt-server/serialmanager.cpp), [tcpstreamserver.cpp](../qt-server/tcpstreamserver.cpp) |
| Android 송수신 | [TcpSocketClient.kt](../android/app/src/main/java/com/hocheol/smartmonitoringsystem/network/TcpSocketClient.kt), [MonitoringViewModel.kt](../android/app/src/main/java/com/hocheol/smartmonitoringsystem/ui/MonitoringViewModel.kt) |
