# STM32 펌웨어

[문서 안내](README.md) · [하드웨어](03_hardware_and_pinmap.md) · [UART 규격](02_protocol_specification.md#uart)

## 구성과 진입점

C99와 CMSIS 장치 헤더로 주변장치 레지스터를 직접 설정합니다. RTOS, HAL 호출, DMA 기반 계측·송신은 사용하지 않습니다.

| 파일 | 책임 |
| --- | --- |
| [crt0.s](../stm32-firmware/crt0.s) | 벡터 테이블, 초기 MSP, .data 복사, .bss 초기화, `Main` 진입 |
| [rom_0x08000000.lds](../stm32-firmware/rom_0x08000000.lds) | Flash·SRAM 배치와 런타임 초기화 심볼 |
| [main.c](../stm32-firmware/main.c) | 초기화 순서, 주기 작업, 명령 처리, 출력 상태 통합 |
| [clock.c](../stm32-firmware/clock.c), [option.h](../stm32-firmware/option.h) | 클럭 설정과 주파수 상수 |
| [timer.c](../stm32-firmware/timer.c), [isr.c](../stm32-firmware/isr.c) | tick·PWM 생성과 수신 ISR |
| [adc.c](../stm32-firmware/adc.c) | ADC1_IN6 단일 변환 |
| [uart.c](../stm32-firmware/uart.c), [syscalls.c](../stm32-firmware/syscalls.c) | USART2, 표준 입출력 연결 |
| [key.c](../stm32-firmware/key.c), [led.c](../stm32-firmware/led.c), [motor.c](../stm32-firmware/motor.c), [servo.c](../stm32-firmware/servo.c) | 입력과 액추에이터 드라이버 |

`system_stm32f4xx.c`는 빌드되지만 시작 경로에서 `SystemInit()`을 호출하지 않습니다. FPU와 클럭은 `Sys_Init()`에서 직접 설정합니다. `SystemCoreClock`은 초기값 16MHz에서 갱신되지 않으며 현재 드라이버의 주기 계산은 `option.h` 상수를 사용합니다.

## 부팅 순서

~~~mermaid
flowchart TD
    Reset["벡터 테이블 · __start"] --> Runtime["data 복사 · bss 초기화"]
    Runtime --> Sys["Main → Sys_Init<br/>FPU · 96MHz · USART2"]
    Sys --> Init["타이머 → ADC → LED → 팬 → 키 → 서보"]
    Init --> RX["UART RX 인터럽트 활성화"]
    RX --> Run["메인 루프"]
    Run --> Cmd["수신 명령 처리"]
    Cmd --> Key["30ms 경과 시 키 스캔"]
    Key --> Sample["200ms 경과 시 ADC 읽기 · printf"]
    Sample --> Run
~~~

초기 출력은 각 드라이버 초기화에서 정합니다. LED와 팬은 OFF, 서보는 0° 목표입니다. 이후 `Valve_Set_State(0)` 호출은 초기 논리 상태도 0이므로 가드에서 반환합니다.

링커는 Flash `0x08000000`부터 512KiB, SRAM `0x20000000`부터 128KiB를 정의합니다. 벡터의 초기 MSP는 `0x20020000`입니다. `syscalls.c`는 newlib 입출력과 4KiB 힙 상한을 사용하는 `_sbrk()`를 제공합니다. 메인 제어 경로에 명시적인 동적 할당은 없습니다.

## 클럭과 타이밍

코드의 클럭 계산은 다음과 같습니다.

~~~text
HSI 16MHz / PLLM 8 × PLLN 192 / PLLP 4 = SYSCLK 96MHz
AHB /1  → HCLK 96MHz
APB1 /2 → PCLK1 48MHz, 해당 설정에서 TIM2·TIM4 입력 96MHz
APB2 /1 → PCLK2 96MHz
~~~

`FLASH->ACR`은 3 wait states와 prefetch·Flash instruction/data cache enable 비트를 설정합니다. 96MHz는 이 프로젝트의 설정값이며 MCU의 최대 사양이라는 뜻은 아닙니다. 전압 조건과 실제 클럭·타이밍은 보드에서 확인해야 합니다.

| 항목 | 코드 설정에서 계산한 값 |
| --- | --- |
| TIM4 prescaler | `1919` → 96MHz / 1920 = 50kHz, 20µs tick |
| TIM4 auto-reload | `49` → 50 count, 1ms update |
| TIM2 prescaler | `95` → 1µs tick |
| TIM2 auto-reload | `19999` → 20ms 주기, 50Hz |
| ADC 공통 prescaler | PCLK2 / 6 → 16MHz |
| USART2 | PCLK1 48MHz 기준 115200bps, BRR 계산 결과 `0x01A1` |

타이머 입력 계산은 현재 APB 분주 설정을 전제로 합니다. 타이머 클럭 선택의 일반 규칙은 [RM0383의 RCC·TIM 설명](https://www.st.com/resource/en/reference_manual/rm0383-stm32f411xce-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)을 참고합니다.

## ISR과 메인 루프

| 맥락 | 접근 상태 | 처리 |
| --- | --- | --- |
| TIM4 ISR | `g_sys_tick` 쓰기 | update flag 정리 후 tick 증가 |
| USART2 ISR | `g_rx_cmd`·`g_rx_flag` 쓰기 | DR의 하위 8비트 저장, flag 설정 |
| 메인 루프 | tick·수신 상태 읽기, flag 해제 | 명령 해석·키 스캔·ADC·출력 |

주기 판정은 `g_sys_tick - last_tick >= interval` 형태의 unsigned 차이를 사용합니다. 작업 시각에 `last_tick`을 현재 tick으로 갱신하므로 밀린 주기를 모두 따라잡는 스케줄러는 아닙니다.

`volatile`은 공유 상태의 접근을 유지하기 위한 선언이며 명령 소비 절차 전체를 원자적으로 만들지 않습니다. 수신 바이트 읽기·명령 처리·flag 해제가 한 임계 구역으로 묶이지 않아 새 수신과 겹칠 수 있습니다. 큐·오버런 집계·UART 오류 복구는 별도로 구현하지 않았습니다.

`ADC1_Read()`의 EOC 대기와 `printf()`의 UART TXE 대기는 polling입니다. 명시적 timeout은 없습니다. 따라서 delay를 사용하지 않는 주기 판정과 전체 실행 경로의 non-blocking 여부를 구분합니다.

## ADC와 출력 드라이버

ADC1은 PA6를 analog로 설정하고 regular sequence 길이 1, 첫 채널 6, sampling time 480 cycles를 사용합니다. 소프트웨어로 변환을 시작하고 EOC를 기다린 뒤 12비트 값을 반환합니다. `(480 + 12) / 16MHz = 30.75µs`는 설정 기반 변환 시간 계산이며 함수 전체의 실측 소요 시간이 아닙니다.

서보는 다음 정수식으로 `TIM2->CCR1`을 갱신합니다.

~~~text
pulse_us = 500 + angle × 2000 / 180
0° → 500µs / 90° → 1500µs / 180° → 2500µs
~~~

`Servo_Set_Angle()`은 입력을 최대 180으로 제한합니다. PWM은 계속 출력하며 이동 완료 후 자동으로 끄는 함수는 없습니다. 저장된 `g_servo_current_angle`은 목표값으로, 실제 위치를 읽은 값이 아닙니다.

LED는 PB0·PB1, 팬은 PC0·PC1의 ODR을 갱신합니다. 두 출력 레지스터와 PWM 갱신은 순차 호출이며 동일한 하드웨어 edge에서 동시에 전환한다는 보장은 없습니다.

## 공통 상태 함수와 버튼

`Valve_Set_State()`는 상태가 다를 때만 논리 상태를 바꾸고 `LED_Display() → Motor_Display() → Servo_Display()`를 호출합니다. 같은 요청의 중복 갱신을 줄이지만 PWM 발생 자체를 중지하거나 서보 떨림·전류 감소를 보장하지는 않습니다.

키 입력은 내부 pull-up을 사용하고 30ms 간격으로 HIGH→LOW 변화를 검사합니다. 별도의 연속 안정 샘플 확인이나 시간 기반 debounce 상태 머신은 없습니다. 짧은 누름이 샘플 사이에 끝나면 검출하지 못할 수 있습니다.

통합 상태의 동작과 원격·현장 요청의 순서는 [아키텍처의 상태 소유권](01_system_architecture.md#상태-소유권)을 참고합니다.

## 빌드와 검증 범위

[Makefile](../stm32-firmware/Makefile)은 Cortex-M4, FPv4-SP-D16 hard-float ABI, `-std=gnu99`, `-O3`, `-Wall`을 사용합니다. 출력은 `rom_0x08000000.elf/.bin/.map`과 disassembly 파일입니다.

로컬 검토에서는 Arm GNU Toolchain 15.2.1로 임시 복사본의 기존 Makefile 빌드를 통과했습니다. 이 결과는 실제 클럭·PWM 파형·인터럽트 지연·서보 각도 검증을 포함하지 않습니다. 실행 명령과 점검 항목은 [설치·운용·검증](08_setup_and_operation_guide.md)에 정리했습니다.
