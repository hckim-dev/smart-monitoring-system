# 설치·운용

[문서 안내](README.md) · [하드웨어](03_hardware_and_pinmap.md) · [프로토콜](02_protocol_specification.md)

## 준비 환경

| 구성 | 환경·도구 |
| --- | --- |
| STM32 | Arm GNU Toolchain 15.2.1, GNU Make 호환 도구, STM32CubeProgrammer |
| Qt | C++17 compiler, qmake, Widgets·Network·SerialPort·Charts |
| Android | Gradle 9.5.0, AGP 9.3.1, SDK 37, daemon JDK 25 |
| 장치·연결 | STM32F411RE 시연 장치, Android API 26 이상 카메라 기기, USB VCP·TCP 네트워크 |

Android의 Kotlin Compose plugin은 2.4.10, Compose BOM은 2026.06.01, CameraX는 1.6.1입니다. [버전 카탈로그](../android/gradle/libs.versions.toml)를 기준으로 하며 Java source/target compatibility 11과 daemon JDK는 별개입니다.

## 빌드

각 절차는 **저장소 루트에서 시작하는 별도 PowerShell 세션** 기준입니다.

### STM32

[Makefile](../stm32-firmware/Makefile)의 `TOOL_DIR`·`VERSION`을 실제 설치와 맞춥니다.

~~~powershell
arm-none-eabi-gcc --version
Set-Location stm32-firmware
mingw32-make all
~~~

출력은 `rom_0x08000000.elf/.bin/.map`과 disassembly 파일입니다. 설치 위치가 다르면 `mingw32-make all TOOL_DIR=C:/설치경로 VERSION=15.2.1`로 지정합니다.

SWD 연결과 대상 보드를 확인한 뒤 플래싱합니다.

~~~powershell
STM32_Programmer_CLI.exe -c port=SWD -w ./rom_0x08000000.elf -v -rst -q
~~~

### Qt

Qt Creator에서 `qt-server/qt-server.pro`를 열고 SerialPort·Charts가 포함된 Kit를 선택합니다. 현재 소스는 Qt 5 스타일 namespace macro를 사용합니다. 명령줄은 MinGW Kit 예시입니다.

~~~powershell
qmake -v
New-Item -ItemType Directory -Force qt-server/build | Out-Null
Set-Location qt-server/build
qmake ../qt-server.pro
mingw32-make -j4
~~~

실행 target은 `SmartMonitoringServer`이며 출력 위치는 Kit·configuration에 따라 다릅니다. qmake·compiler·모듈의 Qt major version과 설치 prefix를 맞춥니다.

### Android

Android Studio에서 `android/`를 열어 Gradle Sync를 수행하거나 Wrapper를 사용합니다. SDK 경로는 로컬 환경에 지정하며 `local.properties`는 커밋하지 않습니다.

~~~powershell
Set-Location android
.\gradlew.bat --version
.\gradlew.bat :app:assembleDebug
~~~

빌드가 성공하면 연결된 디바이스를 확인한 뒤 APK를 설치합니다. 아래 경로는 계속 `android/`에 있는 상태를 기준으로 합니다.

~~~powershell
adb devices
adb install -r app/build/outputs/apk/debug/app-debug.apk
~~~

첫 실행 시 카메라 권한을 허용합니다.

## 연결과 구동

1. [핀맵](03_hardware_and_pinmap.md)에 따라 하드웨어·전원을 확인합니다.
2. Qt에서 실제 VCP 포트를 선택해 **115200bps**로 연결하고 ADC·차트 갱신을 확인합니다.
3. 임계값·자동 차단·CSV 저장 여부를 정하고 **TCP 서버를 시작**합니다.
4. Android에 PC의 실제 IP와 port를 입력해 접속합니다. 기본 port는 **8080**입니다.
5. 영상 전송을 켜고 Qt의 영상과 모바일의 센서값을 확인합니다.

같은 LAN·PC 핫스팟 주소를 사용하거나 Tailscale을 별도로 구성해 VPN 주소로 접속합니다. 앱 기본 IP `192.168.137.1`은 실제 네트워크에 맞게 바꿉니다. 방화벽의 해당 TCP 포트 허용 여부도 확인합니다.

