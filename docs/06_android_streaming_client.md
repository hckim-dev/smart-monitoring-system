# Android 클라이언트

[문서 안내](README.md) · [프로토콜](02_protocol_specification.md) · [빌드](08_setup_and_operation_guide.md#android)

## 구성과 상태

`MainActivity`에서 `MonitoringScreen`을 표시합니다. 화면은 카메라, 연결, 센서·제어, 로그의 네 영역으로 나뉩니다.

| 구성 | 책임 |
| --- | --- |
| MonitoringScreen | Compose UI, 권한 요청, CameraX 바인딩 |
| MonitoringViewModel | StateFlow 상태, 센서값 해석, 사용자 동작 조율 |
| TcpSocketClient | IO 코루틴의 TCP 연결·송수신과 이벤트 |
| ImageUtils | ImageProxy의 JPEG 변환·회전 보정 |

~~~mermaid
flowchart LR
    UI["MonitoringScreen"] -->|"사용자 동작"| VM["MonitoringViewModel"]
    VM -->|"StateFlow"| UI
    VM -->|"연결 · 명령 · JPEG"| Client["TcpSocketClient"]
    Client -->|"SharedFlow 이벤트"| VM
    Camera["CameraX"] --> Image["ImageUtils"]
    Image -->|"JPEG"| VM
~~~

IP·port, 연결·전송 상태, 카메라·플래시, 센서값·임계값을 `StateFlow`로 관리합니다. 화면은 `collectAsStateWithLifecycle()`로 구독합니다. 기본 주소는 `192.168.137.1:8080`이며 실제 서버 주소로 바꿔 사용합니다.

`GAS:값:임계값` 수신 후 UI에서 위험 여부를 계산합니다. 제어 버튼은 `"1"`·`"0"` 송신을 요청하고 로그를 추가합니다. 로그는 구동 완료 ACK가 아니며, 최근 100개를 메모리에 유지합니다.

<table>
  <tr><th>카메라·연결</th><th>센서·제어</th></tr>
  <tr>
    <td><img src="../images/android_app_camera.png" alt="카메라와 서버 연결 화면" width="220"></td>
    <td><img src="../images/android_app_control.png" alt="센서값과 차단·복구 제어 화면" width="220"></td>
  </tr>
</table>

## 카메라에서 화면까지

| 단계 | 처리 | 실행 맥락 |
| --- | --- | --- |
| 카메라 연결 | Preview·ImageAnalysis를 lifecycle에 바인딩 | main executor |
| 프레임 분석 | 연결·전송 ON일 때 JPEG 변환 | 전용 단일 executor |
| 송신·수신 | 길이 헤더·JPEG 송신, 센서 줄 수신 | Dispatchers.IO |
| UI 갱신 | 소켓 이벤트 → ViewModel → 화면 | UI / ViewModel scope |

Manifest에 INTERNET·CAMERA 권한을 선언하고 카메라는 런타임 권한을 요청합니다. 4:3 비율을 우선 선택하고 프리뷰는 FIT_CENTER로 표시합니다. 고정 해상도·FPS는 지정하지 않습니다.

카메라·플래시 변경 시 use case를 다시 바인딩합니다. 분석 후 `ImageProxy.close()`를 호출하고, 화면 dispose 시 executor를 shutdown합니다. 프레임 변환의 예외 경로에서 close가 보장되도록 하는 처리는 후속 점검 대상입니다.

### 이미지 변환

현재 경로는 `YUV_420_888 → Y·V·U 버퍼 복사 → NV21 지정 → JPEG 품질 50`입니다. 회전이 필요하면 Bitmap 디코딩·Matrix 회전 후 다시 JPEG로 압축하고 중간 Bitmap을 recycle합니다.

`rowStride`·`pixelStride`·cropRect를 반영하지 않습니다. YUV_420_888은 여러 plane 배치를 허용하므로 다른 기기에서 색상·행 정렬을 검증해야 합니다. [공식 포맷 정의](https://developer.android.com/reference/android/graphics/ImageFormat#YUV_420_888)

### 최신 프레임 정책

CameraX의 `KEEP_ONLY_LATEST`는 **분석 단계**의 대기를 제한합니다. [CameraX 분석 정책](https://developer.android.com/media/camera/camerax/analyze)

현재 송신은 프레임마다 별도 코루틴으로 요청하므로 느린 네트워크에서는 Mutex 대기 중인 JPEG가 쌓일 수 있습니다. 큐 상한·오래된 프레임 폐기와 명령 우선순위는 다음 개선 과제입니다.

## 소켓과 수명

`connect()`는 기존 연결을 정리하고 Socket을 생성합니다. 연결 timeout은 5초, 수신 timeout은 10초입니다. 연결 후 수신 코루틴이 줄 단위 메시지를 읽습니다.

`sendRaw()`는 4바이트 길이와 JPEG를, `sendText()`는 명령 한 줄을 씁니다. 두 함수가 같은 Mutex를 사용해 영상 헤더·본문과 명령의 바이트 혼합을 막습니다. 이 Mutex는 연결·해제와 멤버 교체 전체를 보호하는 lock은 아닙니다.

연결 정리는 receive job cancel → Socket close → stream 참조 해제 순서입니다. ViewModel의 `onCleared()`도 disconnect를 요청합니다. scope 취소와 종료 이벤트의 겹침, 빠른 재연결은 자원 수명 검증이 필요합니다.

Qt의 센서 전송이 멈추면 영상 송신 중에도 읽기 timeout이 발생할 수 있습니다. 연결 해제 이벤트는 연결·전송 상태를 내리며 마지막 센서값은 유지합니다.

## 코드와 빌드

- [화면](../android/app/src/main/java/com/hocheol/smartmonitoringsystem/ui/MonitoringScreen.kt) · [ViewModel](../android/app/src/main/java/com/hocheol/smartmonitoringsystem/ui/MonitoringViewModel.kt)
- [소켓](../android/app/src/main/java/com/hocheol/smartmonitoringsystem/network/TcpSocketClient.kt) · [이미지 변환](../android/app/src/main/java/com/hocheol/smartmonitoringsystem/util/ImageUtils.kt)
- [app 설정](../android/app/build.gradle.kts) · [버전 카탈로그](../android/gradle/libs.versions.toml)

minSdk는 26, compileSdk·targetSdk는 37, CameraX는 1.6.1입니다. 전체 도구 버전과 APK 생성 절차는 [실행 가이드](08_setup_and_operation_guide.md#android)를 참고합니다.
