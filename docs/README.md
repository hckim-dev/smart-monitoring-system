# 기술 문서

[프로젝트 소개·시연](../README.md)에서 결과를 보고, 관심 있는 영역의 문서로 이어서 읽을 수 있습니다.

| 문서 | 내용 |
| --- | --- |
| [01 아키텍처](01_system_architecture.md) | 세 구성요소의 책임, 데이터 흐름, 상태 전이 |
| [02 프로토콜](02_protocol_specification.md) | UART·TCP 메시지 형식과 수신 경계 |
| [03 하드웨어](03_hardware_and_pinmap.md) | 실제 장치 구성, 핀맵, 전원 연결 |
| [04 펌웨어](04_firmware_driver_guide.md) | 부팅, 타이머, ADC, ISR·메인 루프 |
| [05 Qt](05_qt_control_center.md) | 관제 이벤트, 자동 차단, 차트·CSV·설정 |
| [06 Android](06_android_streaming_client.md) | 카메라·코루틴·UI 상태 관리 |
| [07 문제 해결](07_troubleshooting_log.md) | 타이밍·전원·프레이밍 문제와 설계 판단 |
| [08 실행 방법](08_setup_and_operation_guide.md) | 빌드 환경, 연결 순서, 동작 확인 |

**구조를 이해하려면 01 → 02**, **직접 구동하려면 03 → 08** 순서로 읽는 것을 권합니다.

구현 설명은 현재 소스 기준입니다. 개발 중 관찰과 타이밍 계산은 실측 결과와 구분하며, 구동 시 확인할 항목은 [동작 확인](08_setup_and_operation_guide.md#시나리오별-검증)에 모았습니다.

## 자료

[장치 사진](../images/hardware_setup.png) · [서보 상태 비교](../images/valve_states.png) · [시연 영상](../images/demo.mp4) · [발표 PDF](smart_monitoring_system_presentation.pdf)

발표 PDF와 화면 캡처는 제작 당시 자료입니다. 현재 구현의 세부 동작과 버전은 기술 문서·소스를 참고합니다. 자격 증명과 서명 키 관리는 [Credential safety](credential-security.md)에 정리했습니다.