Android 수신 timeout은 10초이므로 시리얼 샘플을 먼저 확인하면 센서 데이터 부재와 연결 문제를 구분하기 쉽습니다.

## 설정과 저장 위치

| 항목 | 기본값·위치 |
| --- | --- |
| 임계값 / 자동 차단 | `3000` / ON |
| 시리얼 속도 / TCP port | `115200` / `8080` |
| CSV 저장 | ON, `gas_log_yyyy-MM-dd.csv` |
| CSV 경로 | `./logs`, 프로세스 작업 디렉터리 기준 |
| 설정 파일 | `config.ini`, 실행 파일 디렉터리에 종료 시 저장 |

연결은 사용자가 시작합니다. 설정 복원과 로드 보정 규칙은 [Qt 설정](05_qt_control_center.md#설정과-수명)을 참고합니다.

## 시나리오별 검증

아래는 실행 시 확인할 항목이며 완료된 시험 결과를 뜻하지 않습니다.

| 확인 항목 | 조작 | 코드상 기대 동작 |
| --- | --- | --- |
| 정상 표시 | 임계값 미만 입력 | 양쪽 센서값·차트·정상 배지 |
| 자동 차단 | 자동 차단 ON, 래치 해제 상태에서 위험 입력 | 차단 요청, 서보 90° 목표·팬·LED ON |
| 정상화·복구 | 입력을 낮추고 원격 복구 | 배지는 먼저 정상, 복구 명령 후 0°·팬·LED OFF |
| 위험 중 복구 | 위험 입력을 유지한 채 원격 복구 | 다음 샘플에서 재차단 가능 |
| 현장 조작 | PB4·PB5 입력 | MCU 출력 변경, Qt 래치는 별도 유지 |
| 재부팅·통신 해제 | MCU reset 또는 연결 해제 | 마지막 값·출력·래치와 연결 표시를 따로 확인 |
| 세션·종료 | 새 클라이언트 접속, 서버·앱 종료 | 연결 교체·자원 해제·이벤트 순서 확인 |
| 영상 부하 | 영상 ON 상태에서 입력·명령 조작 | 지연·메모리·UI 응답 관찰 |

입력 변화 → 명령 송신 → PWM 변화 → 실제 구동을 나눠 기록하면 제어 지연의 원인을 구분할 수 있습니다. 측정에는 보드·폰·빌드 버전, 네트워크·부하 조건과 로그·파형을 함께 남깁니다.

## 검증 현황

펌웨어는 Arm GNU 15.2.1과 기존 Makefile로 컴파일·링크·ELF/BIN 생성을 확인했습니다. Qt 재빌드는 로컬 Charts 모듈 부재로 중단됐으며 Android APK 재빌드는 미검증입니다. 장치·UI 사진과 [시연 영상](assets/demo.mp4)은 제공하지만 정량 지연·전원 파형·가스 농도 정확도 측정 자료는 없습니다.

## 증상별 확인

| 증상 | 확인 위치 |
| --- | --- |
| 펌웨어 도구 탐색 실패 | Makefile TOOL_DIR·VERSION |
| Qt compiler·module 오류 | Kit PATH, qmake와 같은 Qt 버전의 SerialPort·Charts |
| Gradle 구성 실패 | daemon JDK·SDK·카탈로그·오류 원문 |
| UART 연결 실패 | 실제 VCP·baud·다른 앱의 점유·USB |
| TCP 연결 실패 | listen·IP·port·방화벽·VPN |
| 영상 중 연결 해제 | UART 샘플 부재와 수신 timeout |
| 영상 색상·배치 이상 | YUV plane stride와 ImageUtils |
| 정상 배지인데 서보 유지 | 수동 복구 방식과 [상태 모델](01_system_architecture.md#상태-소유권) |
| 현장 복구 후 재차단 생략 | Qt 래치와 MCU 상태 불일치 |

자격 증명·서명 키 관리는 [Credential safety](credential-security.md)를 참고합니다.
