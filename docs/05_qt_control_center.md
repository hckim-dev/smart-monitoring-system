# Qt 관제 서버

[문서 안내](README.md) · [상태 모델](01_system_architecture.md#상태-소유권) · [프로토콜](02_protocol_specification.md)

## 구성

`QApplication → MainWindow → 이벤트 루프`로 실행하는 C++17 Widgets 애플리케이션입니다.

| 모듈 | 책임 | 소스 |
| --- | --- | --- |
| MainWindow | 이벤트 연결, 위험 판정, UI·CSV·설정 조율 | [구현](../qt-server/mainwindow.cpp) |
| SerialManager | COM 연결, 줄 수신·검증, 제어 문자 송신 | [구현](../qt-server/serialmanager.cpp) |
| TcpStreamServer | 세션, JPEG·명령 수신, 센서값 송신 | [구현](../qt-server/tcpstreamserver.cpp) |
| ChartManager | ADC 곡선, 임계값 선, 샘플 축 이동 | [구현](../qt-server/chartmanager.cpp) |
| SettingsManager / Logger | INI 저장·복원 / 이벤트 로그 형식 | [설정](../qt-server/settingsmanager.cpp) · [로그](../qt-server/logger.h) |

[mainwindow.ui](../qt-server/mainwindow.ui)는 화면을 정의합니다. 버튼 슬롯은 UI 자동 연결을, 통신 signal은 생성자의 명시적 연결을 사용합니다.

| 정상 입력 | 위험 입력 |
| --- | --- |
| ![정상 입력의 관제 화면](../images/qt_normal.png) | ![위험 입력의 관제 화면](../images/qt_danger.png) |

## 이벤트 흐름

~~~mermaid
flowchart LR
    Serial["SerialManager"] --> Gas["onGasDataReceived"]
    Gas --> View["수치 · 차트 · 배지"]
    Gas --> Data["TCP 전송 · CSV"]
    Gas --> Auto["자동 차단 조건 검사"]
    Auto --> Send["sendChar"]
    TCP["TCP 원격 명령"] --> Manual["수동 제어 슬롯"]
    Manual --> Send
    Frame["JPEG 수신"] --> Preview["영상 표시"]
~~~

`onGasDataReceived()`는 수치·차트 갱신 → TCP 전송 → 위험 판정·CSV·배지 → 자동 차단 검사 순서로 실행됩니다. 자동 차단 조건은 `자동 차단 ON && ADC >= 임계값 && !m_isValveClosed`입니다.

차단·복구 송신이 성공을 반환하면 래치를 갱신합니다. Android 명령도 같은 수동 제어 슬롯을 사용합니다. 배지는 센서 위험 여부이며, 입력값 하락만으로 복구 명령을 보내지는 않습니다.

## 통신과 실행 맥락

시리얼은 115200 8N1을 기본으로 하고, 완성된 줄의 정수·범위를 검사합니다. 포트 목록은 시작 시와 미연결 상태의 콤보박스 클릭 시 갱신합니다. `QSignalBlocker`로 목록 재구성 중 signal을 억제하고 이전 선택을 복원합니다.

실제 포트가 없으면 COM1~COM10을 표시합니다. `ResourceError`가 발생하면 연결을 닫으며, 마지막 센서값·차트·래치는 보존합니다.

TCP는 클라이언트 한 개를 유지합니다. JPEG를 재조립한 뒤 `QPixmap`으로 디코딩하고 label 크기에 맞춰 비율을 유지하며 표시합니다. 메시지 경계는 [프로토콜 문서](02_protocol_specification.md#수신-버퍼와-메시지-경계)를 참고합니다.

통신·디코딩·차트·파일 쓰기는 같은 GUI 스레드에서 실행합니다. 별도 worker thread는 없으며, CSV 접근은 자동 차단 송신보다 앞에 있으므로 영상·저장 부하를 함께 측정해야 합니다.

## 차트와 기록

| 항목 | 동작 |
| --- | --- |
| Y축 | ADC `0~4095` |
| X축 | 시각 대신 샘플 index |
| 표시 | 파란 ADC 곡선, 빨간 임계값 선 |
| 데이터 유지 | 현재 조건식 기준 최대 51개 점 |
| CSV | `gas_log_yyyy-MM-dd.csv`에 샘플별 append |
| 이벤트 로그 | 시각·카테고리·레벨을 붙여 QTextEdit에 표시 |

`m_maxDataPoints`는 50이지만 `m_dataIndex > m_maxDataPoints` 조건으로 제거하므로 51개 점이 남습니다. CSV timestamp는 PC에서 저장한 초 단위 시각입니다.

~~~csv
Timestamp,ADC_Value,Threshold,Status
2026-10-26 14:30:00,1265,3000,NORMAL
~~~

형식 설명용 예시입니다. 이벤트 로그는 센서 CSV와 별개이며 파일로 저장하지 않습니다.

## 설정과 수명

| 설정 | 기본값 |
| --- | --- |
| 임계값 / 자동 차단 | `3000` / ON |
| 시리얼 속도 / TCP port | `115200` / `8080` |
| CSV 저장 / 경로 | ON / `./logs` |

설정은 실행 파일 디렉터리의 `config.ini`에 종료 시 저장하고 다음 실행 때 복원합니다. CSV의 기본 상대 경로는 프로세스 작업 디렉터리 기준입니다. 연결은 사용자가 별도로 시작합니다.

로드 시 잘못된 임계값·port는 기본값으로 보정합니다. UI는 임계값 0을 허용하지만 로드 시 0은 3000으로 보정합니다. 파일 저장 실패를 별도 보고하는 처리는 없습니다.

통신 manager는 MainWindow의 QObject 자식, SettingsManager는 `unique_ptr`로 관리합니다. 서버 중지·앱 종료와 재연결 시 소켓·이벤트 순서는 [동작 확인 항목](08_setup_and_operation_guide.md#시나리오별-검증)에 포함합니다.

## 빌드

[qmake 프로젝트](../qt-server/qt-server.pro)는 Widgets·Network·SerialPort·Charts를 사용합니다. Qt 5 스타일 namespace macro가 있으므로 Qt major version과 Kit를 확인합니다.

→ [Qt 빌드 절차](08_setup_and_operation_guide.md#qt)
